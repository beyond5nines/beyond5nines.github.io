---
layout: post
title: "Everything is Awesome!"
title_display: 'Everything is <span class="letter-flicker">A</span>wesome!'
date: 2026-04-18 12:00:00 -0000
categories: neo4j apoc observability
series: Everything is Awesome!
pinned: true
excerpt: "A series on what APOC procedures really cost in production: hidden page cache evictions, GC pauses, and replication lag."
redirect_from:
  - /2026/04/18/everything-is-awesome/
---

APOC — Awesome Procedures On Cypher — is one of the first things you install on a Neo4j cluster. One function call replaces fifty lines of Cypher. Build a tree from paths. Introspect your schema. Batch a million deletes. The developer experience is genuinely good.

The operational experience is a different story.

This is a series about what APOC procedures actually cost when they run on a production cluster — the page cache evictions they don't surface, the heap pressure that spills into GC pauses, the Raft replication lag that follows, and the monitoring gaps that keep you blind until something else breaks.

For two weeks, our followers fired daily Raft replication alerts — 28,103 entries behind, nearly double the 15,000 threshold. Clients hit `TransactionIdTrackerException: Database 'graphdb' not up to the requested version` on reads. The debug logs pointed at replication lag. The query logs pointed at APOC. Nobody connected them until we aligned timestamps manually.

**APOC gives you developer ergonomics. It doesn't give you operational visibility.**

---

## In This Series

### Part 1: [The Tree That Ate the Cache](/everything-is-awesome-01/)

An API endpoint builds a tree from graph paths using `apoc.convert.toTree`. One second per call. No alerts fire. Behind the scenes, each call evicts 1,750 pages from cache. Over 31 minutes, 53 calls scatter 92,658 page faults across the store — preventing the page cache from recovering after a separate incident, for eleven hours.

The fix: depth-limited traversals, replacing the procedure with a custom Cypher projection, and concurrency caps on the calling endpoint.

---

### Part 2: [The Schema Query That Wasn't](/everything-is-awesome-02/)

A schema introspection call runs on every Spark job connect — five APOC calls before the first application query. Returns 39 KB of result data. Causes 56,000 page faults. `apoc.meta.nodeTypeProperties` and `apoc.meta.relTypeProperties` don't read metadata — they sample real nodes and relationships across every label in the database, scattering reads across the entire store.

The fix: explicit sample limits, label filtering, caching the result application-side, and upgrading the Spark connector to v5.x which uses built-in schema procedures instead of APOC sampling. Connect-phase page faults dropped from ~56,000 to under 2,000.

---

### Part 3: [The Monitoring Gap](/everything-is-awesome-03/)

Five monitoring layers — application dashboards, Prometheus metrics, APOC's own monitoring procedures, `SHOW TRANSACTIONS`, and the troubleshooting runbook — all missed APOC as a contributor to the February 17 thread exhaustion incident. A single `toTree` call caused 9,001 page faults and consumed 42 seconds of CPU. A wide-open `allowlist=apoc.*` let any client call any procedure. The monitoring saw a one-second read.

The fix: query log pipelines with Vector and VictoriaMetrics, APOC-specific dashboard panels, cumulative page fault alerting by query pattern, tightening the procedure allowlist to an explicit set, and a runbook update that adds cumulative impact analysis alongside single-query triage.

---

*Every `CALL apoc.` is a workload, not a function call. This series documents the operational cost of convenience — the gaps Neo4j doesn't fill, and the signals you have to build yourself.*
