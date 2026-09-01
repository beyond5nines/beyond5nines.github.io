---
layout: post
title: "Everything is Awesome! The Schema Query That Wasn't"
date: 2026-04-22 12:00:00 -0000
categories: neo4j apoc observability
series: Everything is Awesome!
redirect_from:
  - /2026/04/22/everything-is-awesome-02/
---

## The 8 KB Connect Tax

Every Spark job that reads from our graph database starts with a connect phase. The driver opens a Bolt session, negotiates protocol, and — before executing the first application query — introspects the schema. This is the neo4j-spark-connector doing its job: it needs to know what labels exist, what properties they have, and what types those properties are, so it can map graph data to DataFrame columns.

The introspection looked like this in the query log:

```cypher
CALL apoc.meta.nodeTypeProperties($config)
YIELD propertyName, propertyTypes
WITH DISTINCT propertyName, propertyTypes
WITH propertyName, collect(propertyTypes) AS propertyTypes
RETURN propertyName, reduce(acc = [], elem IN propertyTypes | acc + elem)
  AS propertyTypes
```

Returned 8–18 KB. Ran once per connect. Took 3–14 seconds. Nobody had ever flagged it because it happened during the connect phase — before any application query, outside any dashboard SLA.

Then we looked at the page faults.

## What `apoc.meta.nodeTypeProperties` Actually Does

