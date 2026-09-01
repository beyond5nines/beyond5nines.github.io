---
layout: post
title: "Everything is Awesome! The Tree That Ate the Cache"
date: 2026-04-20 12:00:00 -0000
categories: neo4j apoc observability
series: Everything is Awesome!
redirect_from:
  - /2026/04/20/everything-is-awesome-02/
  - /everything-is-awesome-02/
---

<!-- TODO: Add opening incident alert block — e.g., PagerDuty page or Slack notification showing the initial replication lag alert or transaction exception spike -->

## The One-Second Query

Our upstream tree API was one of the quieter endpoints. A client sends a list of node IDs, we walk the graph upstream to find connected assets, return a tree. Simple read. One second per call. No one had ever flagged it.

The query looked like this:

```cypher
UNWIND $node_ids AS node_id
MATCH (n:Node {id: node_id})
OPTIONAL MATCH path = (n)<-[:UPSTREAM*]-(a:Asset)
WITH collect(path) AS paths, node_id, n
CALL apoc.convert.toTree(paths, false, $config)
YIELD value
OPTIONAL MATCH (n)-[:LOCATED_AT]->(loc:Location)
RETURN collect(value) AS upstream_tree, loc.name AS node_location
```

Runs in about a second. Returns a JSON tree. Planning time: 0 ms. Wait time: 0 ms. By every metric a dashboard would show you, this query is clean.

Then we had an incident. Two read replicas fell behind the primary. Replication lag climbed for eleven hours. 37,000 transaction-tracking exceptions. The team focused on the obvious culprit — a full table scan from the analytics pipeline that read 153 million pages to return 10 rows. That was the trigger. But the page cache never recovered, even after the scan stopped. Something else was holding the cache down.

It was this query. Running every 35 seconds. Evicting 1,750 pages per call. For eleven hours.

## What `apoc.convert.toTree` Actually Does

The name suggests a transformation — take some paths, return a tree. And that's true. But the cost isn't in the transformation. It's in what has to happen before the procedure even starts.

Look at the query again:

```cypher
OPTIONAL MATCH path = (n)<-[:UPSTREAM*]-(a:Asset)
WITH collect(path) AS paths
CALL apoc.convert.toTree(paths, false, $config)
```

Three lines. Three problems.

**Line 1: `[:UPSTREAM*]` — unbounded variable-length traversal.** No depth limit. Cypher will follow `:UPSTREAM` relationships as deep as the graph goes. If a node has five assets one hop upstream, fine. If it has a subtree that fans out 8 levels deep across thousands of assets, every one of those paths gets materialized.

**Line 2: `collect(path)` — full materialization in heap.** Before `toTree` is called, Cypher collects every matched path into a list in JVM heap. Not streamed. Not lazy. The entire path set — every node, every relationship, every property — sits in memory simultaneously.

**Line 3: `apoc.convert.toTree()` — builds nested HashMaps.** The procedure takes the in-memory path list and constructs a recursive tree of Java HashMap objects. Every node's properties get copied into new Map instances. There is no streaming output. The full tree is built in heap before a single row is returned.

The procedure is doing what it says. The problem is that by the time it runs, you've already done the expensive part — and nothing told you.

## What the Production Logs Showed

We pulled query logs for a normal 31-minute window. No incident. Just steady-state traffic. 53 calls to this endpoint, spread across two read replicas.

| Metric | Min | Avg | Max |
|--------|-----|-----|-----|
| Execution time | 1,003 ms | ~1,170 ms | 4,658 ms |
| Page faults | 1,378 | ~1,750 | 4,257 |
| Page hits | 23,380 | ~37,000 | 131,688 |
| CPU time | 129 ms | ~190 ms | 2,200 ms |
| Result size | 6.8 MB | ~10.5 MB | 47.6 MB |
| Planning time | 0 ms | 0 ms | 0 ms |
| Wait time | 0 ms | 0 ms | 0 ms |

The CPU-to-execution ratio averaged 16%. This query was spending 84% of its time waiting — on page cache reads, on disk I/O, on the storage layer fetching pages that weren't cached. This isn't a compute problem. It's an I/O-bound workload that presents as a fast read.

The outlier stands out: 4.6 seconds, 4,257 page faults, 47.6 MB result. That was a node with a deep upstream subtree — the unbounded `[:UPSTREAM*]` chasing every branch. But it wasn't the outlier that caused trouble. It was the other 52 calls, averaging 1,750 page faults each, running every 35 seconds, all day.

## Death by a Thousand Cuts

One call evicting 1,750 pages is noise. Fifty-three calls evicting 92,658 pages in half an hour is a pattern. Over 11 hours, it's a sustained eviction engine.

Here's why the math matters. Neo4j's page cache is a fixed-size LRU. Every page fault means a page was needed but not cached, so the storage engine reads it from disk and evicts the least-recently-used page to make room. When the query pattern is a variable-length traversal across the graph, it reads pages from scattered locations — not the hot working set that other queries depend on.

During the replication lag incident, the timeline looked like this:

1. **The analytics pipeline** ran a full scan — 153 million page reads. This evicted most of the hot working set in one shot. The trigger.
2. **The upstream tree API** kept running — 53 calls per half hour, each scattering ~1,750 page faults across random upstream paths. This was the amplifier.
3. **Checkpoint cycles** kicked in, flushing dirty pages to disk. Each cycle found more dirty pages than the last because the cache was constantly churning. Checkpoint duration grew from 5 minutes to 7.5 minutes — a 48% increase in 41 minutes.
4. **Each checkpoint blocked replication** for 2–4 minutes. The followers couldn't catch up because every time they started recovering, the next checkpoint stalled them again.

