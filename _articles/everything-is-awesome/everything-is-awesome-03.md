---
layout: post
title: "Everything is Awesome! The Monitoring Gap"
short_title: "The Monitoring Gap"
date: 2026-04-24 12:00:00 -0000
categories: neo4j apoc observability
series: Everything is Awesome!
excerpt: "Five monitoring layers, none of which tracked APOC procedure cost. A StackOverflowError inside apoc.convert.toTree preceded thread starvation by two minutes — and nothing alerted."
redirect_from:
  - /2026/04/24/everything-is-awesome-03/
---

## The Stack That Saw Everything

Our observability stack was not trivial. Grafana dashboards with Prometheus metrics. ELK for application and access logs. VictoriaMetrics for long-term storage. Vector agents on every node, shipping structured logs through an aggregation pipeline. A troubleshooting runbook that walked you through six layers of triage — from dashboard panels to pod health to Neo4j query logs — with escalation decision trees at the bottom.

We had monitoring for CPU, memory, disk, pod restarts, unhealthy targets, HPA scaling, error rates, P99 latency, response time by client, request rate by status code, checkpoint duration, page cache hit ratios, and Raft replication lag. We had alerts. We had dashboards. We had runbooks.

None of it told us that an APOC procedure was about to take down the cluster.

## Five Layers, One Blind Spot

Here's what each monitoring layer sees when `apoc.convert.toTree` runs — and what it doesn't.

### Layer 1: Application Dashboards (Grafana)

The API dashboard shows request rate, error rate, P99 latency, and response time by endpoint. When the upstream tree API runs `apoc.convert.toTree` and returns in one second, the dashboard sees a successful request with normal latency. When it runs 53 times in 31 minutes, the dashboard sees 53 successful requests.

**What's missing:** No connection between the API response and the database cost. The dashboard doesn't know that each one-second response evicted 1,750 pages from cache. There's no panel for "cumulative page faults by endpoint" because that metric doesn't exist at the application layer.

### Layer 2: Infrastructure Metrics (Prometheus)

Neo4j exposes `page_cache.hits`, `page_cache.page_faults`, `page_cache.evictions`, `page_cache.hit_ratio` as global counters. You can see the rate increasing on a time-series graph. During the incidents described in [Part 1](/everything-is-awesome-01/) and [Part 2](/everything-is-awesome-02/), the page fault rate climbed. The eviction rate climbed. The hit ratio dropped.

**What's missing:** No attribution. The counters tick up, but there's no label, no tag, no annotation connecting the eviction spike to a specific query, procedure, or service account. You see the symptom on the dashboard. You open an investigation. You check CPU (normal), memory (normal), disk I/O (elevated, but why?). The Prometheus metric tells you *that* the cache is churning. It doesn't tell you *what* is churning it.

### Layer 3: APOC's Own Monitoring

APOC ships four monitoring procedures: `apoc.monitor.ids`, `apoc.monitor.kernel`, `apoc.monitor.store`, and `apoc.monitor.tx`. They return node/relationship ID counters, kernel metadata, store sizes, and aggregate transaction counts. We checked all four.

**What's missing:** Everything that matters. Not one of these procedures reports per-query page cache impact. Not one of them knows that `apoc.convert.toTree` just evicted 1,750 pages. Not one of them distinguishes a one-second metadata read from a 48-second full-database sampling pass. APOC provides procedures that generate enormous I/O workloads, and a monitoring surface that is completely blind to the I/O those workloads generate.

### Layer 4: `SHOW TRANSACTIONS`

If you happen to be watching at the right moment, `SHOW TRANSACTIONS` shows a running `CALL apoc.convert.toTree(...)` with elapsed time and status. You'd see a one-second read query. One second later, it's gone. The next call starts. By the time 53 calls have evicted 92,000 pages, every individual transaction has long since completed and disappeared from the output.

**What's missing:** History. `SHOW TRANSACTIONS` is a point-in-time snapshot. There's no "show me all transactions that ran in the last 30 minutes." There's no "show me cumulative page faults by query pattern." You have to be looking at the right second to catch a one-second query. And even if you catch it, you see elapsed time — not page faults, not eviction count, not heap allocation.

### Layer 5: The Troubleshooting Runbook

Our runbook — a document the team maintained for 504 investigation — had six sections: dashboard triage, application log investigation, Neo4j query log deep-dive, escalation decision trees. The classification table at the end mapped symptoms to root causes:

| Symptom | Likely Root Cause |
|---------|-------------------|
| P99 spike + high response time | Slow Neo4j queries starving workers |
| CPU/Memory near limit | Query regression or traffic surge |
| Single client driving all traffic | Runaway consumer |

**What's missing:** The runbook reaches the query log in Part 3 — Step 2 says "sort by `total_time_ms` DESC." This finds the slowest individual queries. It does not find the query that ran 53 times at normal speed and collectively evicted the page cache. The runbook's implicit model is that the problem query is the slow one. `apoc.convert.toTree` was never the slow one. It was the frequent one.

## The Incident That Proved It

On February 17, the graph database node on `ip-10-80-61-236` experienced thread pool exhaustion at 12:52 UTC. The API service returned HTTP 500 errors. The incident analysis identified the primary cause: the analytics pipeline's Spark service account ran multiple unbounded full-graph scans that held Bolt threads for 19–30 minutes, exhausting the thread pool.

But thread starvation was first detected at 12:26 — **26 minutes before** the main incident. No Spark queries were running yet. The pre-incident logs showed normal traffic from two service accounts: the Lambda functions and the API service. The incident analysis listed the likely cause of the early starvation: `apoc.convert.toTree` queries from the API service account.

Here's what the query log showed in the 40 minutes before the main incident:

### Pre-Incident `toTree` Calls

| Time | Execution | Page Faults | Page Hits | CPU Time |
|------|-----------|-------------|-----------|----------|
| 12:12:00 | 47,881 ms | 9,001 | 793,601 | 42,335 ms |
| 12:15:17 | 1,723 ms | 0 | 6 | — |
| 12:17:50 | 1,137 ms | 1,141 | 67,967 | 279 ms |
| 12:26:29 | 1,584 ms | 131 | 5,084 | — |
| 12:29:37 | 2,860 ms | 1,354 | 104,571 | — |

The first call stands out: **48 seconds, 9,001 page faults, 793,601 page hits, 42 seconds of CPU time.** This was a single `toTree` call that hit a deep upstream subtree — the same unbounded `[:UPSTREAM*]` pattern from Part 1, except this one found a subtree large enough to read 793,000 pages and fault 9,000 more. It consumed 42 seconds of CPU on a node that was simultaneously serving API traffic.

The call at 12:17 hit 67,967 pages with 1,141 faults. The call at 12:29 hit 104,571 pages with 1,354 faults. Between them, these three calls alone accounted for over 965,000 page reads and 11,500 page faults — in 18 minutes, from a single query pattern that each monitoring layer classified as "normal read traffic."

Nine minutes after the 12:26 thread starvation warning, the Spark queries started. Twenty-six minutes later, the service was down.

### The StackOverflowError

At 12:50:38 — two minutes before thread starvation collapsed the service — the query log recorded this:

```
Failed to invoke procedure `apoc.convert.toTree`:
  Caused by: java.lang.StackOverflowError
```

| Metric | Value |
|--------|-------|
| Execution time | 2,366 ms |
| Page faults | 212 |
| Page hits | 751,184 |
| Error | `java.lang.StackOverflowError` |
| Service account | API service account |

The procedure hit a recursive tree structure deep enough to overflow the JVM call stack. 751,184 page hits — three-quarters of a million pages read into memory for a single tree construction that ultimately failed. And this was a *failed* call. The successful calls before it were doing the same work without the stack overflow, silently.

This was the only signal the monitoring stack actually caught — because it was an error. The procedure finally failed loudly enough to leave a trace. But by 12:50, the node was already 24 minutes into the Spark-driven thread exhaustion. The `toTree` error was a late symptom, not an early warning. And even this error didn't trigger an alert — it was found in the query log during the post-incident investigation.

## Why APOC Creates a Monitoring Gap

The gap isn't accidental. It's structural.

**APOC procedures run inside the Neo4j process.** They execute as JVM code within the database server. From the Bolt protocol's perspective, the client sent a query and received a result. The Prometheus metrics see aggregate counters increment. Neither layer has visibility into what happened *inside* the procedure call.

