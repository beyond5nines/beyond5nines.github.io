---
layout: post
title: "NOERROR, No App — Part 4: The Fix"
short_title: "Part 4: The Fix"
date: 2026-06-20 12:00:00 -0000
categories: aws route53 cloudtrail athena
series: "NOERROR, No App"
author: Rahul Sharma
excerpt: "One org trail, one S3 bucket, one Athena table — replaces N per-account pipelines. Partition projection gives sub-second queries across every account without a crawler or ETL."
redirect_from:
  - /part4-the-fix/
---

*Part 4 of the [NOERROR, No App](/noerror-no-app/) series. [Part 1](/noerror-no-app-01/) has the incident; [Part 3](/noerror-no-app-03/) has the options we weighed.*

## Where we left off

In **[Part 1](/noerror-no-app-01/)**, a deleted Route 53 record took the app down while DNS looked healthy — a **NODATA** answer, not the louder `NXDOMAIN` (the mechanics and a reproducible lab are in [Part 2](/noerror-no-app-02/)). Finding *which of three accounts* held the zone, then *who* deleted it, cost 75 minutes of hand-searching 60-plus records across three consoles — because at the time there was no audit path at all. The per-account notification pipelines we built afterwards ([Part 3](/noerror-no-app-03/)) answered the wrong question and aged independently. Audit is a read-side concern; it belongs in one place. [Part 3](/noerror-no-app-03/) has the cost comparison against AWS Config, self-managed Elasticsearch/OpenSearch and CloudTrail Lake — S3 + Athena won because it charges for retention and for the questions you actually ask, not for data arriving; CloudWatch, where AWS now steers ex-Lake users, has the same standing-ingestion problem as the others. Here's the one place.

## The fix: one trail, one bucket, one place to query

Invert it. Instead of N pipelines pushing notifications out, every account's own CloudTrail delivers into one central bucket, and one query surface reads across all of them.

```
each account ── own CloudTrail ──►  audit account: S3 (cloudtrail/)
                (bucket policy,         │  partition projection
                 write-only)            ▼
                                      Athena
                                        │
                      EventBridge (hourly) → Query Lambda
                                        │
                                        ▼
                               SNS → Slack / Email / Ticket
```

#### 1. A trail per account, delivered write-only to a dedicated account

Here's a constraint that shaped the whole design: these were **separate, standalone AWS accounts — never enrolled in an AWS Organization.** That rules out the tidy path most write-ups assume (an organization trail at a management account that captures every account automatically). With no Org, there's no org trail, no management account, and — as you'll see — no SCPs either. (Running Control Tower already? Its landing zone gives you that org trail out of the box — skip most of this post and point Athena at the log-archive bucket it's already filling.)

So the fix is deliberately low-tech: **each account runs its own CloudTrail, and each delivers to the same S3 bucket in a dedicated audit account** via a cross-account, write-only grant. Same end state — every account's control-plane events in one place — assembled from standalone parts instead of inherited from an Org.

The bucket policy is what makes cross-account delivery both work and stay locked down. CloudTrail writes; nothing else does. The non-obvious parts are the per-account `aws:SourceAccount` / `aws:SourceArn` conditions (without them the policy is either rejected or too broad) and the account-ID in the object path. CloudTrail also needs an `s3:GetBucketAcl` statement on the bucket itself — shown here alongside the write, with one source account as the example (repeat the `SourceAccount`/`SourceArn` pair, or list the account IDs, for each account you onboard):

```json
[
  {
    "Sid": "AWSCloudTrailAclCheck",
    "Effect": "Allow",
    "Principal": { "Service": "cloudtrail.amazonaws.com" },
    "Action": "s3:GetBucketAcl",
    "Resource": "arn:aws:s3:::audit-cloudtrail"
  },
  {
    "Sid": "AWSCloudTrailWrite",
    "Effect": "Allow",
    "Principal": { "Service": "cloudtrail.amazonaws.com" },
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::audit-cloudtrail/AWSLogs/111122223333/*",
    "Condition": {
      "StringEquals": {
        "s3:x-amz-acl": "bucket-owner-full-control",
        "aws:SourceAccount": "111122223333"
      },
      "ArnLike": {
        "aws:SourceArn": "arn:aws:cloudtrail:us-east-1:111122223333:trail/audit-trail"
      }
    }
  }
]
```

