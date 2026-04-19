---
layout: post
title: "Look Ma, No Servers! AWS Glue and the 'No Space Left on Device' Trap"
date: 2026-04-11 12:00:00 -0000
categories: aws glue serverless spark
series: Look Ma! no servers
redirect_from:
  - /2026/04/10/look-ma-no-servers-02/
  - /look-ma-no-servers-02/
---

## The 48-Minute Lie

Our product Glue job looked healthy for 48 minutes and 59 seconds. Progress metrics moving. No anomalies in the console. Then Slack lit up.

```
Job: product-glue-job
Status: FAILED
Duration: 48m 59s
Error: error while calling spill() on
       org.apache.spark.util.collection.unsafe.sort.UnsafeExternalSorter
       ... : No space left on device
```

This wasn't an underpowered job. It was running on **10 G.2X workers** — our standard allocation for heavy ETL. Yet it ran out of disk. And the thing that made it maddening: the Glue console gave us nothing. No disk utilization metric. No spill volume. No warning before the wall.

## What We Didn't Understand About Glue's Disk

The first thing we had to accept was that "serverless" is a marketing word. The disk is still there. AWS just hides it behind an abstraction — which means the constraints are real, but the visibility is something you have to build yourself.

Every Glue worker gets a fixed local ephemeral disk, shared across three consumers:

1. **Shuffle write buffers** — intermediate output from `join`, `groupBy`, `repartition`
2. **Spill files** — when the in-memory sort buffer overflows, Spark writes to local disk
3. **Executor scratch space** — deserialization, output staging, checkpoints

When spill accumulates faster than the sort completes, local disk fills. Spark throws `No space left on device` and the job dies — no graceful degradation, no retry. You find out at 48m 59s.

> **Important:** disk space is not pooled across workers — each worker has its own 128 GB independently. With 10 G.2X workers, one is the driver, leaving 9 executors. The spill error means at least one executor exhausted its local 128 GB. The constraint isn't total cluster disk. It's per-worker.

We had the picture. Now we needed the fix.

---

## Phase 1: S3 Shuffle — Move the Wall to S3

Our first move was to change *where* the shuffle writes. S3 Shuffle redirects shuffle output and spill files from local ephemeral disk to S3. Instead of filling up the worker's local disk, shuffle data now flows to S3 over the network — and S3 doesn't run out of space.

Three parameters to enable it:

```
--write-shuffle-files-to-s3  = TRUE
--write-shuffle-spills-to-s3 = TRUE
--conf spark.shuffle.glue.s3ShuffleBucket = s3://<shuffle-bucket>
```

Before touching production, we ran a controlled POC across four configurations. We intentionally used smaller workers (G.1X / 5 workers) to reproduce the failure reliably — our production job ran on 10 G.2X workers, but the same spill mechanics apply regardless of worker size when shuffle volume is high enough.

| Run | Workers / DPU | S3 Shuffle | Outcome | Duration | Note |
|-----|--------------|------------|---------|----------|------|
| A | G.2X / 10w / 20 DPU | No | **Success** | 26m 05s | Baseline — adequate resources |
| B | G.1X / 5w / 5 DPU | No | **FAIL** | 48m 59s | `No space left on device` |
| C | G.1X / 5w / 5 DPU | **Yes** | **Success** | 1h 27m | Same constrained config — S3 absorbs the spill |
| D | G.2X / 10w / 20 DPU | **Yes** | **Success** | 26m 26s | No overhead when disk isn't the bottleneck |

Run C stopped the argument. Identical constrained sizing to the failing run. S3 Shuffle on. Job completes. Run D answered the question we always get next: *does turning this on hurt the jobs that weren't failing?* 21 seconds slower than baseline. Within noise.

After enabling, the shuffle artifacts now land in S3 instead of local disk — and they're durable across executor restarts:

```
s3://<shuffle-bucket>/shuffle-data/<job-id>/
  ├── shuffle_53_21062_0.data
  ├── shuffle_53_21062_0.index
  ├── shuffle_98_37726_0.data
  └── shuffle_98_37726_0.index
```

The job was no longer dying. But we now had something we didn't have before: the shuffle files themselves, sitting in S3, readable. So we looked.

---

## Phase 2: Digging Into the Shuffle Files

### Step 1: What format are these files?

```bash
xxd shuffle_53_21062_0.data | head -2
```

```
00000000: 4c5a 3442 6c6f 636b 4c5a 3442 6c6f 636b  LZ4BlockLZ4Block
```

Magic bytes `LZ4Block` — Spark's `LZ4BlockOutputStream`. So compression was active. Whether it was *doing anything useful* was a different question.

We had two files to work with:
- `shuffle_53_21062_0.data` — **7.85 MB** + `.index` (72 partitions, 33 empty)
- `shuffle_98_37726_0.data` — **27.46 KB** + `.index` at **0.57 KB** (72 partitions, 33 empty)

The size difference between them was already a story.

### Step 2: What does the partition layout look like?

The `.index` file is a binary sequence of big-endian int64 offsets — one per partition boundary. We parsed it:

