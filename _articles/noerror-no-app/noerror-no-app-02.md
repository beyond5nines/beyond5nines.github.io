---
layout: post
title: "NOERROR, No App — NODATA vs NXDOMAIN, and how to prove it to yourself"
short_title: "NODATA vs NXDOMAIN"
date: 2026-06-18 12:00:00 -0000
categories: aws route53 dns
series: "NOERROR, No App"
excerpt: "NODATA and NXDOMAIN are two codes apart but mean opposite things: one says gone, the other says empty."
redirect_from:
  - /part2-nodata-vs-nxdomain/
---

*Part 2 of the [NOERROR, No App](/noerror-no-app/) series. [Part 1](/noerror-no-app-01/) has the incident this comes from — but the failure mode below is general, and worth knowing whether or not you've hit it yet.*

An `A` record gets deleted. A DNS lookup against the name *succeeds* — no error, no timeout — and still hands back nothing usable. That gap between "the query worked" and "the app works" is easy to misread, expensive when you do, and only two response codes apart.

## NODATA vs NXDOMAIN — why the difference matters

These two look similar at a glance and mean opposite things:

- **NXDOMAIN**: the name does not exist. `dig` reports `status: NXDOMAIN`, `nslookup` says *can't find … NXDOMAIN*. The loud version, where every tool tells you the name is gone.
- **NODATA**: the name exists but holds nothing of the type you asked for. `dig` reports `status: NOERROR` with `ANSWER: 0`, `nslookup` says *No answer*. The query technically succeeded, which is what makes it deceptive.

The discriminator is the RCODE plus an empty answer, nothing else. In particular, the SOA in the authority section is not the tell: NXDOMAIN carries one too, since both kinds of answer get negatively cached. What the SOA does tell you is which zone replied — the parent (`example.com.`) means the queried name is a record inside it, whereas the name itself would mean the name is its own delegated zone.

A NODATA response commonly shows up this way:

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51334
;; flags: qr aa rd; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; AUTHORITY SECTION:
example.com.		300	IN	SOA	ns1.example.com. admin.example.com. 2026061243 7200 300 604800 900
```

The most common cause is exactly what it sounds like: the record type you're asking for is gone, but the name still owns *some* other record — a leftover TXT, an orphaned MX, anything — so the server can't say "no such name," only "not this type." A one-line `dig` sees `NOERROR` and moves on; so does a probe that checks only whether the zone replied. That's the trap.

> **An aside on that `900`.** It is the SOA's `MINIMUM` field, commonly misread as the negative-cache TTL outright. Per [RFC 2308](https://www.rfc-editor.org/rfc/rfc2308.html), the effective negative TTL is `min(SOA record TTL, MINIMUM)` — so if the record TTL is `300`, resolvers hold an empty answer for up to 300 seconds, not 900. Implementations vary and some cap negative caching by policy, so treat it as a ceiling, not a guarantee.

Two DNS flags are worth telling apart when you're reading any response: `aa` is set by the responder and means this server is authoritative for the zone. `rd` is set by the client and only records that recursion was requested — it says nothing about how the answer was obtained. `aa` with no `ra` beside it is a server answering from its own zone data rather than a resolver serving from cache.

## Operator checklist

If you're staring at an unreachable service right now, this is the order that resolves the ambiguity fastest.

```bash
# 1 — Which failure is it? NODATA and NXDOMAIN mean opposite things.
dig app.example.com               # status: NOERROR + ANSWER: 0  -> NODATA (name exists, type missing)
                                  # status: NXDOMAIN             -> the name itself is gone
nslookup app.example.com          # "No answer" vs "can't find ... NXDOMAIN"

# 2 — Rule out a cached answer and the near misses.
dig @<zone-nameserver> app.example.com    # aa set = straight from zone data, not a cache
dig app.example.com AAAA                  # are IPv6 clients quietly succeeding?
dig app.example.com ANY                   # which types exist at this name? (RFC 8482: some resolvers return HINFO instead of all records)

# 3 — Which account owns the zone? Repeat per profile; no tool surfaces an absent record across accounts.
aws route53 list-hosted-zones --profile <p> \
    --query "HostedZones[*].[Name,Id]" --output table

# 4 — List the records. Include AliasTarget or alias rows render as None.
aws route53 list-resource-record-sets --profile <p> \
    --hosted-zone-id "<ZONE_ID>" \
    --query "ResourceRecordSets[*].[Name, Type, ResourceRecords[0].Value || AliasTarget.DNSName]" \
    --output table

# 5 — Who changed it? Route 53 events surface through us-east-1.
aws cloudtrail lookup-events --profile <p> --region us-east-1 \
    --lookup-attributes AttributeKey=EventName,AttributeValue=ChangeResourceRecordSets \
    --max-results 10

# 6 — The detail: caller, source IP, console vs CLI vs Terraform, and the record itself.
aws cloudtrail lookup-events --profile <p> --region us-east-1 \
    --lookup-attributes AttributeKey=EventName,AttributeValue=ChangeResourceRecordSets \
    --max-results 10 --query 'Events[].CloudTrailEvent' --output text \
  | jq -r '[.eventTime, .userIdentity.arn, .sourceIPAddress, .userAgent,
            (.requestParameters.changeBatch.changes[] | .action + " " + .resourceRecordSet.name)] | @tsv'

