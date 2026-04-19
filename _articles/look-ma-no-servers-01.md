---
layout: post
title: "The specified subnet does not have enough free addresses to satisfy the request - Investigation"
date: 2026-04-09 12:00:00 -0000
categories: aws glue serverless
series: Look Ma! no servers
redirect_from:
  - /2026/04/09/look-ma-no-servers-01/
---

## The 7AM Ritual

Our daily batch consisted of 2 critical Glue jobs per weekday and 2 non-critical Glue jobs every 2 hours. Every morning at 7:00 AM, our batch ETL jobs kicked off. By 7:03 AM, Slack lit up.

```
Job: customer-analytics-daily
Status: FAILED
Duration: 0 minutes
Error: The specified subnet does not have enough free
       addresses to satisfy the request
```

1–2 intermittent failures everyother weekday morning. Data warehouse SLAs slipping. Downstream teams blocked before they'd finished their first coffee. And the thing that made it maddening: our calculation confirmed that we had capacity. Plenty of it.

Something was consuming IPs that we couldn't see.

## What We Didn't Understand About Glue's Networking

The first thing we had to accept was that "serverless" is a marketing word. The network is still there. The servers are still there. AWS just hides them behind an abstraction — which means the constraints are real, but the visibility is something you have to build yourself.

Our Glue jobs pull custom Python libraries and dependencies from a self-hosted JFrog Artifactory inside of a private VPC. To reach it, every Glue job binds to a Glue Connection — a VPC and a subnet. And every Glue worker that spins up grabs an Elastic Network Interface: a virtual NIC holding a real IP from the subnet's CIDR, for the lifetime of the job.

Four concurrent jobs × ten workers each = 40 IPs. From a /26 with 59 usable. We'd sized for ~5 minutes of ENI cleanup lag between runs — a reasonable assumption, since AWS doesn't document otherwise. On paper, fine. In practice, failing.

> AWS reserves 5 IP addresses per subnet, so a /26 provides 59 usable addresses.

So we started looking at the ENIs themselves.

### Step 1: What's actually in the subnet right now?

We ran this during a live failure at 7:03AM:

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=subnet-id,Values=subnet-xxxxx" \
  --query 'NetworkInterfaces[*].Status' \
  --output text | sort | uniq -c
```

The output stopped us cold:

```
  8 available
 50 in-use
```

58 ENIs. In a subnet with 59 usable IPs. The math was already over before our first job tried to launch.

But *50 in-use*? We were only running four jobs with ten workers each. That's 40. Where were the other twelve coming from?

### Step 2: Who owns the ENIs that shouldn't be there?

We pulled the full inventory with descriptions and requester IDs:

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=subnet-id,Values=subnet-xxxxx" \
  --query "NetworkInterfaces[*].[NetworkInterfaceId, Status, Description, RequesterId]" \
  --output table
```

Every ENI came back owned by an AWS-managed Glue service account. The 12 "extra" in-use ENIs had descriptions matching jobs that had finished hours ago. One of them was from the non-critical batch that had run at 5AM and completed cleanly around 6:00.

It was 7:03. Those ENIs had been attached, doing nothing, for over an hour and three minutes.

### Step 3: Why won't they go away?

We picked one of the ghost ENIs and asked it directly:

```bash
aws ec2 describe-network-interfaces \
  --network-interface-ids eni-xxxx \
  --query "NetworkInterfaces[0].Attachment"
```

```json
{
  "AttachTime": "2026-04-09T01:51:32+00:00",
  "AttachmentId": "eni-attach-xxxx",
  "DeleteOnTermination": false,
  "DeviceIndex": 1,
  "NetworkCardIndex": 0,
  "InstanceOwnerId": "xxxx",
  "Status": "attached"
}
```

The attachment block revealed two things. First, "DeleteOnTermination": false — meaning the ENI lifecycle is managed asynchronously by the Glue service rather than tied directly to instance termination. Second, and more surprising: the ENI had been in in-use state for over an hour after the job completed. in-use should mean an active workload. It didn't appear to. Glue was leaving ENIs attached to instances we couldn't see, for reasons we couldn't observe, long after the work was done. The flag told us cleanup would be async; the attach time told us how async.

That explained the 50. What about the 8 in `available`?

### Step 4: What about the "available" ones?

We pulled one of the `available` ENIs:

```bash
aws ec2 describe-network-interfaces \
  --network-interface-ids eni-yyyy \
  --query "NetworkInterfaces[0].Status"

"available"
```

