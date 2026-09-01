---
layout: post
title: "NOERROR, No App — The Incident"
short_title: "The Incident"
date: 2026-06-17 12:00:00 -0000
categories: aws route53 cloudtrail
series: "NOERROR, No App"
excerpt: "A deleted Route 53 A record took the app down while DNS returned NOERROR. Finding the owning account cost 75 minutes."
redirect_from:
  - /part1-the-incident/
---

*Part 1 of the [NOERROR, No App](/noerror-no-app/) series. Start there for the background and the fix menu.*

| Time | What happened |
|---|---|
| T+0 | Monitoring probe goes red; tickets start arriving |
| T+5 | App tier ruled out — tasks healthy, logs quiet, no user requests |
| T+10 | ALB, WAF and TLS ruled out; failure sits before the TCP connection |
| T+12 | `dig` returns `NOERROR` with `ANSWER: 0` — the `A` record is gone, the zone is fine |
| T+15 | Hunt begins for which of three accounts owns the zone |
| T+40 | Found in the legacy account; record manually recreated in the Route 53 console |
| T+45 | App serving again, once the negative cache aged out |
| T+75 | Attribution complete — the deleting call located in that account's trail |

## The alert, and ruling out the app

It started with a Slack alert: Our monitoring probe had gone red. Tickets flooded in seconds later, so this was total and it was hitting real users. No 500s, no latency spike, nothing serving. The first instinct is always the app tier, so that is where we looked. It was all healthy.

The ECS service was healthy: Fargate tasks running, not cycling, desired count matching running. Host metrics flat, no throttling, no OOM kills. We tried logging into the app ourselves and it wouldn't load. The logs turned the investigation. They were clean, but not reassuringly so: only internal health-check pings against the container's IP, no user requests at all.

So we worked outward. The ALB, WAF and TLS were all fine.

That narrowed it. Tasks up, platform up, routing intact, app still unreachable. If nothing between the user and the container was broken, the failure sat earlier than all of it — before a TCP connection is even opened. Name resolution. Does `app.example.com` resolve to an address?

*Output below is reproduced from the lab in Part 2, zone values simplified; the real query went to the Route 53 nameservers.*

```
$ nslookup -port=5300 app.example.com 127.0.0.1
Server:		127.0.0.1
Address:	127.0.0.1#5300

*** Can't find app.example.com: No answer
```

"No answer" is not "no such host." The lookup succeeded and the name was still in the zone; there was just no address record to hand back. That is a different failure from the name being gone. To rule out a stale positive answer cached between us and the source, we asked the authoritative server directly:

```
$ dig @127.0.0.1 -p 5300 app.example.com

; <<>> DiG 9.20.24 <<>> @127.0.0.1 -p 5300 app.example.com
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51334
;; flags: qr aa rd; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; QUESTION SECTION:
;app.example.com.		IN	A

;; AUTHORITY SECTION:
example.com.		300	IN	SOA	ns1.example.com. admin.example.com. 2026061243 7200 300 604800 900

;; SERVER: 127.0.0.1#5300(127.0.0.1) (UDP)
```

Status `NOERROR` with `ANSWER: 0`: the query succeeded and returned nothing. The authority section carries an SOA owned by `example.com.` — the parent zone — so `app` was an ordinary record inside a zone that was alive and answering. There was just no `A` record for it. The `aa` flag (set by the responder) with no `ra` beside it told us this was straight from zone data, not a cache serving something stale.

We ruled out the near misses while we were there. An `AAAA` query came back the same way, `NOERROR` with an empty answer, so IPv6 clients weren't quietly succeeding. And `app` was a plain address record, not a `CNAME` or alias pointing at some other target that had failed on its own. One record type, deleted.