# 6b — No Route 53 in this account? Widen to every mutating call.
aws cloudtrail lookup-events --profile <p> --region us-east-1 \
    --lookup-attributes AttributeKey=ReadOnly,AttributeValue=false \
    --max-results 50 --query 'Events[].{Time:EventTime,Event:EventName,User:Username}' \
    --output table

# 7 — After restoring, do not trust your own resolver straight away.
dig @<zone-nameserver> app.example.com    # authoritative: the truth now
dig app.example.com                       # your resolver: what users still see
# wait min(SOA record TTL, SOA MINIMUM) for the negative cache to age out before the all-clear
```

Two habits worth keeping. Only one `--lookup-attributes` is allowed per `cloudtrail lookup-events` call, so filter on the most selective attribute and post-process the rest. And check the `SERVER:` line in any `dig` output before you believe it — it is the difference between reading your zone and reading someone else's.

## References

- [RFC 2308 — Negative Caching of DNS Queries](https://www.rfc-editor.org/rfc/rfc2308.html) — defines NODATA (RCODE `NOERROR` with no answer records) and specifies the negative-cache TTL as `min(SOA record TTL, SOA MINIMUM)`, not the `MINIMUM` field alone.
- [Logging Amazon Route 53 API calls with AWS CloudTrail](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/logging-using-cloudtrail.html) — CloudTrail records every Route 53 API call, including who made it and from which IP. Three things are easy to conflate: Route 53 is a global service, not a regional one; its management events are recorded as global service events, typically surfaced through `us-east-1`, which is why the lookups here pin that Region; and what you can see depends on where you look — Event History is per account, per Region, 90 days, while a delivered trail depends on that trail's settings, including whether global service events are included.
- [Get notified when changes are made to Route 53](https://repost.aws/knowledge-center/route53-change-notifications) — the per-account EventBridge-on-`ChangeResourceRecordSets` pattern, the shape of the notification pipelines covered in [Part 3](/noerror-no-app-03/).

## Appendix: reproduce it yourself

Ten minutes to run, no root, nothing touching your machine's real DNS. The control case at the end is the part that convinces.

**Install CoreDNS and a current dig.** On macOS:

```bash
brew install coredns bind
```

(On Debian/Ubuntu: `apt-get install bind9-dnsutils`, and grab the CoreDNS binary from its GitHub releases page.)

**Write the zone.** The `app` name gets two records: the `A` we will delete, and a `TXT` that keeps the name alive afterwards. That TXT is the whole experiment.

```bash
mkdir -p ~/dns-lab && cd ~/dns-lab
cat > db.example.com <<'EOF'
$TTL 300
@   IN  SOA ns1.example.com. admin.example.com. (
        2026061243 7200 300 604800 900 )
@       IN  NS   ns1.example.com.
ns1     IN  A    192.0.2.53
www     IN  A    192.0.2.20
api     IN  A    192.0.2.30
app     IN  A    192.0.2.10
app     IN  TXT  "owner=platform-team"
EOF
cat > Corefile <<'EOF'
example.com:5300 {
    file db.example.com
    log
    errors
}
EOF
```

**Start it** in one terminal tab and leave it running. Port 5300 keeps you clear of the system resolver on 53, which is why no `sudo` is needed:

```bash
coredns -conf Corefile
```

**Baseline.** In a second tab, confirm the record resolves. You want `status: NOERROR` with `ANSWER: 1`:

```bash
dig @127.0.0.1 -p 5300 app.example.com
```

**Break it.** Delete the `app IN A` line, keep the `app IN TXT` line, and bump the serial. Then restart CoreDNS (Ctrl-C and re-run) so the zone reloads:

```bash
sed -i '' '/^app     IN  A/d' db.example.com     # on Linux: sed -i '/^app     IN  A/d'
sed -i '' 's/2026061243/2026061244/' db.example.com
```

**The NODATA capture.** Same query, and now you get the incident:

```bash
dig @127.0.0.1 -p 5300 app.example.com
nslookup -port=5300 app.example.com 127.0.0.1
```

`status: NOERROR`, `ANSWER: 0`, an SOA in the authority section, and `No answer` from `nslookup`.

The `nslookup` line is the less portable of the two. The `-port=` form works with the BIND build above and on Windows, but busybox and some distro-packaged variants spell it differently or drop it. `dig` is the reliable one, and prints the RCODE plainly. If your `nslookup` rejects the flag, skip it — the `dig` output is the evidence.

**The control case.** Delete the TXT as well, so the name holds no records at all, bump the serial again, reload, and re-run the same query. It flips to `NXDOMAIN` — the loud, honest failure everyone would prefer to get, and the one-line difference between 75 minutes spent on a cross-account record hunt and a diagnosis in the first minute.

Two things trip people up. Bump the SOA serial on every edit or the reload is a no-op. And keep `@127.0.0.1 -p 5300` on every query: drop it and you are asking the public internet about the real `example.com`, which will answer NODATA for its own reasons and give you a false positive. Check the `SERVER:` line to be sure you queried the lab.

*Next: [Part 3 — Why per-account pipelines don't scale](/noerror-no-app-03/), or back to [Part 1 — The Incident](/noerror-no-app-01/).*