**APOC wraps internal operations into opaque calls.** When you write `CALL apoc.convert.toTree(paths, false, $config)`, the query planner sees a procedure call. It doesn't see the recursive HashMap construction, the heap allocation, the page cache reads that the procedure triggers internally. The planning time is 0 ms — because there's nothing to plan. The actual work happens inside Java code that the query engine doesn't instrument.

**The query log is the only layer with per-call metrics.** The Neo4j query log records `execution_time`, `page_faults`, `page_hits`, `cpu_time`, `result_size`, and `waiting_time` for every query that exceeds the configured threshold. This is the only place where you can see that a specific `toTree` call caused 9,001 page faults. But the query log is a file on disk. It's not real-time. It's not alertable out of the box. It requires a pipeline to parse, aggregate, and surface the data — a pipeline that most teams don't build because the dashboards look comprehensive enough.

**The runbook model assumes the problem query is slow.** Every troubleshooting guide — ours included — starts with "find the slowest queries." Sort by execution time. Look for timeouts. Check for lock contention. This model catches a 30-minute Spark scan. It doesn't catch a one-second procedure that runs 53 times and collectively evicts more pages than the 30-minute scan. The damage model for APOC procedures is cumulative, not acute. The investigation model is acute, not cumulative.

## The Configuration That Made It Worse

The monitoring gap compounds when the cluster doesn't restrict which APOC procedures can run in the first place. Our Neo4j configuration had this:

```properties
dbms.security.procedures.allowlist=apoc.*
dbms.security.procedures.unrestricted=apoc.*
```

Every APOC procedure — all of them — was callable by every service account, with no restrictions. `allowlist=apoc.*` means every procedure in the APOC jar is available. `unrestricted=apoc.*` means every procedure runs with full access to the database internals, bypassing the sandbox that Neo4j applies to user-defined procedures by default.

This is the default most teams ship with, because APOC is treated as a trusted library. And for development, it is. But on a production cluster, the combination of a wide-open allowlist and a blind monitoring stack means any client — the API service, the analytics pipeline, an interactive browser session — can call any APOC procedure, and no monitoring layer will distinguish a harmless `apoc.text.join` from a `apoc.meta.nodeTypeProperties` that scatters 56,000 page faults across the store.

The allowlist is the one mechanism Neo4j provides to **control** the APOC surface area. If `apoc.convert.toTree` and `apoc.meta.nodeTypeProperties` had been excluded from the allowlist, the Spark connector would have failed on connect with a clear `ProcedureNotFound` error — a loud, immediate signal instead of a silent 56,000-page-fault sampling pass. The API service would have gotten an error instead of a one-second response that quietly evicted the page cache. Both failures would have been caught during integration testing, not during a production incident.

Instead, the allowlist let everything through, and the monitoring stack saw nothing. Two blind spots, reinforcing each other.

## The Fix

### 1. Query Log Pipeline

The query log has the data. The gap was that nobody was reading it in near-real-time. We built a pipeline:

- **Vector agent** on each Neo4j node tails `query.log`, parses the structured fields (execution time, page faults, page hits, CPU time, query text, service account).
- **Aggregation layer** groups by query pattern (normalized query text with parameters stripped) and computes rolling 30-minute windows: total page faults, total execution time, call count, max single-call page faults.
- **Alert rules** fire when any query pattern exceeds thresholds:
  - Cumulative page faults > 50,000 per 30-minute window
  - Single-call page faults > 5,000
  - Single-call execution time > 30 seconds
  - Any query with `apoc.` in the text exceeding 1,000 page faults per call

### 2. APOC-Specific Dashboard Panels

Added Grafana panels that specifically track APOC procedure behavior:

| Panel | Source | What It Shows |
|-------|--------|---------------|
| APOC calls per 30m window | Query log pipeline | Call frequency by procedure name |
| APOC page faults (cumulative) | Query log pipeline | Total page cache cost per procedure |
| APOC execution time (P99) | Query log pipeline | Tail latency by procedure |
| Page cache hit ratio vs APOC call rate | Prometheus + query log | Correlation between APOC activity and cache health |

The correlation panel was the most useful. It overlays the global page cache hit ratio from Prometheus with the APOC call rate from the query log. When the hit ratio drops and the APOC rate is elevated, the dashboard shows the relationship that was previously invisible.

### 3. Service Account Attribution

The query log includes the authenticated service account for every query. We added a `service_account` dimension to the pipeline aggregation, which answers the question "which service is causing the most page cache damage right now?" — a question that previously required manually grepping through log files during an incident.