No attachment block. No active workload. No visible reason to exist. And yet: still holding an IP. We watched one for 35 minutes before it finally disappeared. Ran it again a day later — 38 minutes. A third time — 32.

A pattern was forming. Glue's ENI lifecycle wasn't a two-state machine. It was four states, and three of them consumed subnet IPs:

```
CREATED → in-use (attached) → available (detached) → deleted
                   ^                    ^
          holds IP up to 60+ min   holds IP 30–40 min
```

Together, for a single ENI, they could pin an IP for sixty minutes after its job completed. Multiply that by a batch cadence that doesn't wait sixty minutes between runs, and you get 7:03AM every morning.

This isn't just us — it's a known pattern. See [AWS Glue does not clean up network interfaces](https://repost.aws/questions/QUrmpoQEFxRwWvt5Es6nGmaA/aws-glue-does-not-clean-up-network-interfaces) and [AWS Glue Job Not Releasing ENIs After Completion](https://repost.aws/questions/QUBdpMaxjcST-voDzcNld3bA/aws-glue-job-not-releasing-enis-after-completion), where one operator reports 1,200+ accumulated ENIs from the same root cause. Also [Glue Connections Running Out of IPs](https://repost.aws/questions/QUE2ZiLoMFSt-tl7OV5pySbg/glue-connections-running-out-of-ips).

### Step 5: Who else is fighting for this subnet?

Before we could plan a fix, we needed to know the full blast radius. Every Glue connection bound to the subnet, every job bound to those connections:

```bash
aws glue get-connections \
  --query "ConnectionList[?contains(PhysicalConnectionRequirements.SubnetId, 'subnet-xxxxx')].Name" \
  --output table

aws glue get-jobs \
  --query "Jobs[?contains(Connections.Connections, '<connection-name>')].Name" \
  --output table
```

The answer was everything. Critical batch, non-critical batch, ad-hoc backfills, manual reruns — every Glue job in our account shared one /26. One misbehaving backfill could starve the entire morning SLA. It already had been.

We had the picture. Now we needed the fix.

## What We Tried First (And Why It Wasn't Enough)

Our first instinct was the obvious one: make the subnet bigger. We pulled up the VPC console, drafted the CIDR change, and were halfway through the Terraform PR before we stopped and asked: *does this actually solve it?*

A /24 gives you 251 usable IPs. Four times the headroom. But our ENI consumption wasn't static — we'd been bumping worker counts every quarter as data volumes grew, and job durations were creeping up with them. And here's the worse part — without the ghost ENI problem *visible*, we'd still hit silent 7:03AM failures, just less often. Less frequent failures that still blow the SLA are harder to catch, not easier. We closed the PR.

The second instinct was architectural: eliminate the VPC binding entirely with PrivateLink. No VPC, no ENIs, no problem. We spent an afternoon mapping what it would take — and hit a wall. PrivateLink requires an AWS service endpoint to front. Since our Artifactory ran outside AWS, this would require additional infrastructure we weren't prepared to build. Dead end.

Sitting in the postmortem the next morning, one of us drew the ENI lifecycle on the whiteboard next to the job launch timeline and said the thing that reframed it: *the subnet isn't too small. The launcher is blind.*

We were firing Glue jobs into a subnet without ever asking whether there was room. The fix wasn't more IPs. It was feedback.

In [Part 2](/look-ma-no-servers-02/) we cover the fix: how we split the subnet by blast radius, built an adaptive Airflow sensor that reads actual ENI state before launching, and what the observability looks like 90 days in.

## References

- [AWS Knowledge Center: Resolve "subnet does not have enough free addresses" in Glue](https://repost.aws/knowledge-center/glue-specified-subnet-free-addresses)
- [AWS Glue does not clean up network interfaces](https://repost.aws/questions/QUrmpoQEFxRwWvt5Es6nGmaA/aws-glue-does-not-clean-up-network-interfaces)
- [AWS Glue Job Not Releasing ENIs After Completion](https://repost.aws/questions/QUBdpMaxjcST-voDzcNld3bA/aws-glue-job-not-releasing-enis-after-completion) — 1,200+ accumulated ENIs from the same root cause
- [Glue Connections Running Out of IPs](https://repost.aws/questions/QUE2ZiLoMFSt-tl7OV5pySbg/glue-connections-running-out-of-ips)