One thing made it "No answer" rather than "no such name": a leftover TXT record beside the deleted `A` meant the name still existed, so the server could only say "not this type," not "no such name" — the reason we got the quieter `NOERROR` instead of the loud, honest `NXDOMAIN`. Nothing watching the endpoint distinguished that empty, successful response from a healthy one, so "the endpoint is down" pointed straight at the app. *(Full protocol detail and a reproducible lab: [Part 2](/noerror-no-app-02/).)*

The question changed: find the record, find who deleted it, put it back.

## The blind moment

We knew *what* — an A record was deleted. We had no fast way to answer *where* or *who*.

**Where.** Our DNS wasn't one tidy zone. It was several hosted zones spread across three standalone accounts, with no map of which account owned which. Before hunting for the missing record we had to work out which account held `example.com` at all. The console is the wrong tool for this — you can't eyeball an absence, and there's no search across accounts and no diff against yesterday — so we went to the CLI and asked each account in turn. First, which zones live here:

```
# run per account / profile
$ aws route53 list-hosted-zones --query "HostedZones[*].[Name,Id]" --output table
```

Then dump the records for the `example.com` zone and look for `app`:

```
$ aws route53 list-resource-record-sets \
    --hosted-zone-id "YOUR_HOSTED_ZONE_ID" \
    --query "ResourceRecordSets[*].[Name, Type, ResourceRecords[0].Value]" \
    --output table
```

Read that output before trusting it. A handful of rows come back with a value of `None`. Those are not broken records — they are Route 53 **alias** records, whose target lives in an `AliasTarget` field that this query never reads. A row rendering as `None` looks a lot like a record that lost its value, which is a fast way to talk yourself into the wrong theory. So run it again with the alias path folded in, and the `None` rows resolve into real targets:

```
$ aws route53 list-resource-record-sets \
    --hosted-zone-id "YOUR_HOSTED_ZONE_ID" \
    --query "ResourceRecordSets[*].[Name, Type, ResourceRecords[0].Value || AliasTarget.DNSName]" \
    --output table
```

> **What an alias record is.** An alias is a Route 53 extension, not a standard DNS record type. It points a name at an AWS resource — an ALB, a CloudFront distribution, an S3 website endpoint — and Route 53 resolves the target on the authoritative side, returning address records to the client. It shows up everywhere for two reasons: DNS forbids a `CNAME` at a zone apex, so an alias is the only way to point `example.com` itself at a load balancer, and Route 53 does not charge for queries to alias records that map to supported AWS resources, while ordinary queries are billed. The catch for anyone auditing records is that the target sits in `AliasTarget`, not `ResourceRecords` — so a query reaching only for `ResourceRecords[0].Value` prints a null.

Now the listing is honest, and `app` is unambiguous. Run it per account — a separate login each time, its own credentials, its own copy of the answer — and in the legacy account `app` still turned up, but only as a `TXT` row. The `A` record was gone. Not an alias hiding its value, not misconfigured, not pointing somewhere wrong — deleted. Sixty-plus records, three accounts: Route 53 and Resource Explorer can index existing records, but neither surfaces an absent one. Finding something that *isn't* there — across separate logins with no shared view — is a manual process, and this was one. The CLI just made the clicking faster; there was still no single place to ask "does this record exist, in any of our accounts" — effectively the same hand-search a console-only hunt across three separate logins would have taken, just typed instead of clicked. We recreated it manually in the Route 53 console.

The app did not come back instantly. Resolvers that had asked during the outage had cached the empty answer, and caches do not care that the record exists again — for up to a few more minutes they kept serving it from memory. Our first test looked like the fix had failed; it hadn't, we were talking to a resolver that had not aged out. Worth knowing before you declare a DNS fix broken — or declare it fixed to users still holding a cached miss. The diagnostic tools work — they just answer one account at a time.

Notice what the hunt actually was: the same handful of commands, once per account. Three accounts today, and it doesn't shrink as the fleet grows.

**Who.** Not from the Route 53 console, and not from the commands above: the record is gone, so there is nothing left to inspect. Attribution is a CloudTrail question, and the tooling answers it well once you're in the right account — but there is no audit path that spans accounts, only three separate, per-account ones, and finding the right one to query is the same blind hunt as above. Event history surfaces the call from the CLI too:

```
$ aws cloudtrail lookup-events \
    --region us-east-1 \
    --lookup-attributes AttributeKey=EventName,AttributeValue=ChangeResourceRecordSets \
    --max-results 10
```

That gives you the event and the caller, but the part you want is buried in `CloudTrailEvent`, a JSON string on each result. Unpack it and the record falls out:

```
$ aws cloudtrail lookup-events --region us-east-1 \
    --lookup-attributes AttributeKey=EventName,AttributeValue=ChangeResourceRecordSets \
    --max-results 10 --query 'Events[].CloudTrailEvent' --output text \
  | jq -r '[.eventTime, .userIdentity.arn, .sourceIPAddress, .userAgent,
            (.requestParameters.changeBatch.changes[] | .action + " " + .resourceRecordSet.name)] | @tsv'
```

Now you have the timestamp, the calling ARN, the source IP, and — via `userAgent` — whether it came from the console, a CLI, or Terraform, alongside the literal `DELETE app.example.com.` from the change batch.

Note the walls. `lookup-events` is scoped to the account and Region you query, and Event History reaches back 90 days. The tooling could always tell us — but only once we already knew which account to ask.

## We found who — eventually

Once the app was back, that `lookup-events` call — run against the legacy account, the one that had owned the zone — returned the `ChangeResourceRecordSets` delete. The `userIdentity` on it, a role and session, traced back to one of us. An SRE running a cleanup had deleted the record by accident.

Be precise about what the trail proved. A compromised role looks identical in CloudTrail to a legitimate one, so the event alone rules nothing out. What ruled out compromise was that the session traced to a known engineer on a known task at that hour, and they confirmed it. Attribution starts that conversation; it doesn't end it. No breach, no rogue actor — a normal mistake on a busy day, the kind every team makes.

Which is exactly the point. Strip the incident to its parts and it was four separate problems, only one of which was about fixing anything:

- **Detection** flagged that something was down, but not that it was DNS. The probe reddened on the endpoint, and the `NOERROR` lookup made DNS look healthy.
- **Ownership discovery**: working out which of three accounts even held the zone, with no inventory to consult.
- **Attribution**: finding who changed it — one `lookup-events` call, but only against the account you already knew to ask.
- **Recovery**, the actual fix, was a manual record creation in the Route 53 console — trivial once the first three were done.

Three of those four were pure blindness, and they owned the clock. Of the ~45 minutes from page to app-restored, about 12 went to ruling out the app tier and the network path (T+0 to T+12), then roughly 25 to the cross-account hunt for the zone (T+15 to T+40) — the fix itself, once found, took minutes. Working out who came later, with the app already back: another ~30 minutes (T+45 to T+75), almost all of it spent getting into the right account before a single `lookup-events` call answered the question in seconds. Investigation time rather than downtime, but time all the same. Neither task was hard. Both were blind. When the cause is an ordinary accident rather than an attacker, that blindness is the whole cost.

Visibility is only half the answer. It tells you who and where after the fact, not "don't." The complementary guardrail is IAM: scope permissions so that few principals can call `ChangeResourceRecordSets` on load-bearing zones, and require a second set of eyes on those who can. A tightly scoped resource policy won't stop every mistake, but it raises the bar from "any engineer with console access" to "one of two people, both of whom know what they're touching."

So why couldn't we? Because nothing gave us a single view across accounts. The fix we reached for first — a notification pipeline in every account — turned out to answer the wrong question. That story, and the alternatives we weighed, is in [Part 3 — Why per-account pipelines don't scale](/noerror-no-app-03/). The full protocol breakdown of NODATA vs NXDOMAIN, an operator checklist, and a ten-minute reproducible lab are in [Part 2](/noerror-no-app-02/). The thing we settled on gets built in [Part 4 — The Fix](/noerror-no-app-04/).
