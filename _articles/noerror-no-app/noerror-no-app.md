---
layout: post
title: "NOERROR, No App"
title_display: 'NOERROR, No <span class="letter-flicker">A</span>pp'
date: 2026-06-16 12:00:00 -0000
categories: aws route53 cloudtrail
series: "NOERROR, No App"
pinned: true
excerpt: "A deleted Route 53 record took the app down for 75 minutes — this series on why cross-account audit needs one home, not N."
redirect_from:
  - /noerror-no-app-intro/
---

An engineer on our team deleted a Route 53 record by accident during a routine cleanup, and an app went down. Restoring the record was a manual change in the Route 53 console — a minute of work. Working out *which* of our AWS accounts owned it took ~45 minutes of the outage, and attributing the change to a specific person took another ~30 minutes after the app was back — 75 minutes in total, almost none of it spent fixing anything, because in a multi-account estate the answer to "what changed, where, and by whom" is scattered across N accounts with no single place to ask.

That hour, not the deletion, is what this series is about. It comes in four parts:

- **[Part 1 — The Incident](/noerror-no-app-01/)** walks the outage itself: why a healthy-looking app tier and a DNS lookup that *succeeded* sent us hunting the wrong layer, the cross-account hunt, and the four separate problems the incident actually was.
- **[Part 2 — NODATA vs NXDOMAIN, and how to prove it to yourself](/noerror-no-app-02/)** is the reference piece: the protocol detail behind the deceptive `NOERROR`, an operator checklist, and a ten-minute reproducible lab.
- **[Part 3 — Why per-account pipelines don't scale](/noerror-no-app-03/)** covers what we built right after the incident, why it stopped working past ten accounts, and the candidates we considered instead.
- **[Part 4 — The Fix](/noerror-no-app-04/)** builds the thing we wished we'd had: one place that holds every account's control-plane history, queryable in seconds, and the cost math behind it.

**[Part 1 — The Incident](/noerror-no-app-01/)** is where the series starts.
