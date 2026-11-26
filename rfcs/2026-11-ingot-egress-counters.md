# RFC: Egress counters for Ingot

Status: Experimental

## Authors

- [Alan Shaw](https://github.com/alanshaw)

## Motivation

Trial accounts are to be disabled once they pass the trial egress limit (2 TB in fil-one today), and nothing records egress on a Forge network. fil-one's usage job already sums an egress series it asks Hilt for, and Hilt already returns one for a tenant and for a bucket. The series is always empty, because nothing feeds it.

This RFC proposes that Ingot counts the bytes of object data it sends to S3 clients, per bucket and per hour, and pushes the counts to Hilt every 10 minutes. Hilt serves them in the series it already has.

It is an alternative to [Egress tracking for Ingot](https://github.com/fil-one/RFC/pull/41), which records a signed receipt for every read and reports them through the egress tracking service. The two are compared under [Alternatives considered](#signed-receipts-through-the-egress-tracking-service).

## Goals

- Hilt's egress series for a bucket and for a tenant reports what Ingot sent, within minutes.
- A count does not depend on which tier served a read, on encryption, or on the egress tracking service being deployed.
- Counting adds no network call, no signature and no database write to a GET.
- A report that fails or is sent twice changes no total.

## What is counted

Egress is the bytes of object data in the bodies of GetObject responses. Ingot counts them where it hands an object's body to the response, as the response reads them, and adds each byte to the hour in which it was sent.

- **Encryption does not change the count.** The body is counted after decryption and after trimming to the requested range, so a 4 KiB ranged GET counts 4 KiB whatever was read from disk or from a storage node to produce it.
- **The cache does not change the count.** A cache hit and a cache miss send the same bytes.
- **An abandoned GET counts what was sent** before the client went away. The count is taken as the response reads the body, so it can be up to one buffer more than reached the client.
- **A GET that runs across an hour boundary** adds to each hour the bytes sent in it.
- **Server-side reads are not counted.** A copy reads its source through a different path, and catalog blocks are never part of a response body.
- **Response headers and TLS overhead are not counted.** AWS S3 ignores `Accept-Encoding`, so a body is sent as it was stored and no compression needs accounting for.

## Design

```mermaid
sequenceDiagram
    participant Client as S3 client
    participant Ingot
    participant Hilt
    participant Console as fil-one

    Client->>Ingot: GetObject
    Ingot--)Client: bytes
    Ingot->>Ingot: add bytes to (bucket, hour)

    Note over Ingot,Hilt: every 10 minutes
    Ingot->>Hilt: /s3/usage/report (hours whose count changed)
    Hilt->>Hilt: store counts
    Hilt--)Ingot: receipt

    Note over Hilt,Console: every 12 hours
    Console->>Hilt: GET /tenants/{id}/metrics
    Hilt--)Console: egress series
```

### Counting in Ingot

Ingot keeps a running count in memory for each bucket's space and the current hour, in UTC. Every 5 seconds it adds the counts to a table in its Postgres schema and clears them:

| Column | Meaning |
| --- | --- |
| `space` | the bucket's space |
| `hour` | start of the hour, UTC |
| `bytes_sent` | bytes sent in that hour so far |
| `bytes_reported` | the value of `bytes_sent` Hilt last acknowledged |

The write adds to the row for (space, hour), so several Ingot instances sharing a database count into the same rows. A crash loses at most 5 seconds of counts, so egress is under-counted after a crash. A write that fails is retried. If it had in fact been saved and only its reply was lost, its bytes are added twice, which over-counts by at most 5 seconds of one instance's traffic. That is accepted.

A bucket with no reads in an hour has no row for it.

### Reporting to Hilt

A new command, `/s3/usage/report`, carries counts from Ingot to Hilt. The counts are Ingot's own account of what it sent, so Ingot is the subject of the invocation as well as its issuer, and Hilt is the audience. An invocation on the issuer's own authority needs no delegation.

```jsonc
{
  "iss": "did:web:ingot.example.com",
  "aud": "did:web:hilt.example.com",
  "sub": "did:web:ingot.example.com",
  "cmd": "/s3/usage/report",
  "args": {
    // one entry for each space
    "egress": [
      [
        "did:key:zBucketSpaceA",
        [
          // [start of hour (Unix seconds), bytes sent in it so far]
          [1795600800, 5368709120],
          [1795604400, 1048576]
        ]
      ],
      [
        "did:key:zBucketSpaceB",
        [
          [1795600800, 4096]
        ]
      ]
    ]
  }
}
```

Ingot reports every 10 minutes, which is configurable. A report sends every row whose `bytes_sent` differs from `bytes_reported`, up to 1,000 hours in one invocation, and sets `bytes_reported` to the value it sent when Hilt answers.

The hour still in progress is sent with the rest. A report carries an hour's total so far, and Hilt keeps the largest total it is sent, so the same hour is sent again each time its count has grown. Hilt is therefore at most one interval behind, though the counts are kept by the hour.

When more than 1,000 hours are waiting, after many buckets were read or Hilt was unreachable, Ingot sends further invocations, oldest hours first, until none are left. Each invocation is acknowledged on its own, and one space's hours may be split across several, since Hilt stores each hour separately.

The command is named for usage, with egress as one field of its arguments, so that it can carry other counts only the gateway knows:

- **Object counts** are a second field of the same arguments, described under [Object counts](#object-counts).
- **Logical bytes stored** are expected to follow. They are the sum of object sizes as the customer sees them. The upload service counts the bytes a space holds, which include Ingot's catalog segments and the overhead of encryption, and will include parity once blobs are erasure coded. Only Ingot knows what the customer stored.

The entries are tuples, grouped by space, so a space's DID is sent once however many hours it has. A report names no tenant. Hilt knows each bucket's tenant and provider, and looks them up by space.

Hilt accepts a report only from an issuer it has registered as a provider. It stores one row for each (provider, space, hour), where the provider is that issuer, with the bucket's tenant beside it. An hour's total only grows, so Hilt keeps the larger of the row's count and the reported one. A report that is repeated, resent after a lost response, or delivered after a newer one leaves the same totals, and Ingot instances sharing a database can report without coordinating. Hilt keeps the rows for as long as the tenant exists: a bucket read in every hour of a year holds 8,760.

One count Hilt cannot use does not fail the report. Hilt leaves out the counts for a space it does not know, or for a bucket that another provider serves, and stores the rest. Its receipt carries a warning for each space left out, with a `name` (`UnknownBucket` or `BucketRegionMismatch`), a `message` and the `space`. A report fails as a whole only when its issuer is not a registered provider or it cannot be read.

A report Hilt accepts settles every count in it, warned about or not, so Ingot marks them all as reported and sends none again. It logs each warning with its message and counts it under its name. A warning is a named entry, where a list of skipped spaces could say only one thing, so later counts can bring warnings of their own.

Hilt deletes a bucket's record when the bucket is deleted, and a report for the bucket's last hours can arrive afterwards, long afterwards if the appliance was offline. Hilt therefore keeps the space, tenant and provider of a deleted bucket for as long as the tenant exists, and attributes and checks late counts against that.

Ingot sends nothing when it has no rows to report.

Ingot deletes a row 7 days after it was last reported. Until then a report can be repeated if Hilt loses data.

### Serving from Hilt

Hilt's metrics endpoints already return an egress series beside storage and ingress, for a tenant and for a bucket, in windows of whole hours. Hilt fills each sample with the sum of the hours that start in its window: a bucket's from the rows for its space, a tenant's from the rows stored with that tenant, which include its deleted buckets'. A window with no rows reports zero. Counts that arrive later, up to the reporting interval behind or longer after an outage, raise the windows they belong to.

fil-one needs no change to read it. Its usage job asks for each tenant's series from the start of the current billing period to now, sums it and applies the limit, so a tenant's egress is already totalled across its buckets. The limit is on egress within a billing period.

### How late a count is

A byte sent is in Ingot's database within 5 seconds and at Hilt within about 10 minutes at the default interval. fil-one's usage job runs every 12 hours, and that interval decides how far past the limit a trial account can read.

The job sums the whole period again on every run and keeps no running total. A count that reaches Hilt late, after an outage, is missing from one run and included in the next, so a late report delays the limit by at most one run and loses no egress.

### Trust

Hilt takes Ingot's counts on Ingot's word. Ingot already authenticates every request, enforces every bucket policy and decrypts every object, so the counts add little to what is already trusted of it. Appliance operators are not paid for egress, so an operator gains nothing from a wrong count.

Storage nodes are a different case and this RFC does not change it. A node's egress is evidenced by the `/content/retrieve` receipts it issues for invocations its clients sign, reported through the egress tracking service.

## Object counts

Hilt's metrics endpoints also return an object count, for a tenant and for a bucket. Counting objects at the upload service needs Ingot to register every object version there with an `/upload/add` invocation and take it away again with `/upload/remove`, and the registration serves no other purpose. Ingot counts its own objects instead and reports them in the same `/s3/usage/report`.

### What is counted as an object

Ingot counts what AWS reports as `NumberOfObjects` for a general purpose bucket:

- every version of every key, current or not
- every delete marker
- every part of a multipart upload in progress

A PutObject to an unversioned bucket adds one object and removes the one it replaces. A delete that writes a delete marker adds one. Completing a multipart upload removes its parts and adds the object they form, and aborting one removes its parts. Uploading a part number a second time changes nothing.

### Counting objects in Ingot

Each bucket keeps two totals for its whole life: the objects it has gained and the objects it has lost. Neither ever goes down. The transaction that changes a bucket's objects adds to them, and copies both to the row for the bucket's space and the current hour:

| Column | Meaning |
| --- | --- |
| `space` | the bucket's space |
| `hour` | start of the hour, UTC |
| `added` | objects the bucket had ever gained, as of the last change in the hour |
| `removed` | objects the bucket had ever lost, as of the same change |
| `added_reported`, `removed_reported` | the values Hilt last acknowledged |

The counts commit with the change they count, so a crash loses none and a retried write counts nothing twice. The hour is taken from the database's clock, so instances sharing a database agree on it.

The bucket holds `added` less `removed`. Two totals are kept, where one net count would do for Ingot, so that reports stay safe to repeat. A net count goes up and down, and Hilt could not tell a late report from a new one. Each total only grows, so Hilt keeps the largest of each, as it does for egress.

A bucket whose objects did not change in an hour has no row for it.

### Reporting object counts

The counts travel in a second field of the arguments, `objects`, shaped like `egress`:

```jsonc
{
  "cmd": "/s3/usage/report",
  "args": {
    "egress": [],
    "objects": [
      [
        "did:key:zBucketSpaceA",
        [
          // [start of hour (Unix seconds), objects ever gained, objects ever lost]
          [1795604400, 1204, 37]
        ]
      ]
    ]
  }
  // ...
}
```

Ingot sends them on the same schedule as egress, in invocations of their own, with the same limit of 1,000 hours each. A row is sent whenever either total differs from the one Hilt acknowledged, the hour in progress included. Warnings, acknowledgement and the 7 days a reported row is kept are the same as for egress.

### Serving object counts from Hilt

Hilt stores one row for each (space, hour) and keeps the larger of each total it is sent. A sample's object count is `added` less `removed` in the latest hour that starts before the end of the sample's window. It is zero when the bucket has no such hour, or when `removed` is the larger. A tenant's count is the sum of its buckets'.

### What object counts replace

Ingot sends no `/upload/add` or `/upload/remove` invocation, and Hilt delegates neither command to an access key for any S3 permission.

Counting starts from zero when this is deployed. Objects stored before then are not counted, so a bucket that already held some reports fewer than it holds, and reports none while it has lost more objects than it has gained since. Such a bucket's count stays wrong until its environment is reset, which is accepted.

Ingot authorizes a request locally only when the access key holds a delegation for every command Hilt's permission table lists for the action. An Ingot built on a table that still lists the upload commands, facing a Hilt that has stopped delegating them, asks Hilt to authorize every write and delete. So on each stage Ingot moves to the table without them before Hilt stops delegating them.

## Operations

- **Storage.** The counts live in Ingot's Postgres schema with the rest of its state, and are backed up and restored with it. A bucket read in every hour of a week holds 168 rows. No new filesystem or quota is needed.
- **Metrics.** Ingot reports the number of rows waiting to be reported and how long the oldest unreported count has waited, for egress and for object counts each, and Hilt's warnings by name.
- **Alerts.** An Ingot whose oldest unreported count has waited longer than three intervals, and any warning from Hilt. This catches reports that are failing. Detecting an Ingot that is down is left to the monitoring that already watches the appliance.
- **Delegations.** None are added, for an appliance or for an access key. Object counts take the two upload commands away from access keys (see [What object counts replace](#what-object-counts-replace)). Removing a provider from Hilt stops its reports being accepted.

## Alternatives considered

### Signed receipts through the egress tracking service

[Egress tracking for Ingot](https://github.com/fil-one/RFC/pull/41) has Ingot invoke a new `/content/serve` command on itself for each blob it serves, sign the receipt, journal both, and report batches to the egress tracking service, which totals them. It gives one mechanism for Ingot and for storage nodes, a signed record of every read that can be totalled again after a counting bug, and one place to total several Ingots.

The receipts do not make Ingot's figures independent of Ingot. Ingot issues the invocation and signs the receipt, and an Ingot that under-reported would write no record. S3 authentication is symmetric, so whoever can verify a client's request can also forge one, and a gateway cannot hold evidence that only its client could have produced.

The cost of that design falls on the trial limit. It waits for the egress tracking service to move to UCAN 1.0 and be deployed, and it needs a new grant on every access key, a journal with its own storage and quota, a batch endpoint on the S3 listener and two signatures for each blob read.

### Hilt pulls from each Ingot

Hilt could ask each Ingot for its counts on a schedule. Hilt would control the load, and a failed request would be an immediate sign of trouble. It needs Hilt to know and reach every Ingot, and a UCAN endpoint on Ingot's S3 listener, which serves only S3 today. Counts not yet collected would be stuck on an appliance that is offline or being retired.

### An access log

Ingot could write a line for each GET and ship the log, and totals would be computed centrally. Every read would be itemised and totals could be recomputed. The log is far larger than the counts, needs its own shipping and retention, and the trial limit needs none of the detail.

## Evaluation criteria

- A GET of a known number of bytes adds that number to the bucket's and the tenant's egress at Hilt.
- A ranged GET of an encrypted object adds the length of the range.
- A GET abandoned part way adds what was sent, to within one buffer, and a CopyObject adds nothing.
- A report holding a count for a space Hilt does not know stores its other counts and comes back with an `UnknownBucket` warning for that space, and Ingot does not send the count again.
- A PutObject adds one to the bucket's object count at Hilt, and deleting that version takes it away. A multipart upload counts one for each part while it is in progress and one once it is complete.
- An object count report that is repeated, or delivered after a newer one, leaves Hilt's count unchanged.
- The same requests add the same egress with Ingot's cache enabled and with `cache_writes: false`.
- Hilt accepts a report that carries no proofs from a registered provider, and refuses one from any other issuer.
- With Hilt unreachable for an hour and then restored, Hilt's totals equal Ingot's, with nothing counted twice.
- These run against the smelt stack in CI.

## Order of work

1. Add `/s3/usage/report` to libforge.
2. Hilt: store the counts, keep a deleted bucket's tenant and provider, and fill the egress series.
3. Ingot: count, store and report, with the metrics above.

## References

- [Egress tracking for Ingot](https://github.com/fil-one/RFC/pull/41)
- [Hilt](https://github.com/fil-forge/hilt)
- [Ingot](https://github.com/fil-forge/ingot)
