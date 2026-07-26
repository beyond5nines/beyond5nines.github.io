---
layout: post
title: "Can't find app: No answer — Investigation"
date: 2026-06-17 12:00:00 -0000
categories: aws route53 cloudtrail athena
series: "NOERROR, No App"
---

> **The short version.** An `A` record for our app was deleted from a Route 53 zone. A stale TXT record still sat on the same name, so DNS answered authoritatively but without a usable address: `NOERROR` with an empty answer (commonly called NODATA) rather than `NXDOMAIN`. Lookups did fail functionally, so this was not invisible to machines. The gap was in triage, where a non-error RCODE reads as non-actionable. We spent the first stretch of the outage on the app tier. Restoring the record took one `terraform apply`. The hour went to not knowing which of three AWS accounts owned the zone, with no practical centralized way to ask all of them at once.

| Time | What happened |
|---|---|
| T+0 | Monitoring probe goes red; tickets start arriving |
| T+5 | App tier ruled out — tasks healthy, logs quiet, no user requests |
| T+10 | ALB, WAF and TLS ruled out; failure sits before the TCP connection |
| T+12 | `dig` returns `NOERROR` with `ANSWER: 0` — the `A` record is gone, the zone is fine |
| T+15 | Hunt begins for which of three accounts owns the zone |
| T+40 | Found in the legacy account; restored with `terraform apply` |
| T+45 | App serving again, once the negative cache aged out |
| T+75 | Attribution complete — the deleting call located in that account's trail |

### The alert, and ruling out the app

It started with a Slack alert: Our monitoring probe had gone red. Tickets flooded in seconds later, so this was total and it was hitting real users. No 500s, no latency spike, nothing serving. The first instinct is always the app tier, so that is where we looked. Everything was green.

The ECS service was healthy: Fargate tasks running, not cycling, desired count matching running. Host metrics flat, no throttling, no OOM kills. We tried logging into the app ourselves and it wouldn't load. The logs turned the investigation. They were clean, but not reassuringly so: only internal health-check pings against the container's IP, no user requests at all. 

So we worked outward. The ALB target group still showed the tasks healthy and its listener rules were unchanged. . TLS still terminated with a valid certificate.

That narrowed it. Tasks up, platform up, routing intact, app still unreachable. If nothing between the user and the container was broken, the failure sat earlier than all of it — before a TCP connection is even opened. Name resolution. Does `app.example.com` resolve to an address?

```
$ nslookup -port=5300 app.example.com 127.0.0.1
Server:		127.0.0.1
Address:	127.0.0.1#5300

*** Can't find app.example.com: No answer
```

"No answer" is not "no such host." The lookup succeeded and the name was still in the zone; there was just no address record to hand back. That is a different failure from the name being gone. To rule out a stale positive answer cached between us and the source, we asked the authoritative server directly, without `+noall` so the status line shows. (These blocks come from the reproduction at the end of the post, which is why the server is a local CoreDNS; the answer has the same shape either way.)

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

Read what's there and what isn't. Status `NOERROR` with `ANSWER: 0`: the query succeeded and returned nothing. The authority section carries an SOA owned by `example.com.` — the parent zone — so `app` was an ordinary record inside a zone that was alive and answering. There was just no `A` record for it.

Two flags are worth telling apart. `aa` is set by the responder: this server is authoritative for the zone. `rd` is set by us and only records that we asked for recursion, saying nothing about how the answer was obtained. `aa` with no `ra` beside it, as here, is a server answering from its own zone data rather than a resolver answering from cache.

We ruled out the near misses while we were there. An `AAAA` query came back the same way, `NOERROR` with an empty answer, so IPv6 clients weren't quietly succeeding. And `app` was a plain address record, not a `CNAME` or alias pointing at some other target that had failed on its own. One record type, deleted.

One thing made it "No answer" rather than "no such name": `app.example.com` still had a TXT record beside the deleted `A`, a leftover domain-verification string. Because the name still owned a record of some type, the server couldn't say "no such name" — only that the name is here, just not as the type you asked for. That surviving TXT is why we got a successful-looking `NOERROR` instead of the `NXDOMAIN` that would have made this obvious in minute one. Scope matters: this is the behaviour for an existing owner name inside the same authoritative zone, and delegation, wildcards and `CNAME` chains each answer differently. It was also a verification record nobody had pruned long after it stopped mattering — DNS hygiene audits should remove dead records, not only guard live ones.

