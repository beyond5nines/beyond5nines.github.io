---
layout: post
title: "NOERROR, No App — seeing across AWS accounts"
date: 2026-06-16 12:00:00 -0000
categories: aws route53 cloudtrail
series: "NOERROR, No App"
pinned: true
---

An engineer on our team deleted a Route 53 record by accident during a routine cleanup, and an app went down. Restoring the record was a manual change in the Route 53 console — a minute of work. Working out *which* of our AWS accounts owned it took ~45 minutes of the outage, and attributing the change to a specific person took another ~30 minutes after the app was back — 75 minutes in total, almost none of it spent fixing anything, because in a multi-account estate the answer to "what changed, where, and by whom" is scattered across N accounts with no single place to ask.

That hour, not the deletion, is what this series is about. It comes in four parts:

- **[Part 1 — The Incident](/part1-the-incident/)** walks the outage itself: why a healthy-looking app tier and a DNS lookup that *succeeded* sent us hunting the wrong layer, the cross-account hunt, and the four separate problems the incident actually was.
- **[Part 2 — NODATA vs NXDOMAIN, and how to prove it to yourself](/part2-nodata-vs-nxdomain/)** is the reference piece: the protocol detail behind the deceptive `NOERROR`, an operator checklist, and a ten-minute reproducible lab.
- **[Part 3 — Why per-account pipelines don't scale](/part3-why-per-account-pipelines-fail/)** covers what we built right after the incident, why it stopped working past ten accounts, and the candidates we considered instead.
- **[Part 4 — The Fix](/part4-the-fix/)** builds the thing we wished we'd had: one place that holds every account's control-plane history, queryable in seconds, and the cost math behind it.

**[Part 1 — The Incident](/part1-the-incident/)** is where the series starts.