The name suggests it reads metadata. The [documentation](https://neo4j.com/labs/apoc/5/overview/apoc.meta/apoc.meta.nodeTypeProperties/) says it "returns property type information for nodes in the database." Both are technically true. But the implementation doesn't query a metadata catalog. Neo4j doesn't maintain a centralized property schema the way a relational database does. Properties are schema-free — any node can have any property.

So the procedure does the next best thing: it **samples real nodes**. For every label in the database, it reads a configurable number of nodes (default: 100 per label), inspects their properties, and infers the schema from the sample. This means:

1. **Every label gets visited.** If your database has 30 labels, the procedure makes 30 sampling passes.
2. **Each pass reads real node pages.** Not an index. Not a catalog. Actual node store pages, scattered across the data files.
3. **The sampling has no locality.** Nodes of the same label aren't stored contiguously. Each sample set pulls pages from random locations in the store.

The result is a scatter-read pattern that touches a cross-section of the entire database. For a database with millions of nodes across dozens of labels, this means tens of thousands of page reads — for a result that fits in a single Slack message.

## The Companion: `apoc.meta.relTypeProperties`

`nodeTypeProperties` wasn't alone. The Spark connector also runs a second introspection call:

```cypher
CALL apoc.meta.relTypeProperties($config)
YIELD sourceNodeLabels, targetNodeLabels, propertyName, propertyTypes
WITH * WHERE sourceNodeLabels = $sourceLabels
  AND targetNodeLabels = $targetLabels
RETURN *
```

Same principle. Same sampling behavior. Except now it's sampling relationships — which are stored separately from nodes, in their own store files, with their own page cache footprint. Each call scatters reads across the relationship store the same way `nodeTypeProperties` scatters reads across the node store.

Together, a single Spark connect session ran **five APOC introspection calls**: three `nodeTypeProperties` calls and two `relTypeProperties` calls. The connector runs them automatically. They don't appear in application code. They appear in the query log, if you know to look for them.

## What the Production Logs Showed

We pulled query logs for the daily analytics pipeline connect window — 03:36 UTC, every day. The pattern was identical across days.

### `nodeTypeProperties` — Three Calls per Connect

| Date | Call | Execution | Planning | Page Hits | Page Faults | Result Size |
|------|------|-----------|----------|-----------|-------------|-------------|
| Jan 20 | 1/3 | 7,533 ms | 6 ms | 52,953 | 7,443 | 13,448 B |
| Jan 20 | 2/3 | 13,819 ms | 0 ms | 15,645 | 14,981 | 7,832 B |
| Jan 20 | 3/3 | 9,342 ms | 0 ms | 76,189 | 9,315 | 17,640 B |
| Jan 26 | 1/3 | 4,722 ms | 6 ms | 55,462 | 3,833 | 12,848 B |
| Jan 26 | 2/3 | 12,685 ms | 6 ms | 14,485 | 14,968 | 7,832 B |
| Jan 26 | 3/3 | 8,493 ms | 6 ms | 80,721 | 9,277 | 17,640 B |
| Jan 30 | 1/3 | 5,291 ms | 6 ms | 64,649 | 5,654 | 14,064 B |
| Jan 30 | 2/3 | 12,374 ms | 0 ms | 15,729 | 14,967 | 7,832 B |
| Jan 30 | 3/3 | 8,093 ms | 0 ms | 75,326 | 9,298 | 17,640 B |
| Jan 31 | 1/3 | 7,534 ms | 37 ms | 55,983 | 7,452 | 12,848 B |
| Jan 31 | 2/3 | 13,764 ms | 0 ms | 14,282 | 15,084 | 7,832 B |
| Jan 31 | 3/3 | 9,482 ms | 0 ms | 74,062 | 9,536 | 17,640 B |

Three things stand out:

**The second call is always the most expensive.** ~13 seconds, ~15,000 page faults. The first call warms some pages into cache; the second call samples a different set of labels (or the same labels with a different sample seed) and finds cold pages. By the third call, enough of the store is cached that it's slightly faster.

**Page faults per session total ~28,000–32,000.** Three calls, running sequentially, each scattering reads across the node store. For 30–40 KB of result data. That's roughly 800 page faults per kilobyte of useful output.

**The pattern is identical every day.** Same time. Same cost. Same page fault profile. This is a fixed tax on every connect, paid regardless of what the actual application query does.

### `relTypeProperties` — The Second Sampling Pass

On days when the connector also introspected relationships, the additional cost was significant:

| Date | Execution | Page Hits | Page Faults | Result Size |
|------|-----------|-----------|-------------|-------------|
| Feb 2 | 7,365 ms | 12,728 | 11,503 | 312 B |
| Feb 2 | 4,280 ms | 16,864 | 6,664 | 312 B |
| Feb 2 | 4,080 ms | 17,544 | 8,055 | 312 B |

312 bytes of result. 7 seconds. 11,503 page faults. This procedure returned a single relationship type's properties — and scattered reads across the entire relationship store to do it.

**Combined connect-phase cost:** 5 introspection calls, ~25–30 seconds of execution time, ~45,000–50,000 page faults. Before a single row of application data was read.

## The I/O-to-Value Ratio

To put the numbers in perspective:

| Metric | nodeTypeProperties (3 calls) | relTypeProperties (2–3 calls) | Combined |
|--------|-------------------------------|-------------------------------|----------|
| Total execution time | ~26 s | ~16 s | ~42 s |
| Total page faults | ~30,000 | ~26,000 | ~56,000 |
| Total page hits | ~155,000 | ~47,000 | ~202,000 |
| Total result data | ~38 KB | ~1 KB | ~39 KB |
| Page faults per KB of result | ~790 | ~26,000 | ~1,436 |

For comparison, our tree API query from [Part 1](/everything-is-awesome-01/) — the one that ran 53 times in 31 minutes and prevented the page cache from recovering — averaged 1,750 page faults per call and returned ~10.5 MB. The schema introspection caused 56,000 page faults for 39 KB. It was less efficient by three orders of magnitude, measured by useful data returned per page fault.

The difference: the tree API was called 53 times in 31 minutes. The schema introspection was called 5 times in 30 seconds. But those 5 calls happened at 03:36 UTC — the exact moment the analytics pipeline's first Spark job connected. And every subsequent Spark job in the pipeline paid the same connect tax again.

## Why Nobody Noticed

Four reasons this stayed invisible:

**1. It runs during connect, not during queries.** The Spark connector introspects the schema before executing the first `MATCH`. Application developers profile their queries; they don't profile the connection handshake. The 30-second connect phase was attributed to "Spark startup overhead."

**2. It runs automatically.** This isn't a `CALL` in application code. The neo4j-spark-connector runs it internally when establishing a session. It doesn't appear in the application's query files, pull requests, or code reviews. You find it only in the Neo4j query log.

**3. The query completes successfully.** No errors. No timeouts. The procedure returns valid data. Monitoring systems that alert on errors, slow queries, or failures see nothing. A 13-second query that runs once per connect doesn't breach a slow-query threshold set for 30 seconds.

**4. The result is tiny.** 8–18 KB per call. If you look at the result size, it's indistinguishable from a trivial metadata lookup. You have to look at the page fault column — a metric most teams don't monitor per-query — to see the cost.

## The Fix

**Added explicit sample limits to the connector configuration.** The `$config` parameter accepts a `sample` key that limits how many nodes are sampled per label. The default is 100, which seems small until you multiply it by 30+ labels and realize each sample reads pages from random locations. Setting `sample: 10` reduced page faults by ~70% with no measurable loss in schema accuracy for our use case.

**Filtered labels.** The connector doesn't need to know about every label in the database — only the ones the Spark job actually reads. Adding a label filter to the connector configuration eliminated sampling of labels that the job never touches.

**Cached the result application-side.** Schema doesn't change between pipeline runs. We cached the introspection result at the orchestrator level and passed the schema to downstream jobs via configuration rather than re-querying the database on every connect.

**Upgraded the Spark connector.** Version 5.x of the neo4j-spark-connector changed the introspection behavior. The newer version uses `db.schema.nodeTypeProperties()` — a built-in Neo4j procedure that reads from the schema store rather than sampling real nodes. The upgrade alone reduced connect-phase page faults from ~56,000 to under 2,000.

## Results

| Metric | Before | After (config tuning) | After (connector 5.x) |
|--------|--------|-----------------------|------------------------|
| Calls per connect | 5 | 3 | 1 |
| Total page faults | ~56,000 | ~8,000 | < 2,000 |
| Connect phase duration | ~42 s | ~12 s | < 3 s |
| Result size | ~39 KB | ~12 KB | ~8 KB |

## The Lesson

`apoc.meta.nodeTypeProperties` does what it says — it returns property type information for nodes. The documentation is accurate. The function works. The problem was the assumption that "reading metadata" means "reading from a metadata store."

In a schema-free database, there is no metadata store. There are only nodes with properties. Reading metadata means reading nodes. Reading nodes means reading pages. Reading pages from every label in the database means scattering reads across the entire store.

If your Spark connector — or any client library — runs APOC introspection procedures on connect, you don't have a metadata query. You have a full-database sampling pass that runs before your first application query, costs more in page faults than your actual workload, and is invisible to every monitoring layer except the query log.

Check your connect phase. Check your query log for `apoc.meta.`. The most expensive query in your pipeline might be one you never wrote.

---

## References

- [APOC `apoc.meta.nodeTypeProperties` Documentation](https://neo4j.com/labs/apoc/5/overview/apoc.meta/apoc.meta.nodeTypeProperties/)
- [APOC `apoc.meta.relTypeProperties` Documentation](https://neo4j.com/labs/apoc/5/overview/apoc.meta/apoc.meta.relTypeProperties/)
- [Neo4j Spark Connector — Schema Mapping](https://neo4j.com/docs/spark/current/reading/schema/)
- [Neo4j `db.schema.nodeTypeProperties()` — Built-in Replacement](https://neo4j.com/docs/operations-manual/current/reference/procedures/#ref-procedure-db-schema-nodetypeproperties)
- [Neo4j Memory Configuration — Page Cache Sizing](https://neo4j.com/docs/operations-manual/current/performance/memory-configuration/)