> **An aside on that `900`.** It is the SOA's `MINIMUM` field, commonly misread as the negative-cache TTL outright. Per RFC 2308 the effective negative TTL is `min(SOA record TTL, MINIMUM)`, and the record TTL here is the `300` on the left — so resolvers hold this empty answer for up to 300 seconds in this setup, not 900. Implementations vary and some cap negative caching by policy, so treat it as a ceiling. It comes back when we restore the record.

### NODATA vs NXDOMAIN — why the difference mattered

These two look similar at a glance and mean opposite things:

- **NXDOMAIN**: the name does not exist. `dig` reports `status: NXDOMAIN`, `nslookup` says *can't find … NXDOMAIN*. The loud version, where every tool tells you the name is gone.
- **NODATA**: the name exists but holds nothing of the type you asked for. `dig` reports `status: NOERROR` with `ANSWER: 0`, `nslookup` says *No answer*. The query technically succeeded, which is what makes it deceptive.

The discriminator is the RCODE plus an empty answer, nothing else. In particular the SOA in the authority section is not the tell: NXDOMAIN carries one too, since both kinds of answer get negatively cached. What the SOA does tell you is which zone replied — the parent (`example.com.`) means the queried name is a record inside it, whereas the name itself would mean the name is its own delegated zone. Ours was the parent.

That is the trap, and where this series gets its name. DNS answered authoritatively, just not with a usable address, and nothing watching it distinguished an empty successful response from a healthy positive one. A one-line `dig` sees `NOERROR` and moves on; so does a probe that checks only whether the zone replied. Our probe had gone red, but all it knew was that the endpoint was unreachable — and "the endpoint is down" points you at the app. NOERROR, and yet no app. The question changed: find the record, find who deleted it, put it back.

### The blind moment

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

Read that output before trusting it. A handful of rows come back with a value of `None`. Those are not broken records — they are Route 53 **alias** records, whose target lives in an `AliasTarget` field that this query never reads (see the aside below). A row rendering as `None` looks a lot like a record that lost its value, which is a fast way to talk yourself into the wrong theory. So run it again with the alias path folded in, and the `None` rows resolve into real targets:

```
$ aws route53 list-resource-record-sets \
    --hosted-zone-id "YOUR_HOSTED_ZONE_ID" \
    --query "ResourceRecordSets[*].[Name, Type, ResourceRecords[0].Value || AliasTarget.DNSName]" \
    --output table
```

> **What an alias record is.** An alias is a Route 53 extension, not a standard DNS record type. It points a name at an AWS resource — an ALB, a CloudFront distribution, an S3 website endpoint — and Route 53 resolves the target on the authoritative side, returning address records to the client. It shows up everywhere for two reasons: DNS forbids a `CNAME` at a zone apex, so an alias is the only way to point `example.com` itself at a load balancer, and Route 53 does not charge for queries to alias records that map to supported AWS resources, while ordinary queries are billed. The catch for anyone auditing records is that the target sits in `AliasTarget`, not `ResourceRecords` — so a query reaching only for `ResourceRecords[0].Value` prints a null.

Now the listing is honest, and `app` is unambiguous. Run it per account — a separate login each time, its own credentials, its own copy of the answer — and in the legacy account `app` still turned up, but only as a `TXT` row. The `A` record was gone. Not an alias hiding its value, not misconfigured, not pointing somewhere wrong — deleted. Sixty-plus records, three accounts, no cross-account search: finding a record that *isn't* there is not what these tools are built for. We recreated it by re-running `terraform apply` from the record's config in version control.

The app did not come back instantly, and this is where that 300-second negative TTL earns its place. Resolvers that had asked during the outage had cached the empty answer, and caches do not care that the record exists again. For up to five more minutes they kept serving it from memory. Our first test looked like the fix had failed; it hadn't, we were talking to a resolver that had not aged out. Worth knowing before you declare a DNS fix broken — or declare it fixed to users still holding a cached miss.

Notice what the hunt actually was: the same handful of commands in a loop, once per account, each a separate login with its own credentials and its own copy of the answer. Three accounts today, and it doesn't shrink as the fleet grows.

**Who.** Not from the Route 53 console, and not from the commands above: the record is gone, so there is nothing left to inspect. Attribution is a CloudTrail question, and the tooling answers it well. Event history surfaces the call from the CLI too:

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

### We found who — eventually

Once the app was back, that `lookup-events` call — run against the legacy account, the one that had owned the zone — returned the `ChangeResourceRecordSets` delete. The `userIdentity` on it, a role and session, traced back to one of us. An SRE running a cleanup had deleted the record by accident.

