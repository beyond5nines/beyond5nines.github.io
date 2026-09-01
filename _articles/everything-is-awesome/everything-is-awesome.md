---
layout: post
title: "Everything is Awesome!"
title_display: 'Everything is <span class="letter-flicker">A</span>wesome!'
date: 2026-04-18 12:00:00 -0000
categories: neo4j apoc observability
series: Everything is Awesome!
pinned: true
redirect_from:
  - /2026/04/18/everything-is-awesome/
---

## The Series

**"APOC gives you developer ergonomics. It doesn't give you operational visibility."**

APOC — Awesome Procedures On Cypher — is one of the first things you install on a Neo4j cluster. One function call replaces fifty lines of Cypher. Build a tree from paths. Introspect your schema. Batch a million deletes. The developer experience is genuinely good.

The operational experience is a different story.

This is a series about what APOC procedures actually cost when they run on a production cluster — the page cache evictions they don't surface, the heap pressure that spills into GC pauses, the Raft replication lag that follows, and the monitoring gaps that keep you blind until something else breaks.

For two weeks, our followers fired daily Raft replication alerts — 28,103 entries behind, nearly double the 15,000 threshold. Clients hit `TransactionIdTrackerException: Database 'graphdb' not up to the requested version` on reads. The debug logs pointed at replication lag. The query logs pointed at APOC. Nobody connected them until we aligned timestamps manually.

---

## Posts in This Series

### Part 1: [The Tree That Ate the Cache](/everything-is-awesome-01/)
**The problem:** An API endpoint builds a tree from graph paths. One second per call. No alerts fire. No slow-query threshold is breached. Behind the scenes, each call evicts 1,750 pages from cache and pressures GC. Over an hour, Raft replication lag spikes to 8,000 records. Followers start rejecting reads: `TransactionIdTrackerException: Database 'graphdb' not up to the requested version`. The monitoring saw fast reads. The cluster saw a sustained eviction engine — for two weeks.

**The root cause:** `apoc.convert.toTree` materializes all paths in heap before processing — but the real damage is the unbounded variable-length traversal feeding it, which turns every call into a scattered read that evicts the hot working set, stalls GC, and starves Raft replication.

**The fix:** Depth-limited traversals, replacing the procedure with a custom Cypher projection, and concurrency caps on the calling endpoint.

![Raft replication lag correlated with toTree execution windows over 2 weeks](/assets/images/everything-is-awesome/raft-lag-correlation.png)
*Daily Raft alerts (28,103 entries, threshold 15,000) aligned to toTree execution windows in the query log.*

---

### Part 2: [The Schema Query That Wasn't](/everything-is-awesome-02/)
**The problem:** A schema introspection call runs on every Spark job connect — five APOC calls before the first application query. Returns 8–18 KB per call. Takes 3–14 seconds each. Causes 56,000 page faults per connect session for 39 KB of result data.

**The root cause:** `apoc.meta.nodeTypeProperties` and `apoc.meta.relTypeProperties` don't read metadata — they sample real nodes and relationships across every label in the database, scattering reads across the entire store.

**The fix:** Explicit sample limits, label filtering, caching the result application-side, and upgrading the Spark connector to v5.x which uses built-in schema procedures instead of APOC sampling.

---

### Part 3: [The Monitoring Gap](/everything-is-awesome-03/)
**The problem:** Five monitoring layers — application dashboards, Prometheus metrics, APOC's own monitoring procedures, `SHOW TRANSACTIONS`, and the troubleshooting runbook — all missed APOC as a contributor to the February 17 thread exhaustion incident. A single `toTree` call at 12:12 UTC caused 9,001 page faults and consumed 42 seconds of CPU. The monitoring saw a one-second read.

**The root cause:** APOC procedures wrap arbitrary JVM code behind a `CALL` statement, trading operational transparency for developer convenience. Every monitoring layer inherits that trade-off. A wide-open `allowlist=apoc.*` configuration compounds the gap — any client can call any procedure, and the monitoring stack is equally blind to all of them. The only forensic surface is the query log — and nobody was reading it in near-real-time.

**The fix:** Query log pipelines with Vector and VictoriaMetrics, APOC-specific dashboard panels, cumulative page fault alerting by query pattern, tightening the procedure allowlist from `apoc.*` to an explicit set, and a runbook update that adds cumulative impact analysis alongside single-query triage.

---

## The Pattern

Every post in this series follows the same shape:

1. **The call** — what it looks like in your code, and why nobody questioned it
2. **The cost** — what it actually does to your cluster, measured from production logs
3. **The blind spot** — what monitoring doesn't tell you, and why
4. **The fix** — what we changed, and what we'd do differently

If you've ever assumed an APOC procedure was "just a function call" and later found it at the center of a production incident, this series is for you.

---

*Follow along as we document the operational cost of convenience — the APOC gaps Neo4j doesn't fill, and the signals you have to build yourself.*
