---
layout: post
title: "Everything is Awesome!"
title_display: 'Everything is <span class="letter-flicker">A</span>wesome!'
date: 2026-04-18 12:00:00 -0000
categories: neo4j apoc observability
series: Everything is Awesome!
pinned: true
published: false
redirect_from:
  - /2026/04/18/everything-is-awesome/
---

## The Series

**"APOC gives you developer ergonomics. It doesn't give you operational visibility."**

APOC — Awesome Procedures On Cypher — is one of the first things you install on a Neo4j cluster. One function call replaces fifty lines of Cypher. Build a tree from paths. Introspect your schema. Batch a million deletes. The developer experience is genuinely good.

The operational experience is a different story.

This is a series about what APOC procedures actually cost when they run on a production cluster — the page cache impact they don't surface, the heap pressure they don't report, and the monitoring gaps that keep you blind until something else breaks.

---

## Posts in This Series

### Part 1: [The Tree That Ate the Cache](/everything-is-awesome-01/)
**The problem:** An API endpoint builds a tree from graph paths. One second per call. 53 calls in 31 minutes. Each call evicts 1,750 pages from cache. The cache never recovers.

**The root cause:** `apoc.convert.toTree` materializes all paths in heap before processing — but the real damage is the unbounded variable-length traversal feeding it, which turns every call into a scattered read across the page cache.

**The fix:** Depth-limited traversals, replacing the procedure with a custom Cypher projection, and concurrency caps on the calling endpoint.

---

### Part 2: [The Schema Query That Wasn't](/everything-is-awesome-02/)
**The problem:** A schema introspection call runs on every Spark job connect. Returns 8 KB. Takes 3.7 seconds. Reads 87,000 pages.

**The root cause:** `apoc.meta.nodeTypeProperties` doesn't read metadata — it samples real nodes across every label in the database, scattering reads across the entire store.

**The fix:** Explicit sample limits, label filtering, and caching the result application-side instead of re-querying on every connect.

---

### Part 3: [The Monitoring Gap](/everything-is-awesome-03/)
**The problem:** APOC's own monitoring exposes zero per-query metrics. Prometheus sees global counters. `SHOW TRANSACTIONS` sees an opaque procedure call. Nobody connects the procedure to the cache eviction until it's too late.

**The root cause:** APOC procedures that wrap internal operations — batching, tree-building, schema sampling — are invisible to every monitoring layer. The only forensic tool is the query log, after the fact.

**The fix:** Query log analysis pipelines, page cache correlation dashboards, and treating every `CALL apoc.` as a workload, not a function call.

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
