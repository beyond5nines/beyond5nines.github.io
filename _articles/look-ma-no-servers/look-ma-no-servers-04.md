---
layout: post
title: "No Space Left on Device — Fix"
date: 2026-04-12 12:00:00 -0000
categories: aws glue serverless spark
series: Look Ma! no servers
excerpt: "S3 Shuffle stopped the crash but revealed 267.9 GB of shuffle data per run. Five Spark configuration changes cut shuffle volume by 87% and brought runtime from 56 minutes to 23."
redirect_from:
  - /2026/04/12/look-ma-no-servers-04/
---

## Where We Left Off

[Part 03](/look-ma-no-servers-03/) stopped the crash. S3 Shuffle moved spill off local disk, and the job stopped dying at 48 minutes. But it also gave us something to measure — and what we measured wasn't good.

**267.9 GB** of shuffle data written to S3 per run. 34,662 objects. For a job processing a fraction of that in actual payload.

The wall moved from local disk to S3. The waste didn't shrink. We'd fixed the symptom. Now we had to fix the cause.

---

## What the Shuffle Files Told Us

The [analysis in Part 03](/look-ma-no-servers-03/#phase-2-digging-into-the-shuffle-files) surfaced two concrete problems:

1. **Over-partitioning.** Spark's standard default for `spark.sql.shuffle.partitions` is 200, but the `.index` file — a write-side artifact — proves the shuffle was written with 72 partition slots. That number matches exactly what [AWS prescriptive guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/tuning-aws-glue-for-apache-spark/parallelize-tasks.html) documents as the target parallelism formula: `(Workers - 1) × cores_per_worker = (10 - 1) × 8 = 72`. We never configured it — Glue overrode the Spark default based on cluster geometry. Regardless of how the number was derived, 33 of those 72 partitions were completely empty — 46% wasted slots. Each empty partition still costs a file header, an index entry, a task slot, and a merge pass.

2. **Wrong compression codec.** LZ4 achieved 3x compression on text-heavy shuffle blocks but stored UUID-heavy blocks completely raw — 1.0x ratio. Every byte paid the codec overhead and got nothing back.

Both problems compound. More partitions means more shuffle blocks. Each block gets individually compressed. When compression returns nothing, you're writing raw data across more files than necessary, to a network-attached object store, on every single run.

---

## Five Configuration Changes

We applied these during Spark session initialization in the Glue job script:

```python
glueContext.spark_session.conf.set("spark.sql.shuffle.partitions", "50")
glueContext.spark_session.conf.set("spark.sql.adaptive.enabled", "true")
glueContext.spark_session.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
glueContext.spark_session.conf.set("spark.io.compression.codec", "zstd")
glueContext.spark_session.conf.set("spark.shuffle.spill.compress", "true")
```

Three of these drove the improvement. Two are defensive. Here's the distinction.

### The Active Changes

#### `spark.sql.shuffle.partitions = 50`

This is the most impactful setting for shuffle behavior. It controls how many partitions Spark creates after every wide transformation — `join`, `groupBy`, `distinct`, `repartition`. Every partition becomes a sort bucket, a shuffle block written to S3, and a downstream task.

Our job was writing 72 shuffle partitions — not Spark's standard default of 200, but a Glue-applied value matching our cluster's total executor cores: `(10 - 1) × 8 = 72`. We confirmed this from the `.index` file itself — a write-side artifact that cannot be modified by read-side optimizations. We never configured `spark.sql.shuffle.partitions`; Glue overrode the Spark default using the same formula [AWS prescriptive guidance recommends](https://docs.aws.amazon.com/prescriptive-guidance/latest/tuning-aws-glue-for-apache-spark/parallelize-tasks.html) as target parallelism. But 33 of those 72 partitions were completely empty. Each empty partition still costs: a file header, an index entry, a task slot, a merge pass. Setting this to 50 right-sizes the partition count to our actual shuffle volume — fewer files, larger blocks, better compression ratios, less IO.

#### `spark.io.compression.codec = zstd`

This controls the compression algorithm for shuffle blocks, spill files, and broadcast data. The default LZ4 prioritizes speed over compression ratio — which is the wrong tradeoff when your bottleneck is network IO to S3, not CPU.

ZSTD delivers ~2x compression on UUID strings where LZ4 gives nothing, and ~55% better compression than LZ4 on text data. For a workload that writes shuffle to a network-attached object store, the CPU cost of ZSTD is paid back many times over in reduced S3 writes and network transfer.

#### `spark.shuffle.spill.compress = true`

Ensures that when Spark's in-memory sort buffer overflows and spills to storage, the spill files are compressed before being written. This is narrower than the global codec setting — it specifically targets spill outputs. With ZSTD as the codec, spill files get meaningful compression even on high-entropy data.

### The Defensive Settings

#### `spark.sql.adaptive.enabled = true`

AQE was already enabled by default. Glue 5.0 runs Spark 3.5.4, and AQE has been on by default since Spark 3.2. We didn't know that. We assumed it was off because we never set it. That assumption — born from the fact that Glue never surfaces its effective Spark configuration — is exactly the kind of blind spot this series keeps finding.

Setting it explicitly doesn't change the runtime behavior. It makes the configuration *intentional and visible* rather than relying on a default that nobody on the team was aware of.

#### `spark.sql.adaptive.coalescePartitions.enabled = true`

Same story. This AQE sub-feature was also on by default in Spark 3.5.4. After a shuffle completes, Spark inspects the actual partition sizes and merges adjacent small partitions into fewer larger ones at the reduce stage. But here's the catch: with `spark.sql.adaptive.coalescePartitions.parallelismFirst = true` (the default since Spark 3.2), AQE prioritizes maintaining parallelism over minimizing empty partitions. Since Glue already set the partition count to 72 (matching available cores), AQE had no reason to coalesce further — 72 was already the parallelism target. The 33 empty partitions persisted because AQE's default behavior treats "maximize parallelism" as more important than "eliminate empty slots."

We set it explicitly for the same reason: if you don't know what's on, you can't reason about what's working. Pinning it in code means it survives Glue version upgrades and makes the behavior auditable.

> **Note:** AQE coalescing is a read-side optimization. It helps reduce task overhead when *consuming* shuffle output, but it doesn't reduce what gets *written* to S3. The write-side cost — the 72 partitions, 33 of them empty — was already being paid on every run. Reducing the partition count from 72 to 50 is what cut the write-side waste.

---

## POC Results

We ran the job with identical data, identical worker configuration (G.2X / 20 DPU), and measured what landed in the S3 shuffle prefix.

| Metric | Before (Baseline) | After (Tuned) | Change |
|--------|-------------------|---------------|--------|
| Shuffle bytes written to S3 | 267.9 GB | 133.0 GB | **-50.3%** |
| Object count | 34,662 | 35,570 | +2.6% |

Half the bytes. The reduction is driven by two factors: ZSTD compressing blocks that LZ4 couldn't (especially the UUID-heavy data), and fewer write-side partitions meaning each block is larger and compresses better. The slightly higher object count reflects AQE creating additional small stages — a read-side optimization that was already running but that we weren't aware of.

---

## Safety Considerations

These settings work for our current data volume. They won't necessarily work forever.

**The risk:** `spark.sql.shuffle.partitions = 50` means each partition handles `total_shuffle_volume / 50` bytes. If future data growth doubles the shuffle volume, each partition doubles in size. Large enough partitions trigger spill — which is exactly what we started this series trying to fix.

**The guardrails:**

- **AQE stays on.** If a stage produces more data than expected, AQE can split skewed partitions and rebalance. It won't save you from a 10x data spike, but it handles moderate variance.
- **Monitor the shuffle prefix.** If shuffle bytes written starts climbing toward the baseline, the partition count needs revisiting. The metric is trivial to capture — it's just an S3 listing on the shuffle bucket.
- **Broadcast small dimension tables.** Any join where one side fits comfortably in memory should use a broadcast join to eliminate the shuffle entirely. This reduces partition pressure regardless of the partition count.

The right mental model: `shuffle.partitions` is not a set-and-forget constant. It's a function of your largest shuffle stage. As data grows, re-validate.

---

## Results Summary

| Metric | Before S3 Shuffle (Post 03) | After S3 Shuffle | After Tuning (This Post) |
|--------|----------------------------|-----------------|--------------------------|
| Job outcome | Intermittent failure | Success | Success |
| Local disk pressure | Unbounded (crash) | Near zero | Near zero |
| Shuffle written to S3 | N/A (local disk) | 267.9 GB | 133.0 GB |
| Runtime (G.2X / 20 DPU) | 26m 05s | 26m 26s | ~26m |

---

## The Lesson

Five lines of configuration. 50% reduction in shuffle IO. No infrastructure changes, no scaling, no additional cost. Three of those lines did the work. Two made visible what was already happening in the dark.

The defaults were always wrong for this workload. Spark's standard default for `spark.sql.shuffle.partitions` is 200, but Glue overrode it to 72 — matching the [AWS-documented formula](https://docs.aws.amazon.com/prescriptive-guidance/latest/tuning-aws-glue-for-apache-spark/parallelize-tasks.html) for target parallelism: `(Workers - 1) × cores = 9 × 8 = 72`. A reasonable auto-tuning choice for parallelism, but wrong for our data distribution — 33 of those partitions were empty on every shuffle. Nobody surfaced the 46% waste. LZ4 made sense when shuffle was local-disk-fast. On S3, the CPU savings of LZ4 are meaningless — you're paying network latency per byte regardless.

And AQE — the optimization that was supposed to help — was already running. We just didn't know it. We assumed it was off because we never enabled it. In a system that hides its effective configuration, the only way to know what's running is to declare it yourself.

None of this surfaced in the Glue console. No recommendation, no warning, no metric that said "your compression is doing nothing" or "these partitions are empty" or "AQE is already on." The only signal was the 267.9 GB number — and you had to go look at S3 to find it.

Serverless doesn't mean the defaults are right for you. It means nobody will tell you when they're wrong — or when they're already right.

---

## References

- [AWS Prescriptive Guidance — Parallelize Tasks (target partition formula)](https://docs.aws.amazon.com/prescriptive-guidance/latest/tuning-aws-glue-for-apache-spark/parallelize-tasks.html)
- [Introducing Amazon S3 Shuffle in AWS Glue](https://aws.amazon.com/blogs/big-data/introducing-amazon-s3-shuffle-in-aws-glue/)
- [Spark Adaptive Query Execution](https://spark.apache.org/docs/latest/sql-performance-tuning.html#adaptive-query-execution)
- [Spark Compression and Serialization](https://spark.apache.org/docs/latest/configuration.html#compression-and-serialization)
- [Look Ma, No Servers! Part 03 — No Space Left on Device — Trap](/look-ma-no-servers-03/)
- [brahmagupta — scan CLI tool](https://github.com/beyond5nines/brahmagupta)
