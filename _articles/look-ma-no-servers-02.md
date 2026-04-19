---
layout: post
title: "Look Ma, No Servers! 'The specified subnet does not have enough free addresses to satisfy the request' — Fix"
date: 2026-04-10 12:00:00 -0000
categories: aws glue serverless
series: Look Ma! no servers
---

In [Part 1](/look-ma-no-servers-01/) we diagnosed the root cause: AWS Glue's ENI lifecycle extends 60+ minutes beyond job completion, every job category shared a single /26, and the launcher had no visibility into actual subnet state. The fix wasn't more IPs. It was feedback.

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

## Grafana Dashboard

Below is a sample Grafana dashboard we built to monitor ENI utilization across our three subnets. This dashboard surfaces the signals AWS doesn't expose natively — real-time ENI counts, cleanup lag patterns, rolling max buffers, and the 80% alert threshold that triggers before exhaustion hits.

<style>
  .gf-root { background: #111217; color: #d0d4de; font-family: var(--font-sans), sans-serif; font-size: 13px; padding: 12px; border-radius: 8px; }
  .gf-topbar { display: flex; align-items: center; justify-content: space-between; margin-bottom: 14px; }
  .gf-title { font-size: 15px; font-weight: 500; color: #e0e3ec; }
  .gf-meta { display: flex; gap: 10px; align-items: center; }
  .gf-badge { background: #1f2232; border: 0.5px solid #2f3347; padding: 3px 10px; border-radius: 4px; font-size: 11px; color: #8b92a8; }
  .gf-alert-badge { background: #2a1a1a; border: 0.5px solid #a32d2d; color: #f09595; }
  .gf-ok-badge { background: #0d1f17; border: 0.5px solid #3B6D11; color: #97c459; }
  .gf-row { display: grid; gap: 10px; margin-bottom: 10px; }
  .gf-stat-row { grid-template-columns: repeat(5, 1fr); }
  .gf-chart-row { grid-template-columns: 2fr 1fr; }
  .gf-subnet-row { grid-template-columns: repeat(3, 1fr); }
  .gf-panel { background: #181b27; border: 0.5px solid #2a2d3e; border-radius: 6px; padding: 10px 14px; }
  .gf-panel-title { font-size: 11px; color: #8b92a8; text-transform: uppercase; letter-spacing: 0.06em; margin-bottom: 8px; }
  .gf-stat-val { font-size: 28px; font-weight: 500; color: #e0e3ec; line-height: 1; }
  .gf-stat-sub { font-size: 11px; color: #5c6178; margin-top: 3px; }
  .gf-stat-danger { color: #e24b4a; }
  .gf-stat-warn { color: #ef9f27; }
  .gf-stat-ok { color: #63991f; }
  .gf-stat-info { color: #378add; }
  .gf-legend { display: flex; gap: 14px; margin-top: 6px; flex-wrap: wrap; }
  .gf-legend-item { display: flex; align-items: center; gap: 5px; font-size: 11px; color: #8b92a8; }
  .gf-legend-dot { width: 8px; height: 8px; border-radius: 2px; flex-shrink: 0; }
  .gf-util-bar { height: 8px; background: #1f2232; border-radius: 4px; margin: 6px 0 2px; overflow: hidden; }
  .gf-util-fill { height: 100%; border-radius: 4px; transition: width 0.3s; }
  .gf-alert-line-label { font-size: 10px; color: #a32d2d; text-align: right; margin-top: 2px; }
  .gf-sparkrow { display: flex; gap: 4px; align-items: flex-end; height: 32px; margin: 6px 0 4px; }
  .gf-spark-bar { flex: 1; border-radius: 2px 2px 0 0; min-width: 4px; }
  .gf-divider { border: none; border-top: 0.5px solid #2a2d3e; margin: 4px 0 8px; }
  .gf-time-label { display: flex; justify-content: space-between; font-size: 10px; color: #3d4158; margin-top: 2px; }
  @media (max-width: 768px) {
    .gf-stat-row { grid-template-columns: repeat(2, 1fr); }
    .gf-chart-row { grid-template-columns: 1fr; }
    .gf-subnet-row { grid-template-columns: 1fr; }
  }
</style>

<div class="gf-root">
  <div class="gf-topbar">
    <div>
      <div class="gf-title">Glue ENI — Subnet Utilization</div>
      <div style="font-size:11px;color:#5c6178;margin-top:2px;">glue-eni-observations · Last 6 hours · auto-refresh 60s</div>
    </div>
    <div class="gf-meta">
      <span class="gf-badge">us-east-1</span>
      <span class="gf-badge">Subnet A · B · C</span>
      <span class="gf-badge gf-alert-badge">⚑ 1 alert firing</span>
    </div>
  </div>

  <div class="gf-row gf-stat-row">
    <div class="gf-panel">
      <div class="gf-panel-title">Subnet A — utilization</div>
      <div class="gf-stat-val gf-stat-warn">71%</div>
      <div class="gf-util-bar"><div class="gf-util-fill" style="width:71%;background:#ef9f27;"></div></div>
      <div class="gf-stat-sub">42 / 59 IPs</div>
    </div>
    <div class="gf-panel">
      <div class="gf-panel-title">Subnet B — utilization</div>
      <div class="gf-stat-val gf-stat-ok">34%</div>
      <div class="gf-util-bar"><div class="gf-util-fill" style="width:34%;background:#63991f;"></div></div>
      <div class="gf-stat-sub">20 / 59 IPs</div>
    </div>
    <div class="gf-panel">
      <div class="gf-panel-title">Subnet C — utilization</div>
      <div class="gf-stat-val gf-stat-ok">22%</div>
      <div class="gf-util-bar"><div class="gf-util-fill" style="width:22%;background:#63991f;"></div></div>
      <div class="gf-stat-sub">6 / 27 IPs</div>
    </div>
    <div class="gf-panel">
      <div class="gf-panel-title">Sensor blocks (today)</div>
      <div class="gf-stat-val gf-stat-info">3</div>
      <div class="gf-stat-sub" style="margin-top:6px;">of 60 runs · all recovered</div>
    </div>
    <div class="gf-panel">
      <div class="gf-panel-title">IP exhaustion failures</div>
      <div class="gf-stat-val gf-stat-ok">0</div>
      <div class="gf-stat-sub" style="margin-top:6px;">90-day streak</div>
    </div>
  </div>

  <div class="gf-row gf-chart-row">
    <div class="gf-panel">
      <div class="gf-panel-title">Subnet A — ENI count over time (6h)</div>
      <div style="position:relative;width:100%;height:180px;">
        <canvas id="chartA" role="img" aria-label="Subnet A ENI time series showing in-use, available, and rolling max over 6 hours">ENI counts for Subnet A over 6 hours.</canvas>
      </div>
      <div class="gf-legend">
        <div class="gf-legend-item"><div class="gf-legend-dot" style="background:#378add;"></div>in-use</div>
        <div class="gf-legend-item"><div class="gf-legend-dot" style="background:#ef9f27;"></div>available (cleanup lag)</div>
        <div class="gf-legend-item"><div class="gf-legend-dot" style="background:#97c459;opacity:0.6;"></div>rolling max (buffer)</div>
        <div class="gf-legend-item"><div class="gf-legend-dot" style="background:#e24b4a;border-radius:0;height:2px;width:16px;"></div>80% alert threshold</div>
      </div>
    </div>
    <div class="gf-panel">
      <div class="gf-panel-title">Active alert</div>
      <div style="background:#1a0e0e;border:0.5px solid #a32d2d;border-radius:5px;padding:10px 12px;margin-bottom:10px;">
        <div style="font-size:11px;color:#e24b4a;font-weight:500;margin-bottom:4px;">⚑  subnet-a utilization &gt; 80%</div>
        <div style="font-size:11px;color:#7a3030;">Fired 07:03 · duration 4m 12s</div>
        <div style="font-size:11px;color:#7a3030;margin-top:2px;">manual run in wrong subnet</div>
      </div>
      <div class="gf-panel-title" style="margin-top:8px;">Rolling max (last 5 obs · Subnet A)</div>
      <table style="width:100%;font-size:11px;border-collapse:collapse;">
        <thead><tr style="color:#5c6178;"><th style="text-align:left;padding:3px 0;font-weight:400;">run</th><th style="text-align:right;padding:3px 0;font-weight:400;">total ENIs</th><th style="text-align:right;padding:3px 0;font-weight:400;">in-use</th></tr></thead>
        <tbody style="color:#b0b6cc;">
          <tr><td style="padding:3px 0;">dag_run_20260412_0703</td><td style="text-align:right;">42</td><td style="text-align:right;">30</td></tr>
          <tr><td style="padding:3px 0;">dag_run_20260412_0600</td><td style="text-align:right;">38</td><td style="text-align:right;">20</td></tr>
          <tr><td style="padding:3px 0;">dag_run_20260412_0500</td><td style="text-align:right;">35</td><td style="text-align:right;">20</td></tr>
          <tr><td style="padding:3px 0;">dag_run_20260411_1900</td><td style="text-align:right;">41</td><td style="text-align:right;">20</td></tr>
          <tr><td style="padding:3px 0;">dag_run_20260411_0700</td><td style="text-align:right;">33</td><td style="text-align:right;">20</td></tr>
          <tr style="color:#ef9f27;font-weight:500;border-top:0.5px solid #2a2d3e;"><td style="padding:4px 0;">rolling max →</td><td style="text-align:right;">42</td><td style="text-align:right;"></td></tr>
        </tbody>
      </table>
    </div>
  </div>

  <div class="gf-row gf-subnet-row">
    <div class="gf-panel">
      <div class="gf-panel-title">Subnet A (/26 · us-east-1a) — critical batch</div>
      <div class="gf-sparkrow" id="sparkA"></div>
      <div class="gf-time-label"><span>−6h</span><span>−3h</span><span>now</span></div>
      <hr class="gf-divider"/>
      <div style="display:flex;justify-content:space-between;font-size:11px;color:#8b92a8;margin-top:4px;">
        <span>peak today</span><span style="color:#ef9f27;">48 ENIs</span>
      </div>
      <div style="display:flex;justify-content:space-between;font-size:11px;color:#8b92a8;margin-top:2px;">
        <span>cleanup tail avg</span><span style="color:#d0d4de;">22 min</span>
      </div>
      <div style="display:flex;justify-content:space-between;font-size:11px;color:#8b92a8;margin-top:2px;">
        <span>jobs running</span><span style="color:#378add;">2 × 10 workers</span>
      </div>
    </div>
    <div class="gf-panel">
      <div class="gf-panel-title">Subnet B (/26 · us-east-1b) — non-critical batch</div>
      <div class="gf-sparkrow" id="sparkB"></div>
      <div class="gf-time-label"><span>−6h</span><span>−3h</span><span>now</span></div>
      <hr class="gf-divider"/>
      <div style="display:flex;justify-content:space-between;font-size:11px;color:#8b92a8;margin-top:4px;">
        <span>peak today</span><span style="color:#63991f;">24 ENIs</span>
      </div>
      <div style="display:flex;justify-content:space-between;font-size:11px;color:#8b92a8;margin-top:2px;">
        <span>cleanup tail avg</span><span style="color:#d0d4de;">19 min</span>
      </div>
      <div style="display:flex;justify-content:space-between;font-size:11px;color:#8b92a8;margin-top:2px;">
        <span>jobs running</span><span style="color:#378add;">2 × 10 workers</span>
      </div>
    </div>
    <div class="gf-panel">
      <div class="gf-panel-title">Subnet C (/27 · us-east-1c) — backfill / ad hoc</div>
      <div class="gf-sparkrow" id="sparkC"></div>
      <div class="gf-time-label"><span>−6h</span><span>−3h</span><span>now</span></div>
      <hr class="gf-divider"/>
      <div style="display:flex;justify-content:space-between;font-size:11px;color:#8b92a8;margin-top:4px;">
        <span>peak today</span><span style="color:#63991f;">10 ENIs</span>
      </div>
      <div style="display:flex;justify-content:space-between;font-size:11px;color:#8b92a8;margin-top:2px;">
        <span>cleanup tail avg</span><span style="color:#d0d4de;">31 min</span>
      </div>
      <div style="display:flex;justify-content:space-between;font-size:11px;color:#8b92a8;margin-top:2px;">
        <span>jobs running</span><span style="color:#378add;">1 concurrent</span>
      </div>
    </div>
  </div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.js"></script>
<script>
const labels = ['01:00','02:00','03:00','04:00','05:00','06:00','07:00','07:03','07:30','08:00','08:30','09:00'];
const inUse  = [20,20,20,20,20,22,30,30,20,20,20,20];
const avail  = [0,4,8,12,6,10,12,12,8,6,4,2];
const rmax   = [38,38,38,38,38,40,42,42,42,42,42,42];
const thresh = Array(labels.length).fill(Math.round(59*0.8));

new Chart(document.getElementById('chartA'), {
  type: 'line',
  data: {
    labels,
    datasets: [
      { label:'in-use', data: inUse, borderColor:'#378add', backgroundColor:'rgba(55,138,221,0.12)', fill:true, tension:0.3, pointRadius:2, borderWidth:1.5 },
      { label:'available', data: avail, borderColor:'#ef9f27', backgroundColor:'rgba(239,159,39,0.08)', fill:true, tension:0.3, pointRadius:2, borderWidth:1.5 },
      { label:'rolling max', data: rmax, borderColor:'rgba(151,196,89,0.5)', borderDash:[4,3], fill:false, pointRadius:0, borderWidth:1.2 },
      { label:'80% threshold', data: thresh, borderColor:'#a32d2d', borderDash:[6,4], fill:false, pointRadius:0, borderWidth:1.2 }
    ]
  },
  options: {
    responsive: true, maintainAspectRatio: false,
    plugins: { legend: { display: false }, tooltip: { mode:'index', intersect:false, backgroundColor:'#1f2232', titleColor:'#d0d4de', bodyColor:'#8b92a8', borderColor:'#2f3347', borderWidth:1 } },
    scales: {
      x: { ticks: { color:'#5c6178', font:{size:10}, maxRotation:0, autoSkip:true, maxTicksLimit:6 }, grid:{color:'rgba(255,255,255,0.04)'} },
      y: { min:0, max:59, ticks: { color:'#5c6178', font:{size:10}, stepSize:10 }, grid:{color:'rgba(255,255,255,0.06)'} }
    }
  }
});

function buildSpark(id, data, color) {
  const el = document.getElementById(id);
  const mx = Math.max(...data);
  data.forEach(v => {
    const b = document.createElement('div');
    b.className = 'gf-spark-bar';
    b.style.height = Math.round((v/mx)*100)+'%';
    b.style.background = color;
    b.style.opacity = '0.75';
    el.appendChild(b);
  });
}
buildSpark('sparkA',[20,22,28,48,42,40,42,30,25,22,20,20],'#ef9f27');
buildSpark('sparkB',[20,20,20,20,20,22,24,20,20,20,20,20],'#63991f');
buildSpark('sparkC',[6,4,4,8,10,8,6,6,6,6,6,6],'#63991f');
</script>

## References 

- [AWS Knowledge Center: Resolve "subnet does not have enough free addresses" in Glue](https://repost.aws/knowledge-center/glue-specified-subnet-free-addresses)
- [AWS Glue does not clean up network interfaces](https://repost.aws/questions/QUrmpoQEFxRwWvt5Es6nGmaA/aws-glue-does-not-clean-up-network-interfaces)
- [AWS Glue Job Not Releasing ENIs After Completion](https://repost.aws/questions/QUBdpMaxjcST-voDzcNld3bA/aws-glue-job-not-releasing-enis-after-completion) — 1,200+ accumulated ENIs from the same root cause
- [Glue Connections Running Out of IPs](https://repost.aws/questions/QUE2ZiLoMFSt-tl7OV5pySbg/glue-connections-running-out-of-ips)
- [How to delete a network interface associated with a VPC](https://repost.aws/questions/QUMRV13LI8QJKxcZVL3-1mUg/how-delete-networking-interface-asociate-with-vpc)
- [Unable to Delete Stuck ENI in in-use state](https://repost.aws/questions/QU4vgx-AXzQ_CZcrYjZLEDKg/unable-to-delete-stuck-network-interface-eni-after-elb-deletion-need-aws-intervention)