Be precise about what the trail proved. A compromised role looks identical in CloudTrail to a legitimate one, so the event alone rules nothing out. What ruled out compromise was that the session traced to a known engineer on a known task at that hour, and they confirmed it. Attribution starts that conversation; it doesn't end it. No breach, no rogue actor — a normal mistake on a busy day, the kind every team makes.

Which is exactly the point. Strip the incident to its parts and it was four separate problems, only one of which was about fixing anything:

- **Detection** flagged that something was down, but not that it was DNS. The probe reddened on the endpoint, and the `NOERROR` lookup made DNS look healthy.
- **Ownership discovery**: working out which of three accounts even held the zone, with no inventory to consult.
- **Attribution**: finding who changed it — one `lookup-events` call, but only against the account you already knew to ask.
- **Recovery**, the actual fix, was a single `terraform apply`, trivial once the first three were done.

Three of those four were pure blindness, and they owned the clock. Locating the zone and the record ate most of the outage: ~45 minutes from page to app-restored, almost all of it spent finding where to look. Working out who came later, with the app already back — another ~30 minutes, most of it spent getting into the right account before a single `lookup-events` call answered it. Investigation time rather than downtime, but time all the same. Neither task was hard. Both were blind. When the cause is an ordinary accident rather than an attacker, that blindness is the whole cost.

Visibility is only half the answer, though. It tells you who and where after the fact, not "don't." These records were already Terraform-managed, so the cheapest guardrail was sitting right there: `lifecycle { prevent_destroy = true }` on the load-bearing ones, which makes `terraform apply` fail rather than delete and forces anyone who means it to remove the guard in a reviewed commit first.

Be clear about its reach. It stops Terraform destroying the resource and nothing else. A console click or a direct `ChangeResourceRecordSets` call sails straight past it, which is exactly how this record died. Covering that needs IAM scoped so few principals can delete records in the zone at all, plus review on the ones who can.

So why couldn't we?

### What we tried first — and the root cause we missed

We weren't blind by choice. We'd built change notifications after an earlier bad change — per account:

```
each account:  CloudTrail → Lambda → Slack   (+ DynamoDB change log)
```

Each account had its own trail, its own Lambda parsing events into Slack, and a DynamoDB table of changes. At two or three accounts it worked.

The failure wasn't any one of those pieces. It was the shape: we had replicated a stateful pipeline N times over. Every account carried its own copy of the code, the secrets and the runtime, and each copy aged on its own. Past ten accounts that meant one Slack webhook to rotate in ten places, ten Lambda runtimes drifting apart, the same CVE patched and redeployed ten times, and a fresh pipeline for every account we added. The drift was worst where ownership was weakest: the legacy account had barely been reporting by the time it mattered.

A pipeline meant to give us visibility had become the biggest single consumer of on-call time, and it was blindest in the account that took us down. The lesson wasn't that the idea was wrong. It was that audit is a read-side concern, and read-side concerns belong in one place instead of being copied into every account that emits events. The answer isn't a better cleanup script or a tenth pipeline. It is one place to stand where every account is visible at once.

### The shape of the fix, and what we ruled out

If answers live in N accounts and you can only ask one at a time, the fix is a single place holding all of them. Four candidates were on the table, and why three lost is almost entirely about *what each one bills for* rather than what each can do.

Price them against one scenario: ten accounts, roughly 5 GB a month of CloudTrail management events between them, about 10,000 configuration items per account per month, twelve months retention, and forensic use — ten queries a month, each pruned to a couple of partitions.

| Option | Billed on | Illustrative monthly cost at this scenario | Why it lost |
|---|---|---|---|
| **CloudTrail → S3 + Athena** | retention, plus bytes scanned per query | **≈ $1.40/mo.** First copy of management events is free. 60 GB accumulated × $0.023/GB = $1.38. Ten queries scanning ~100 MB each = 1 GB × $5/TB ≈ $0.005 | *Won.* Cost tracks how long you keep data and how carefully you query, not how busy the estate is |
| **AWS Config** | per configuration item recorded | **≈ $300/mo.** 10 accounts × 10,000 items × $0.003 = $300, recording alone, before any rule evaluations | Bills on how much the estate *changes*, whether or not anyone ever asks. A resource stuck in a create/delete loop bills for every cycle. Route 53 record-level coverage is also thinner than CloudTrail's plain API capture |
| **Elasticsearch + Kibana** | cluster hours, 24/7 | **≈ $280+/mo**, illustrative — three small nodes for a minimal HA cluster, before EBS and snapshots. Runs whether or not you query | Earns its floor only when a cluster is busy, and ours would idle between incidents. Hands back the shard, ILM, upgrade and CVE maintenance we were trying to delete |
| **CloudTrail Lake** | per GB ingested, plus retention | **Moot.** AWS [closed it to new customers on 31 May 2026](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-lake-service-availability-change.html) | Off the menu for anything you build now. We would have passed anyway: a standing charge on all ingested volume, queried or not |