The API query didn't cause the incident. It prevented recovery. The page cache never stabilized because every 35 seconds, another 1,750 pages got evicted. The analytics scan was a one-time hit. The tree query was a continuous drain.

And here's the thing that made it invisible: each individual call was a one-second read. No alerts fired. No slow-query threshold was breached. The monitoring saw 53 fast queries. It didn't see 92,658 page faults.

## What Monitoring Didn't Tell Us

We checked every layer:

**APOC's own monitoring** (`apoc.monitor.*`) — four procedures available. They return node/relationship ID counters, kernel metadata, store sizes, and aggregate transaction counts. Not one of them reports per-query page cache impact. Not one of them knows that `toTree` just evicted 1,750 pages.

**`SHOW TRANSACTIONS`** — shows a running `CALL apoc.convert.toTree(...)` with elapsed time and status. If you happen to be looking at the right second, you'd see a one-second read query. By the time it matters, it's already completed and gone.

**Prometheus metrics** — `page_cache.hits`, `page_cache.page_faults`, `page_cache.evictions` are global counters. They tick up. You can see the rate increasing on a dashboard. But there's no label, no tag, no annotation connecting the eviction spike to this specific query. You see the symptom. You don't see the cause.

**The query log** — this is where we found it. Per-query `page_hits`, `page_faults`, `cpu_time`, `execution_time`, `planning_time`, `waiting_time`. We searched for all queries containing `apoc.convert.toTree`, aggregated page faults, and there it was: 14% of all page faults in the window came from one query pattern that nobody had flagged.

The query log was the only forensic tool that worked. Not real-time. Not alertable. After the fact.

## The Fix

**Bounded the traversal.** The single highest-impact change was adding a depth limit:

```cypher
// Before: unbounded
OPTIONAL MATCH path = (n)<-[:UPSTREAM*]-(a:Asset)

// After: bounded to 5 hops
OPTIONAL MATCH path = (n)<-[:UPSTREAM*1..5]-(a:Asset)
```

This caps the worst case. Our data showed that 95%+ of upstream trees are 4 hops or fewer. The remaining deep trees return a truncated result — acceptable for an API response, and far better than evicting the entire page cache.

**Replaced `toTree` with a Cypher projection.** `apoc.convert.toTree` is deprecated — Neo4j recommends `apoc.paths.toJsonTree` as the replacement. But the deeper fix was eliminating the APOC dependency entirely. A custom Cypher projection that builds the tree shape in the `RETURN` clause avoids the in-heap materialization step and lets Neo4j stream results.

**Capped concurrency on the endpoint.** 53 calls in 31 minutes from the API layer meant the application was calling this endpoint on every request without rate limiting. A concurrency cap of 5–10 simultaneous calls prevents the sustained eviction pattern even if individual calls are still expensive.

**Added alerting on per-query page faults.** The query log has the data. A pipeline that tails the log, extracts `page_faults` per query pattern, and alerts when any single pattern exceeds a threshold (e.g., >50,000 cumulative faults per 30-minute window) would have caught this before the incident, not after.

## Results

| Metric | Before | After |
|--------|--------|-------|
| Avg page faults per call | ~1,750 | <!-- TODO: add post-fix measurement --> |
| Max execution time | 4,658 ms | <!-- TODO --> |
| Total page faults / 31-min window | ~92,658 | <!-- TODO --> |
| Checkpoint duration | 7.5 min (degraded) | <!-- TODO --> |
| Cache recovery after incident | Sustained drain (11 hrs) | <!-- TODO --> |

## The Lesson

`apoc.convert.toTree` did exactly what it promised — it built a tree from paths. The documentation is accurate. The function works. The problem was never the procedure. It was the assumption that a one-second read query is harmless.

Fifty-three times in thirty-one minutes, this query scattered reads across the page cache, evicted hot pages that other queries depended on, and prevented the cache from recovering after a separate incident. Each call was invisible to every monitoring layer except the query log. And nobody was watching the query log for a pattern that looked, by every real-time metric, like a fast read.

If you're running `apoc.convert.toTree` — or any APOC procedure that takes a `LIST<PATH>` as input — check what feeds it. If it's a variable-length traversal without a depth limit, you don't have a one-second query. You have an unbounded I/O workload that happens to finish in one second.

`CALL apoc.` is not a function call. It's a workload. Profile it like one.

---

## References

- [APOC `apoc.convert.toTree` Documentation](https://neo4j.com/labs/apoc/5/overview/apoc.convert/apoc.convert.toTree/)
- [APOC `apoc.paths.toJsonTree` (Replacement)](https://neo4j.com/labs/apoc/5/overview/apoc.paths/apoc.paths.toJsonTree/)
- [Neo4j Memory Configuration — Page Cache Sizing](https://neo4j.com/docs/operations-manual/current/performance/memory-configuration/)
- [Neo4j Metrics Reference — Page Cache Metrics](https://neo4j.com/docs/operations-manual/current/monitoring/metrics/reference/)
- [Neo4j Performance — Disks, RAM and Other Tips](https://neo4j.com/docs/operations-manual/current/performance/disks-ram-and-other-tips/)