```python
import struct

with open("shuffle_98_37726_0.index", "rb") as f:
    data = f.read()

offsets = [struct.unpack(">q", data[i:i+8])[0] for i in range(0, len(data), 8)]
non_empty = sum(1 for i in range(len(offsets)-1) if offsets[i+1] > offsets[i])
empty     = sum(1 for i in range(len(offsets)-1) if offsets[i+1] == offsets[i])
print(f"Partitions: {len(offsets)-1}, non-empty: {non_empty}, empty: {empty}")
```

```
Partitions: 72, non-empty: 39, empty: 33
```

**Both files had the same partition layout: 33 of 72 partitions completely empty.** `spark.sql.shuffle.partitions` defaults to 200 — we'd never configured it. Spark was managing 72 sort buckets across both shuffles, regardless of how much data was in them. Despite the 285x size difference between the two files, the over-partitioning penalty was identical.

### Step 3: Is LZ4 actually compressing anything?

We decompressed the blocks and compared compressed size to decompressed size:

```python
import lz4.block

with open("shuffle_53_21062_0.data", "rb") as f:
    raw = f.read()

offset, blocks = 16, []
while offset < len(raw):
    size = struct.unpack(">i", raw[offset:offset+4])[0]
    offset += 4
    if size <= 0: break
    blocks.append(lz4.block.decompress(raw[offset:offset+size], uncompressed_size=65536))
    offset += size
```

The output stopped us:

```
shuffle_53 (HR text):    21,840 bytes → 65,536 bytes  ratio 3.0x  ✓
shuffle_98 (UUID graph): 27,420 bytes → 27,420 bytes  ratio 1.0x  ✗
```

`shuffle_98` was being stored **raw inside the LZ4 wrapper**. The codec couldn't find repeated patterns in UUID strings — every byte paid the codec overhead and got nothing back. LZ4 was doing zero work on nearly half the shuffle output.

### Step 4: How did the original failure actually unfold?

The shuffle files weren't the proximate cause. The accumulated spill files were. Here's what the disk timeline looked like on the original failing run:

```
Stage 1 — Shuffle write:
  200 partition slots written, 33/72 empty on both shuffles
  shuffle_53: 7.85 MB across 39 non-empty partitions
  shuffle_98: 27.46 KB across 39 non-empty partitions

Stage 2 — Sort and merge:
  UnsafeExternalSorter fills in-memory buffer
  → Spill #1 written to local disk
  → Spill #2, #3... none reclaimed until sort completes

Stage 3 — What happened before S3 Shuffle:
  Without S3 Shuffle, spill files accumulated on local disk
  → Local disk exhausted → No space left on device. Job killed.
  With S3 Shuffle enabled, spill is offloaded to S3 — these files
  are from a successful run.
```

Over-partitioning made it worse: 200 sort buckets forces more merge passes, more intermediate writes, more disk consumed per record.

### What the Analysis Is Telling Us

We now had two concrete problems the files were pointing at — neither of which was visible in the Glue console.

**Over-partitioning.** We had never configured `spark.sql.shuffle.partitions`, so it was falling back to the default of 200. Spark was managing sort buckets, writing partition headers, and performing merge passes for data that doesn't exist. Every empty partition is wasted disk and wasted CPU.

**Wrong compression codec for the data.** LZ4 worked fine on text-heavy data — 3x on `shuffle_53`. On UUID-heavy `shuffle_98`, it stored blocks completely raw: 0x compression, pure overhead. ZSTD would deliver ~2x compression on UUID strings where LZ4 gives nothing, and ~55% better compression than LZ4 on text data.

The partition tuning, Adaptive Query Execution, and the codec switch are the next set of changes. We'll cover the implementation and results in Part 03.

---

## Results

| Metric | Before | After S3 Shuffle |
|--------|--------|-----------------|
| Job success rate | Intermittent failure | 100% |
| Local disk peak pressure | Unbounded | Near zero |
| Runtime (well-provisioned) | 26m 05s | 26m 26s |

---

## The Lesson: Build the Visibility AWS Didn't

The job was running on 10 G.2X workers. It wasn't undersized. The failure was happening because "serverless" hid the disk constraint entirely.

S3 Shuffle fixed the failure. But it also handed us something AWS never did: the shuffle artifacts, in a place we could actually read them. The over-partitioning and the codec mismatch were always there. We just couldn't see them until the files were somewhere we could look.

The gap between "compression is enabled" and "compression is doing anything useful" doesn't show up in any Glue metric. It shows up when you open the files.

---

## References

- [Introducing Amazon S3 Shuffle in AWS Glue](https://aws.amazon.com/blogs/big-data/introducing-amazon-s3-shuffle-in-aws-glue/)
- [Spark Adaptive Query Execution](https://spark.apache.org/docs/latest/sql-performance-tuning.html#adaptive-query-execution)
- [Spark Compression and Serialization](https://spark.apache.org/docs/latest/configuration.html#compression-and-serialization)
- [Look Ma, No Servers! Part 01 — Subnet IP Exhaustion](/look-ma-no-servers-01/)
