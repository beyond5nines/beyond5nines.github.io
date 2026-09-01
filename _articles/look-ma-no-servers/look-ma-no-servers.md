---
layout: post
title: "Look Ma, No Servers!"
title_display: 'Look <span class="letter-flicker">M</span>a, No Servers!'
date: 2026-04-08 12:00:00 -0000
categories: aws glue serverless
series: Look Ma! no servers
pinned: true
excerpt: "Serverless moved the servers out of sight, not out of our problem — a series on the failures AWS Glue's abstraction hid from us."
redirect_from:
  - /2026/04/08/look-ma-no-servers/
---

Remember yelling "Look ma, no hands!" right before eating pavement? That's our AWS Glue story.

We adopted serverless expecting less infrastructure work. Instead, we spent months debugging Linux OOM killers, YARN memory limits, and orphaned ENIs — without visibility into actual servers.

The pitch was clear: no patching, no capacity planning, no servers to babysit. Hand the undifferentiated heavy lifting to AWS and focus on the data. We bought it.

What we got was the same constraints with none of the visibility. The servers are still there — AWS just hides them behind a managed abstraction. ENIs still expire on an undocumented schedule. Disk still runs out in ways the console won't show you. Memory still gets killed at limits that don't appear in the documentation.

**Serverless moves the servers out of sight, not out of your problem.**

This series documents what we found: the undocumented behaviors, the production failures they caused, and the observability we had to build ourselves to stop being surprised by a service that holds all the signals and shares almost none of them.

---

## In This Series

### Part 1: [Subnet IP Exhaustion — Investigation](/look-ma-no-servers-01/)

Our daily batch jobs started failing at 7:03 AM with "The specified subnet does not have enough free IP addresses." Our math showed we had capacity. The subnet disagreed.

The root cause: AWS Glue's ENI lifecycle extends 60+ minutes past job completion — undocumented, invisible in the console, and compounding with every run. Every job category shared one /26. The launcher was blind to actual subnet state.

---

### Part 2: [Subnet IP Exhaustion — Fix](/look-ma-no-servers-02/)

The fix wasn't more IPs. It was feedback.

Blast radius isolation via subnet splitting — critical batch, non-critical batch, and ad-hoc jobs each trapped in their own /26. Then an adaptive Airflow sensor backed by DynamoDB that reads actual ENI state before each launch, with a rolling-max buffer that learns from observed cleanup lag rather than assuming a static window.

**Result:** Zero IP exhaustion failures in 90 days. 12 sensor-blocked launches that self-recovered within 10 minutes, no manual intervention.

---

### Part 3: [No Space Left on Device — Trap](/look-ma-no-servers-03/)

A G.2X job ran healthy for 48 minutes and 59 seconds, then died: `No space left on device`. On workers with 128 GB of local disk each. The Glue console showed nothing — no disk utilization metric, no spill volume, no warning before the wall.

S3 Shuffle stopped the bleeding. But it also handed us something AWS never did: the shuffle artifacts themselves, sitting in S3, readable. The files exposed what was actually wrong — 46% empty partitions from an unconfigured default, and a compression codec doing zero work on UUID data where LZ4 can find no repeated patterns. The failure wasn't about disk size. It was about what was being written to it.

---

### Part 4: [No Space Left on Device — Fix](/look-ma-no-servers-04/)

Post 03 identified the trap: 46% empty shuffle partitions from an unconfigured default and a compression codec doing zero work on UUID data. This post applies the fix — five Spark configuration changes that cut shuffle bytes written to S3 by 50%, from 267.9 GB to 133.0 GB per run. No infrastructure changes, no scaling. Just defaults that were always wrong for the workload.

---

*Every managed service has an observability gap. Finding it is part of operating the service.*