### 4. Runbook Update

Updated the troubleshooting runbook to add a step between "find the slowest queries" and "classify the bottleneck":

> **Step 2.5 — Check cumulative impact by query pattern.** Group queries by normalized pattern. Sort by total page faults, not by individual execution time. A query that runs 50 times at 1 second each with 1,750 page faults per call is more damaging than a single query that runs for 30 seconds with 2,000 page faults. Look for `apoc.` calls specifically — they wrap internal operations that don't appear in planning time or wait time metrics.

### 5. Allowlist Tightening

Replaced the blanket `apoc.*` allowlist with an explicit list of the APOC procedures the cluster actually needs:

```properties
# Before
dbms.security.procedures.allowlist=apoc.*
dbms.security.procedures.unrestricted=apoc.*

# After
dbms.security.procedures.allowlist=apoc.convert.toTree,apoc.text.*,apoc.coll.*
dbms.security.procedures.unrestricted=apoc.convert.toTree
```

Any procedure not on the list now returns `ProcedureNotFound` — a loud failure at integration test time instead of a silent cost at production time. The `apoc.meta.*` procedures were removed entirely; the Spark connector upgrade to v5.x eliminated the dependency, and no other client needs them. If a new service tries to call `apoc.meta.nodeTypeProperties`, it gets an error and a conversation — not a 56,000-page-fault sampling pass that nobody sees.

## Results

| Metric | Before | After |
|--------|--------|-------|
| Time to identify APOC as incident contributor | Post-incident (hours–days) | < 5 minutes (alert) |
| APOC page fault visibility | Query log only (manual grep) | Dashboard + alert pipeline |
| APOC procedures callable by any client | All (~300+ procedures) | Explicit allowlist (~15 procedures) |
| Runbook coverage of cumulative patterns | Not addressed | Explicit step with examples |
| Mean time to attribute cache eviction source | Manual log correlation | Automated service account attribution |

The February 17 incident would have looked different with this pipeline. The 12:12 `toTree` call — 48 seconds, 9,001 page faults — would have fired the single-call alert immediately. The cumulative pattern from 12:12 to 12:29 — 11,500 page faults from one query pattern in 18 minutes — would have fired the rolling-window alert before the Spark queries started. The early thread starvation at 12:26 might still have happened, but the investigation would have started with "which query pattern is causing page cache pressure?" instead of ending there.

## The Lesson

The monitoring gap for APOC procedures isn't a missing feature. It's a missing model. The standard model — dashboards show aggregate metrics, alerts fire on thresholds, investigations start with the slowest query — works for queries that the database engine plans, executes, and instruments natively. It breaks for procedures that wrap arbitrary JVM code behind a `CALL` statement.

APOC gives you developer ergonomics. A single `CALL` replaces dozens of lines of Cypher. But that `CALL` also replaces the visibility that those dozens of lines would have given you — per-step execution time, per-step page access, per-step memory allocation. The procedure trades operational transparency for developer convenience, and every monitoring layer inherits that trade-off.

The query log is the only forensic surface that survives the abstraction. It records what the procedure actually cost — but only after the fact, only if you're reading it, and only if you know that a one-second read query can be the most expensive thing running on your cluster.

If your monitoring stack doesn't parse the query log for APOC-specific patterns, you don't have a monitoring gap. You have a blind spot the exact shape of your most expensive procedures — and you won't find it until something else breaks.

---

## References

- [Neo4j Query Logging Configuration](https://neo4j.com/docs/operations-manual/current/monitoring/logging/query-logging/)
- [Neo4j Metrics Reference — Page Cache Metrics](https://neo4j.com/docs/operations-manual/current/monitoring/metrics/reference/)
- [Neo4j Procedure Allowlisting](https://neo4j.com/docs/operations-manual/current/security/securing-extensions/)
- [APOC Monitoring Procedures](https://neo4j.com/labs/apoc/5/overview/apoc.monitor/)
- [Neo4j `SHOW TRANSACTIONS` Command](https://neo4j.com/docs/cypher-manual/current/clauses/transaction-clauses/)
- [Vector Log Agent — Neo4j Log Parsing](https://vector.dev/docs/reference/configuration/sources/file/)
- [Grafana Dashboard Annotations from Log Events](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/annotate-visualizations/)