That — a trail in each account writing to a bucket in an account those workloads can't read or delete from — is the whole fix to the *visibility* problem. Ten drifting notification pipelines collapse into a plain log-delivery path with no application code in it. The honest tradeoff versus an org trail: onboarding a new account isn't automatic — you point one more trail at the bucket and add its `SourceAccount` to the policy. A one-time step per account, not a new pipeline per account.

#### 2. Securing the audit account

Centralizing audit only helps if the audit itself can't be quietly altered or read by the accounts it's watching:

* **Tamper-evidence.** Versioning + S3 Object Lock make delivered logs WORM. A note on the mode, because it's a one-way door: compliance mode is irreversible — once an object is written with a retention date, nobody (not admins, not the account root, not AWS) can delete it or shorten that date until it expires, and you can't turn the mode off. That's exactly what a regulation like SEC 17a-4 wants, but set a 7-year retention on a high-volume audit bucket and you've committed to paying to store all of it for 7 years with no escape hatch. The honest framing: default to governance mode, with the bypass permission (`s3:BypassGovernanceRetention`) withheld from everyone except a break-glass role — you get the same tamper-resistance day to day, but you can still unwind a fat-fingered retention period. Reserve compliance mode for when a regulation actually requires provably unbreakable immutability, and only once you're certain of the retention length.
* **Isolation, not just policy.** The trail and bucket live in a dedicated audit account. Member accounts hold only a write-only cross-account role scoped to their own prefix — no read, no delete, and no CloudTrail permissions at all. There's nothing in a workload account to disable the trail with, because the trail isn't there.
* **Encryption.** SSE-KMS with a key whose policy grants `kms:GenerateDataKey*` to `cloudtrail.amazonaws.com` and decrypt only to the audit-reader roles.
* **Who can query.** Read access is scoped to the audit account. If multiple teams query, Lake Formation gives column- and row-level grants (e.g. a team sees only its own `recipientaccountid`) without minting per-team buckets.

#### 3. Make it queryable without a crawler bill

CloudTrail lands gzipped JSON partitioned by account, region, and date. The cheap way to query it is **partition projection** — you describe the partition scheme once in the table DDL and Athena computes partitions at query time. No Glue crawler running on a schedule, no `MSCK REPAIR`, and — critically — partition *pruning*, so a query only scans the prefixes it needs.

Every `${var}` in the location template needs a matching projection definition; `account`, `region`, and `log_date` all get one. Name the date partition column `log_date` rather than `date` — `date` is a reserved word in Athena/Presto DDL, and using it as a bare column name will bite the first time someone copies this table definition and hits a parser error. Since you know your account IDs, list them as an `enum` (cleaner than `injected`, and it lets a query span all accounts without supplying one):

```sql
TBLPROPERTIES (
  'projection.enabled' = 'true',
  'projection.account.type' = 'enum',
  'projection.account.values' = '111122223333,444455556666,777788889999',
  'projection.region.type' = 'enum',
  'projection.region.values' = 'us-east-1,us-west-2,eu-west-1',
  'projection.log_date.type' = 'date',
  'projection.log_date.range' = '2026/01/01,NOW',
  'projection.log_date.format' = 'yyyy/MM/dd',
  'storage.location.template' =
    's3://audit-cloudtrail/AWSLogs/${account}/CloudTrail/${region}/${log_date}/'
)
```

That's what makes queries fast and cheap: filter on `region` and `log_date` and you scan a couple of prefixes, not the whole lake. Size your hot window (Standard storage class) to how far back you actually query, and move anything older to a colder tier only if you're confident you won't need to query it directly.

#### 4. The query we couldn't run that night

Route 53 is a global service, so its events land in `us-east-1`. The change is in `requestParameters.changeBatch.changes[]`, an array that can mix `CREATE` and `DELETE` actions in one call — so you unnest it and filter on the action, not a substring of the blob:

```sql
WITH r53 AS (
  SELECT eventtime,
         recipientaccountid AS account,
         useridentity.type  AS caller_type,
         -- Resolve the human: IAM username, else the role-session name.
         COALESCE(
           useridentity.username,
           regexp_extract(useridentity.arn, '/([^/]+)$', 1)
         ) AS human,
         useridentity.sessioncontext.sessionissuer.username AS via_role,
         sourceipaddress AS source_ip,
         CAST(json_extract(requestparameters, '$.changeBatch.changes')
              AS ARRAY(JSON)) AS changes
  FROM   cloudtrail_logs
  WHERE  region = 'us-east-1'                           -- partition prune (Route 53 is global)
    AND  log_date BETWEEN '2026/06/12' AND '2026/06/13' -- partition prune
    AND  eventname = 'ChangeResourceRecordSets'
    AND  errorcode IS NULL                              -- successful calls only
)
SELECT eventtime, account, human, via_role, caller_type, source_ip,
       json_extract_scalar(c, '$.action')                 AS action,
       json_extract_scalar(c, '$.resourceRecordSet.name') AS record
FROM   r53
CROSS JOIN UNNEST(changes) AS t(c)
WHERE  json_extract_scalar(c, '$.action') = 'DELETE'
  AND  json_extract_scalar(c, '$.resourceRecordSet.name') = 'app.example.com.'  -- trailing dot
ORDER  BY eventtime DESC;
```

One row: timestamp, the **account** the zone lived in, the **human** who issued the delete — resolved from the role-session name, which under IAM Identity Center is their email or username — the role they assumed to do it, and the source IP. *Who, when, where*, across every account, scanning two partitions. The hour-long, three-console hunt is a query that returns before you've finished reading it.

The `human` column resolves cleanly when sessions carry a person — IAM users, or SSO/Identity Center sessions where the session name is the user's login. If your roles use non-human session names (CI pipelines, generic automation), `human` falls back to that string; then `via_role` plus `source_ip` are the lead, and you pin the person by correlating the session with its matching `AssumeRole` event.

(Exact partition names and the `useridentity`/`requestparameters` typing depend on your table DDL; adjust the JSON paths to match yours.)

#### 5. The hourly sweep

The only automation in the design: an hourly EventBridge schedule runs one Athena query — mutating calls across all accounts, today's `log_date` partition only — and posts a digest to SNS, fanning out to Slack, email, or a ticket. One Lambda, running in the audit account, scan bounded by the same partition filter as everything else.

## Honest about what this is — and isn't

This is a forensics and trend system, not a real-time alarm:

- **It's detective, not preventive.** It tells you who deleted the record; it doesn't stop the delete. For a load-bearing record, pair it with change control and tighter IAM on who can call `ChangeResourceRecordSets` for that zone.
- **The trail can be turned off.** Each account's trail is local, and with no Organization there's no SCP to stop `cloudtrail:StopLogging` there. Deny that action (and `DeleteTrail`/`UpdateTrail`/`PutEventSelectors`) to everyone but a break-glass role, and alarm on delivery gaps per account prefix — a stopped trail is silence, not an error.
- **The cadence is forensic.** CloudTrail delivery to S3 lags the API call by up to ~15 minutes, and the hourly sweep adds up to an hour worst case to alert — fine for "what changed across the accounts," not for standing between you and a hard-down outage. When seconds matter on a specific dangerous call, pair this with an **EventBridge rule** matching the event pattern (`Delete*`, `Terminate*`, Route 53 record deletes) straight to SNS: Athena stays the system of record, the rule is the fast tap on the shoulder.

## The cost that never showed up on the AWS bill

The Athena line item is the small number. The one that matters is the cost this replaced: ten near-identical notification stacks to patch, rotate secrets in, and upgrade, plus a new one every time an account was spun up — on-call and platform time that grew with every account added. Maintenance now drops to one delivery path with no application code, so onboarding an account is roughly a bucket-policy line, and "who changed this, where" goes from an hour across three consoles to a sub-second query.
