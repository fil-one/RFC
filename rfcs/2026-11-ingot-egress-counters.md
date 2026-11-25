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

The write adds to the row for (space, hour), so several Ingot instances sharing a database count into the same rows. A crash loses at most 5 seconds of counts. Egress is under-counted after a crash and never counted twice.

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

A task on Ingot's task runner runs every 10 minutes, which is configurable. It sends every row whose `bytes_sent` differs from `bytes_reported`, up to 1,000 hours in one invocation, and sets `bytes_reported` to the value it sent when Hilt answers.

The hour still in progress is sent with the rest. A report carries an hour's total so far, and Hilt replaces what it holds, so the same hour is sent again each time its count has grown. Hilt is therefore at most one interval behind, though the counts are kept by the hour.

When more than 1,000 hours are waiting, after many buckets were read or Hilt was unreachable, the task sends further invocations, oldest hours first, until none are left. Each invocation is acknowledged on its own, and one space's hours may be split across several, since Hilt stores each hour separately.

The command is named for usage, with egress as one field of its arguments, so that it can carry other counts only the gateway knows. Two are expected to follow:

- **Object counts.** Ingot registers every object with the upload service today only so that it can be counted there.
- **Logical bytes stored.** The sum of object sizes as the customer sees them. The upload service counts the bytes a space holds, which include Ingot's catalog segments and the overhead of encryption, and will include parity once blobs are erasure coded. Only Ingot knows what the customer stored.

The entries are tuples, grouped by space, so a space's DID is sent once however many hours it has. A report names no tenant. Hilt knows each bucket's tenant and provider, and looks them up by space.

Hilt accepts a report only from an issuer it has registered as a provider. It stores one row for each (provider, space, hour), where the provider is that issuer, with the bucket's tenant beside it. A report replaces the row's count, so one that is repeated, or resent after a lost response, leaves the same totals. Hilt keeps the rows for as long as the tenant exists: a bucket read in every hour of a year holds 8,760.

One count Hilt cannot use does not fail the report. Hilt skips the counts for a space it does not know, or for a bucket that another provider serves, stores the rest, and names the spaces it skipped in its receipt. Ingot marks the skipped rows as reported, so they are not sent again, and counts them in a metric. A report fails as a whole only when its issuer is not a registered provider or it cannot be read.

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

## Operations

- **Storage.** The counts live in Ingot's Postgres schema with the rest of its state, and are backed up and restored with it. A bucket read in every hour of a week holds 168 rows. No new filesystem or quota is needed.
- **Metrics.** Ingot reports the number of rows waiting to be reported, how long the oldest unreported count has waited, and the counts Hilt skipped.
- **Alerts.** An Ingot whose oldest unreported count has waited longer than three intervals, and any count Hilt skipped. This catches reports that are failing. Detecting an Ingot that is down is left to the monitoring that already watches the appliance.
- **Delegations.** None are added or changed, for an appliance or for an access key. Removing a provider from Hilt stops its reports being accepted.

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
- A report holding a count for a space Hilt does not know stores its other counts, and Ingot does not send the skipped count again.
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
