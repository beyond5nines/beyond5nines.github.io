---
layout: post
title: "Look Ma, No Servers!"
date: 2026-04-08 12:00:00 -0000
categories: aws glue serverless
series: Look Ma! no servers
pinned: true
redirect_from:
  - /2026/04/08/look-ma-no-servers/
---

## The Series

**"Serverless moves the servers out of sight, not out of your problem."**

This is a series about the hidden constraints of AWS Glue and other "serverless" services — the ones that don't show up in the console, aren't mentioned in the documentation, and only reveal themselves when something breaks at 7AM.

Each post in this series follows the same pattern: a production failure that seemed impossible, the investigation that uncovered the root cause, and the observability we built to make sure it never happens again.

---

## Posts in This Series

### Part 1: [Subnet IP Exhaustion — Investigation](/look-ma-no-servers-01/)
**The problem:** Glue jobs failing with "The specified subnet does not have enough free IP addresses" — even though our math said we had capacity.

**The root cause:** AWS Glue's ENI lifecycle extends far beyond job completion. ENIs remain attached for 60+ minutes after jobs finish, silently consuming IPs. Every job category shared one /26. The launcher was blind.

---

### Part 2: [Subnet IP Exhaustion — Fix](/look-ma-no-servers-02/)
**The fix:** Blast radius isolation via subnet splitting, plus an adaptive Airflow sensor with a rolling-max threshold backed by DynamoDB that queries actual ENI state before launching.

**The results:** Zero IP exhaustion related failures.

---

### Part 3: [No Space Left on Device — Trap](/look-ma-no-servers-03/)
**The problem:** Glue jobs failing after 48 minutes with "No space left on device" — on workers that should have had plenty of disk.

**The root cause:** Spark shuffle spill consumes local disk faster than expected when partition counts are wrong and compression is ineffective.

**The fix:** Right-sizing partitions, switching to ZSTD compression, and building disk utilization observability that AWS doesn't provide.

---

## The Pattern

Every post in this series follows the same shape:

1. **The failure** — what broke, when, and how it presented
2. **The investigation** — the queries, the metrics, the things we had to build to see what was happening
3. **The root cause** — the undocumented behavior or hidden constraint
4. **The fix** — architectural changes and observability improvements
5. **The results** — before/after metrics proving the fix worked

If you've ever stared at a Glue failure wondering why AWS won't just *tell you* what's wrong, this series is for you.

---


*Follow along as we document the operational realities of running Glue at scale — the gaps AWS doesn't fill, and the signals you have to build yourself.*
