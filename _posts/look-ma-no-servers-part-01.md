---
title: "Look Ma, No Servers! The Hidden Limit of AWS Glue: Subnet IP Exhaustion"

series: Look Ma! no servers
---

## The 7AM Ritual

Our daily batch consisted of 2 critical Glue jobs per weekday and 2 non-critical Glue jobs every 2 hours. Every morning at 7:00 AM, our batch ETL jobs kicked off. By 7:03 AM, Slack lit up.

```
Job: customer-analytics-daily
Status: FAILED
Duration: 0 minutes
Error: The specified subnet does not have enough free
       addresses to satisfy the request
```

1–2 intermittent failures everyother weekday morning. Data warehouse SLAs slipping. Downstream teams blocked before they'd finished their first coffee. And the thing that made it maddening: our calculation confirmed that we had capacity. Plenty of it.

Something was consuming IPs that we couldn't see.

## What We Didn't Understand About Glue's Networking

The first thing we had to accept was that "serverless" is a marketing word. The network is still there. The servers are still there. AWS just hides them behind an abstraction — which means the constraints are real, but the visibility is something you have to build yourself.

Our Glue jobs pull custom Python libraries and dependencies from a self-hosted JFrog Artifactory inside of a private VPC. To reach it, every Glue job binds to a Glue Connection — a VPC and a subnet. And every Glue worker that spins up grabs an Elastic Network Interface: a virtual NIC holding a real IP from the subnet's CIDR, for the lifetime of the job.

Four concurrent jobs × ten workers each = 40 IPs. From a /26 with 59 usable. We'd sized for ~5 minutes of ENI cleanup lag between runs — a reasonable assumption, since AWS doesn't document otherwise. On paper, fine. In practice, failing.

> AWS reserves 5 IP addresses per subnet, so a /26 provides 59 usable addresses.

So we started looking at the ENIs themselves.

### Step 1: What's actually in the subnet right now?

We ran this during a live failure at 7:03AM:

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=subnet-id,Values=subnet-xxxxx" \
  --query 'NetworkInterfaces[*].Status' \
  --output text | sort | uniq -c
```

The output stopped us cold:

```
  8 available
 50 in-use
```

58 ENIs. In a subnet with 59 usable IPs. The math was already over before our first job tried to launch.

But *50 in-use*? We were only running four jobs with ten workers each. That's 40. Where were the other twelve coming from?

### Step 2: Who owns the ENIs that shouldn't be there?

We pulled the full inventory with descriptions and requester IDs:

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=subnet-id,Values=subnet-xxxxx" \
  --query "NetworkInterfaces[*].[NetworkInterfaceId, Status, Description, RequesterId]" \
  --output table
```

Every ENI came back owned by an AWS-managed Glue service account. The 12 "extra" in-use ENIs had descriptions matching jobs that had finished hours ago. One of them was from the non-critical batch that had run at 5AM and completed cleanly around 6:00.

It was 7:03. Those ENIs had been attached, doing nothing, for over an hour and three minutes.

### Step 3: Why won't they go away?

We picked one of the ghost ENIs and asked it directly:

```bash
aws ec2 describe-network-interfaces \
  --network-interface-ids eni-xxxx \
  --query "NetworkInterfaces[0].Attachment"
```

```json
{
  "AttachTime": "2026-04-09T01:51:32+00:00",
  "AttachmentId": "eni-attach-xxxx",
  "DeleteOnTermination": false,
  "DeviceIndex": 1,
  "NetworkCardIndex": 0,
  "InstanceOwnerId": "xxxx",
  "Status": "attached"
}
```

The attachment block revealed two things. First, "DeleteOnTermination": false — meaning the ENI lifecycle is managed asynchronously by the Glue service rather than tied directly to instance termination. Second, and more surprising: the ENI had been in in-use state for over an hour after the job completed. in-use should mean an active workload. It didn't appear to. Glue was leaving ENIs attached to instances we couldn't see, for reasons we couldn't observe, long after the work was done. The flag told us cleanup would be async; the attach time told us how async.

That explained the 50. What about the 8 in `available`?

### Step 4: What about the "available" ones?

We pulled one of the `available` ENIs:

```bash
aws ec2 describe-network-interfaces \
  --network-interface-ids eni-yyyy \
  --query "NetworkInterfaces[0].Status"

"available"
```

No attachment block. No active workload. No visible reason to exist. And yet: still holding an IP. We watched one for 35 minutes before it finally disappeared. Ran it again a day later — 38 minutes. A third time — 32.

A pattern was forming. Glue's ENI lifecycle wasn't a two-state machine. It was four states, and three of them consumed subnet IPs:

```
CREATED → in-use (attached) → available (detached) → deleted
                   ^                    ^
          holds IP up to 60+ min   holds IP 30–40 min
```

Together, for a single ENI, they could pin an IP for sixty minutes after its job completed. Multiply that by a batch cadence that doesn't wait sixty minutes between runs, and you get 7:03AM every morning.

This isn't just us — it's a known pattern. See [AWS Glue does not clean up network interfaces](https://repost.aws/questions/QUrmpoQEFxRwWvt5Es6nGmaA/aws-glue-does-not-clean-up-network-interfaces) and [AWS Glue Job Not Releasing ENIs After Completion](https://repost.aws/questions/QUBdpMaxjcST-voDzcNld3bA/aws-glue-job-not-releasing-enis-after-completion), where one operator reports 1,200+ accumulated ENIs from the same root cause. Also [Glue Connections Running Out of IPs](https://repost.aws/questions/QUE2ZiLoMFSt-tl7OV5pySbg/glue-connections-running-out-of-ips).

### Step 5: Who else is fighting for this subnet?

Before we could plan a fix, we needed to know the full blast radius. Every Glue connection bound to the subnet, every job bound to those connections:

```bash
aws glue get-connections \
  --query "ConnectionList[?contains(PhysicalConnectionRequirements.SubnetId, 'subnet-xxxxx')].Name" \
  --output table

aws glue get-jobs \
  --query "Jobs[?contains(Connections.Connections, '<connection-name>')].Name" \
  --output table
```

The answer was everything. Critical batch, non-critical batch, ad-hoc backfills, manual reruns — every Glue job in our account shared one /26. One misbehaving backfill could starve the entire morning SLA. It already had been.

We had the picture. Now we needed the fix.

## What We Tried First (And Why It Wasn't Enough)

Our first instinct was the obvious one: make the subnet bigger. We pulled up the VPC console, drafted the CIDR change, and were halfway through the Terraform PR before we stopped and asked: *does this actually solve it?*

A /24 gives you 251 usable IPs. Four times the headroom. But our ENI consumption wasn't static — we'd been bumping worker counts every quarter as data volumes grew, and job durations were creeping up with them. And here's the worse part — without the ghost ENI problem *visible*, we'd still hit silent 7:03AM failures, just less often. Less frequent failures that still blow the SLA are harder to catch, not easier. We closed the PR.

The second instinct was architectural: eliminate the VPC binding entirely with PrivateLink. No VPC, no ENIs, no problem. We spent an afternoon mapping what it would take — and hit a wall. PrivateLink requires an AWS service endpoint to front. Since our Artifactory ran outside AWS, this would require additional infrastructure we weren't prepared to build. Dead end.

Sitting in the postmortem the next morning, one of us drew the ENI lifecycle on the whiteboard next to the job launch timeline and said the thing that reframed it: *the subnet isn't too small. The launcher is blind.*

We were firing Glue jobs into a subnet without ever asking whether there was room. The fix wasn't more IPs. It was feedback.

## Phase 1: Splitting by Blast Radius

We still had to deal with the shared-subnet problem before admission control could do anything useful. If one category of job poisoned the pool, admission control would just politely refuse to launch anything — SLA misses with extra steps.

So we split the /26 into three subnets. Not by capacity. By *what could hurt what*.

- **Subnet A (/26, us-east-1a)** — critical batch (customer-analytics, product-metrics)
- **Subnet B (/26, us-east-1b)** — non-critical batch (reporting, aggregations)
- **Subnet C (/27, us-east-1c)** — backfill, ad hoc, manual reruns

> Subnet C is intentionally small — ad-hoc failures are acceptable; the sensor here is a courtesy, not an SLA gate

The logic we kept coming back to: if a backfill leaves 20 ENIs lingering in `available` for 40 minutes, we want that cleanup tail *trapped*. Contained in Subnet C. Not bleeding into the critical morning window in Subnet A.

We put each subnet in a different AZ, but we were careful not to oversell that to ourselves. A Glue job binds to one Connection, one subnet, one AZ. There's no native multi-AZ for a single Glue job. All we bought was that *different job categories* live in different AZs. A us-east-1a event still takes out critical batch until we manually repoint the Connection. 

The cutover was straightforward. Three new subnets, three new Glue Connections, DAGs repointed via Terraform. We staged it over a week — critical first (highest risk, do it while we're paying attention), non-critical next, backfills last. Every migration window we waited for the subnet to drain to zero active ENIs before flipping traffic.

The 7AM failures dropped immediately. But we knew the story wasn't over. Isolation stopped categories from starving each other. It didn't stop *one* category from starving its own subnet during a bad cleanup day. We'd moved the failure mode, not killed it.

## Phase 2: Teaching Airflow to Look Before It Leapt

The whiteboard insight — *the launcher is blind* — was still unaddressed. We'd given the launcher more rooms to work in, but it still walked into each one with its eyes closed.

So we wrote a sensor. Simple in shape: before each Glue job category runs, a custom Airflow sensor calls `DescribeNetworkInterfaces` filtered by subnet ID, counts ENIs across all states (`in-use` and `available`), and makes a call — launch now, or wait sixty seconds and ask again. Max wait: ten minutes, then fail loud.


### Why a Static Buffer Didn't Work

Our first version reserved a flat 20 IPs as the "cleanup lag" buffer. It was wrong in both directions. On fast-cleanup days we were blocking launches that would have succeeded fine. On slow-cleanup days — the ones we actually cared about — 20 wasn't enough, and jobs still failed.

We started logging cleanup lag per run and staring at the distribution. It wasn't even close to stable. Some days it peaked at 15, some days at 45 (due to adhoc manual re-runs). A static number was always going to be wrong. We needed the sensor to learn.

### Adaptive Threshold: Rolling Max via DynamoDB

The data path ended up clean because we resisted the urge to build anything fancy.

**The table.** `glue-eni-observations` in DynamoDB, partitioned by `subnet_id`, sorted by `observed_at`. Each record captures `total_enis`, `in_use`, `available`, and `dag_run_id`. TTL 30 days for debugging.

**The writer.** The sensor itself. On every poke, before making the admit/wait decision, it writes what it observed. No separate Lambda, no post-job hook — the observation always reflects exactly what the sensor saw at decision time.

**The reader.** On each poke, the sensor reads back the last N observations for its subnet (N=5, tunable) and takes the **max** of `total_enis`. That becomes the buffer estimate:

```
# At decision time:
current_total = count all ENIs in subnet (in-use + available)
free_subnet_ips = subnet_size - current_total

# Buffer = worst cleanup tail observed across last N pre-launch observations
cleanup_buffer = max(rolling_max_available_enis, MIN_BUFFER)

# Admit if there's room for new workers + expected cleanup tail overhang
admit if free_subnet_ips >= workers_needed + cleanup_buffer
```

`MIN_BUFFER = 10` is our floor — a guard against initial runs after implementing the changes.

**Why max, not average.** We argued about this one. Average hides the worst case. If four of the last five runs saw 20 ENIs and one saw 45, average says 25, reality says 45. We're sizing for the peak we've actually observed, not the one we'd prefer.

**Why DynamoDB, not XCom.** XCom is per-DAG-run. We needed observations shared across every DAG hitting the same subnet — critical batch, non-critical batch, and ad-hoc all contribute signal about Subnet A's state. DynamoDB gave us one shared view keyed by subnet, which is what we actually needed.

### Things That Bit Us (Or Nearly Did)

- **Worker slot exhaustion.** Our first version used `mode='poke'`, which holds the Airflow worker slot while sleeping. With multiple blocked sensors queued up, we nearly exhausted the Airflow worker pool before a single Glue job ran. Switched to `mode='reschedule'` — releases the slot while waiting. Non-obvious, critical.
- **EC2 API throttling.** Four sensors polling every 60s is ~4 `DescribeNetworkInterfaces` calls/minute — well under the limit. But Describe\* calls share a rate-limit bucket across the account. If other teams adopt this pattern, or someone runs heavy EC2 automation in parallel, we'll hit throttling sooner than expected. Something to watch, not panic about yet.
- **Silent sensor waits.** Early on we had a sensor hang indefinitely on a misconfigured subnet. Nothing alerted. The job just sat there looking "in progress." We added the 10-minute timeout and explicit task failure after that. A sensor that waits forever is operationally invisible — worst failure mode in the system.

### Limitations We've Accepted

- **Subnet ceiling.** Each subnet has its own cap. Subnet A's 59 IPs will run out if we add critical jobs or bump worker counts. Six-month capacity review catches growth before we hit the wall.
- **Lookback tuning.** Rolling max over 5 runs needs occasional adjustment. Too short and we miss cleanup patterns; too long and the buffer grows stale. Reviewed quarterly.
- **No Glue Auto Scaling.** Glue 4 supports it, but the sensor can't predict how many workers Glue will actually provision mid-run. Fixed worker counts give us deterministic IP math. Worth revisiting if workloads become more variable.

### Observability

A Grafana dashboard plots ENI utilization against available capacity. The correlation between cleanup spikes and job launches is immediately visible — which, after months of staring at this problem without that view, still feels like cheating.

An alert fires at 80% utilization — above the admission threshold but below exhaustion. It fires when something bypasses the sensor: a manual run, an unexpected workload, a new service quietly grabbing ENIs in the same subnet. That alert has caught us twice — both times from developers running one-offs in the wrong subnet.


### Before vs After Architecture

<img width="2558" height="1568" alt="architecture" src="https://github.com/user-attachments/assets/f8c026c9-8da6-4baf-83a6-9ed338f93756" />


### On-Call Runbook

When the 80% alert fires:

1. Run the Step 1 ENI count query from the investigation above. Confirm actual state.
2. Check for orphaned ENIs (`available` with no active job) — typically safe to delete once confirmed detached from any active Glue job via `aws ec2 delete-network-interface` after confirming. See [common deletion failure modes](https://repost.aws/questions/QUMRV13LI8QJKxcZVL3-1mUg/how-delete-networking-interface-asociate-with-vpc) and the [`InvalidParameterValue` error](https://repost.aws/questions/QU4vgx-AXzQ_CZcrYjZLEDKg/unable-to-delete-stuck-network-interface-eni-after-elb-deletion-need-aws-intervention) you'll hit if you try to delete before the ENI fully detaches.
3. Check for stuck job runs (`STOPPING` state that never resolves):
   ```bash
   aws glue get-job-runs --job-name <job-name> --max-results 10 \
     --query "JobRuns[*].[Id, JobRunState, StartedOn, CompletedOn]" --output table
   aws glue batch-stop-job-run --job-name <job-name> --job-run-ids <run-id>
   ```
   ENIs should detach within 5 minutes of a force-stop.
4. If neither explains it: pause non-critical DAGs, admit only SLA-critical jobs manually, notify downstream stakeholders. Resume normal scheduling once utilization drops below 60%.

Hitting 80% regularly is a signal to expand subnet capacity before it becomes an incident.

## Results

| Metric | Before | After (90 days) |
|---|---|---|
| Job runs/week | 20 | 20 |
| IP exhaustion failures/week | ~5 | 0 |
| Weekly failure rate | ~25% | 0% |
| 7AM on-call pages (this cause) | ~3/week | 0 |
| Sensor-blocked launches (recovered in <10 min) | n/a | 12 / 360 runs (3.3%) |

>All 12 blocked launches were cleanup-tail events — the sensor held the job, waited out the lag, and launched successfully within the 10-minute window. No manual intervention required.

## The Lesson: Build the Observability AWS Didn't
The IP exhaustion bug wasn't about IPs. It was about an undocumented cleanup window in a managed service we couldn't see into — invisible to CloudWatch, unmentioned in the Glue console. The fix wasn't clever networking; it was building the signal AWS didn't ship: ENI state, persisted, queryable, wired into admission.
"Serverless" moves the servers out of sight, not out of your problem. Every underlying constraint — IPs, disk, memory, quotas — still binds you. AWS decides which to expose. When they don't, silence isn't absence of risk; it's absence of signal.
We've hit the same shape twice since: No space left on device during Spark checkpoints, and exit code 137 with no executor logs. Both required the same playbook — instrument what AWS didn't, feed it back into admission or sizing, alert before the wall. We'll write those up next.

If you take one thing from this post: assume every managed service has an observability gap, and treat finding it as part of operating the service.

## Grafana dashboard

## References 

- [AWS Knowledge Center: Resolve "subnet does not have enough free addresses" in Glue](https://repost.aws/knowledge-center/glue-specified-subnet-free-addresses)
- [AWS Glue does not clean up network interfaces](https://repost.aws/questions/QUrmpoQEFxRwWvt5Es6nGmaA/aws-glue-does-not-clean-up-network-interfaces)
- [AWS Glue Job Not Releasing ENIs After Completion](https://repost.aws/questions/QUBdpMaxjcST-voDzcNld3bA/aws-glue-job-not-releasing-enis-after-completion) — 1,200+ accumulated ENIs from the same root cause
- [Glue Connections Running Out of IPs](https://repost.aws/questions/QUE2ZiLoMFSt-tl7OV5pySbg/glue-connections-running-out-of-ips)
- [How to delete a network interface associated with a VPC](https://repost.aws/questions/QUMRV13LI8QJKxcZVL3-1mUg/how-delete-networking-interface-asociate-with-vpc)
- [Unable to Delete Stuck ENI in in-use state](https://repost.aws/questions/QU4vgx-AXzQ_CZcrYjZLEDKg/unable-to-delete-stuck-network-interface-eni-after-elb-deletion-need-aws-intervention)