Pricing changes, and those figures are illustrative: the useful comparison here is billing shape, not the exact dollar amount. Substitute your own numbers before trusting the ratio, since the gap moves with how much your estate changes versus how much you query. But the shape is the point. Three of the four charge continuously for data arriving. The fourth charges for keeping it and for the questions you actually ask, which in forensic use is a handful a month.

That leaves the boring one.

### What's next

Three of those four gaps, detection aside, were the same failure in different clothes: an answer that existed in some account, with no way to ask for all at once. In **Part 2 — the Fix** (coming soon) we build it. Each account gets a CloudTrail delivered write-only into a single bucket in a dedicated audit account, so every account's control-plane history lands somewhere you can query together. The two questions that cost us most of an hour — which account owns this zone, and who changed this record — collapse into one query that returns in seconds. We also cover the bucket policy and partition layout that keep the cost sane, and what broke during the cutover.

### Operator checklist

If you are staring at an unreachable service right now, this is the order that would have saved us the hour.

```bash
# 1 — Which failure is it? NODATA and NXDOMAIN mean opposite things.
dig app.example.com               # status: NOERROR + ANSWER: 0  -> NODATA (name exists, type missing)
                                  # status: NXDOMAIN             -> the name itself is gone
nslookup app.example.com          # "No answer" vs "can't find ... NXDOMAIN"

# 2 — Rule out a cached answer and the near misses.
dig @<zone-nameserver> app.example.com    # aa set = straight from zone data, not a cache
dig app.example.com AAAA                  # are IPv6 clients quietly succeeding?
dig app.example.com ANY                   # which record types DO still exist at this name?

# 3 — Which account owns the zone? Repeat per profile; there is no cross-account search.
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

Two habits worth keeping. Only one `--lookup-attributes` is allowed per call, so filter on the most selective attribute and post-process the rest. And check the `SERVER:` line in any `dig` output before you believe it — it is the difference between reading your zone and reading someone else's.

### References

- [RFC 2308 — Negative Caching of DNS Queries](https://www.rfc-editor.org/rfc/rfc2308.html) — defines NODATA (RCODE `NOERROR` with no answer records) and specifies the negative-cache TTL as `min(SOA record TTL, SOA MINIMUM)`, not the `MINIMUM` field alone.
- [Logging Amazon Route 53 API calls with AWS CloudTrail](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/logging-using-cloudtrail.html) — CloudTrail records every Route 53 API call, including who made it and from which IP. Three things are easy to conflate: Route 53 is a global service, not a regional one; its management events are recorded as global service events, typically surfaced through `us-east-1`, which is why the lookups here pin that Region; and what you can see depends on where you look — Event History is per account, per Region, 90 days, while a delivered trail depends on that trail's settings, including whether global service events are included.
- [Get notified when changes are made to Route 53](https://repost.aws/knowledge-center/route53-change-notifications) — the per-account EventBridge-on-`ChangeResourceRecordSets` pattern, i.e. the shape of the notification pipelines we started with.

### Appendix: reproduce it yourself

The original incident's terminal output contained internal names I can't publish, so the `dig` and `nslookup` blocks above were reproduced in a throwaway CoreDNS lab. That is why the server is `127.0.0.1:5300` and every address is from the `192.0.2.0/24` documentation range. The behaviour is identical to that morning: `NOERROR`, an empty answer, a parent-zone SOA.

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

**The control case.** Delete the TXT as well, so the name holds no records at all, bump the serial again, reload, and re-run the same query. It flips to `NXDOMAIN` — the loud, honest failure we would have preferred to get, and the one-line difference between an hour of looking at the wrong tier and a diagnosis in the first minute.

Two things trip people up. Bump the SOA serial on every edit or the reload is a no-op. And keep `@127.0.0.1 -p 5300` on every query: drop it and you are asking the public internet about the real `example.com`, which answers NODATA for its own reasons and looks deceptively like success. Check the `SERVER:` line to be sure you queried the lab.
