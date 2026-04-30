# RFC: Bucket metadata — canonical MST and operational database

**Author**: Forrest (@frrist)

**Date**: 2026-04-30

**Status**: Draft

## Abstract

ms3t buckets hold state in two layers: a content-addressed Merkle Search Tree (MST) in Forge that is the canonical, portable representation of user data, and a Postgres database that is the authoritative runtime store. This RFC defines what is in each layer, why, and how they relate. Postgres serves all S3 queries; the MST exists for credible exit, Forge-native durability, federation, and incremental snapshots. Service features (policies, lifecycle, multipart, IAM, locks) live only in Postgres.

## Motivation

A Storacha S3 bucket is "Web3" only if a customer can walk away with their bucket. The falsifiable test: hand them a CAR file and they reconstruct an interoperable bucket on any IPFS-aware platform. That requires a content-addressed, self-describing representation of bucket state — which the MST provides.

The current prototype (`pkg/ms3t/architectural.md`) uses the MST for both the write path and the read path. The MST is bad at the read path. For S3 prefix listing, range reads, multipart, IAM checks, and lifecycle, a relational store with proper indexes is the right shape. The supabase/storage project has demonstrated this design at scale; we are adopting its schema as prior art and re-using sprue's existing Postgres machinery.

But moving runtime queries to Postgres does not make the MST optional. The MST is what makes the body chunks in piri *reachable from Forge*: without a continuously-maintained MST root, the bytes are orphaned from the indexing-service's point of view. The MST is also the only verifiable identity for "the state of this bucket" — required for federation, replication, snapshots, and credible exit.

This RFC defines the split.

## What is an MST

A **Merkle Search Tree** is a content-addressed key-value tree. Every node is serialized, hashed, and identified by its CID; the root CID names the whole tree state at a point in time. Mutation is copy-on-write — any change rewrites the path of nodes from the affected leaf to the root, producing a new root CID. Unchanged subtrees keep their CIDs, so successive versions share structure.

For this RFC, MST keys are S3 object keys and values are CIDs of `ObjectManifest` blocks. The "current state of a bucket" is therefore a single CID — the MST root — from which every committed object is reachable.

The implementation in `pkg/ms3t/mst/` is forked from atproto, where the structure represents social-graph repos. The implications of that origin for S3 are discussed in §"Why MST" and §"Fanout and sizing".

## Why MST

The MST plays five roles, in priority order:

1. **Credible exit.** A CAR file containing the MST root, every reachable node, every reachable `ObjectManifest`, and every reachable body chunk is a complete, portable bucket. A recipient with no Storacha-specific code can reconstruct it.
2. **Forge-native durability.** The MST root is the discoverable, indexer-resolvable handle from which all bucket bytes are reachable. Without continuous MST commits, body chunks in piri are orphaned from Forge's perspective. The DB knows which chunks belong to which object; the MST root *is* the durable artifact in Forge.
3. **Federation and replication.** A content-addressed root makes "sync to root R" verifiable across operator boundaries. Postgres logical replication does not.
4. **Free incremental snapshots.** Every commit is a snapshot. Structural sharing makes N retained snapshots ≈ 1 in storage cost.
5. **Tamper-evident history.** Retained roots form a verifiable commitment chain.

The current MST in `pkg/ms3t/mst/` is forked from atproto's repo MST: 4-bit fanout, hash-keyed via `sha256(key)` leading-zero count (`pkg/ms3t/mst/mst_util.go:20-49`). atproto's design point is bounded social-graph repos with no prefix-listing requirement. S3 buckets violate all of those: keys can be 1KB, deeply hierarchical, prefix-listing is a first-class operation, and buckets can grow unbounded.

We resolve this by giving the MST one job: be the canonical content-addressed snapshot. Prefix listing and other queries go through Postgres. Fanout becomes a snapshot-efficiency knob (size of diff per write batch), not a query-efficiency knob. Tuning is deferred — see §"Fanout and sizing".

## Canonical state vs service state

The dividing rule is the **credible-exit test**: state belongs in the MST if a customer would want it on exit to another platform; otherwise it is service state and lives only in the database.

| Feature | MST | DB | Notes |
|---|---|---|---|
| Object body content (chunks) | ✓ | refs only | bytes always in piri; DB and MST hold CIDs, not bytes |
| Object manifest (Content-Type, user-meta, ETag, size, timestamps) | ✓ | ✓ mirror | intrinsic to "what the object is" |
| Object tags | ✓ | ✓ mirror | per-object user metadata; travels |
| Object versioning history | ✓ | ✓ mirror | versions are user data |
| Multipart upload state (in-flight) | | ✓ | service-only; on `CompleteMultipartUpload` the assembled object enters the MST |
| Bucket policy / IAM / ACL | | ✓ | service-enforced; no enforcer after exit |
| Bucket CORS, lifecycle, replication, notifications, website | | ✓ | service features over the bucket |
| Object Lock, retention, legal hold | | ✓ | service-enforced immutability contract |
| Bucket tags | | ✓ | operational unit metadata, not content |
| `bucket → root_cid` pointer | | ✓ | operational cursor |
| Owner mapping, audit logs, metrics | | ✓ | platform state |

Bucket tags are a borderline call. They describe the operational unit (`cost-center=engineering`) rather than user content; on exit, a recipient can re-tag the destination bucket. This RFC defaults them to DB-only and revisits if a portability use case appears.

## MST contents and types

The canonical structure is:

```
Bucket = MST<S3Key, ObjectManifest_CID>
```

The leaf value is the CID of an `ObjectManifest`. The proposed type, replacing the current shape at `pkg/ms3t/bucket/manifest.go:10-26`:

```go
type ObjectManifest struct {
  Content      cid.Cid           // body root: UnixFS or Filepack
  ContentType  string
  Created      int64             // unix seconds
  Modified     int64             // unix seconds (S3 Last-Modified)
  Size         uint64
  SHA256       []byte            // full-body sha256, ETag source
  UserMetadata map[string]string // x-amz-meta-*
  Tags         map[string]string
  Previous     cid.Cid           // prior manifest, version chain, cid.Undef == nil
  DeleteMarker bool
}
```

**Versioning** is modelled as a chain via `Previous`. The MST always uses the versioned shape; a bucket with versioning disabled simply means the service never surfaces anything past `current`. This keeps the MST shape stable across the bucket-versioning toggle. A delete is a new manifest with `DeleteMarker: true` and `Previous` linking the prior manifest.

**Body link.** `Content` is opaque to the MST. The target per-object body layout is defined by [`001-forge-s3-flat-file-sharding-strategy.md`](https://github.com/fil-one/RFC/pull/2) — the Forge integration of [RFC#65](https://github.com/storacha/RFC/pull/65) (Filepack) and [RFC #66](https://github.com/storacha/RFC/pull/66) (SDI v0.2 with inline `blocks`): the body is split into 256 MB shards, a UnixFS File root links the shards in order, and an SDI v0.2 inlines the UnixFS root for one-roundtrip retrieval. Under that scheme `Content` is the UnixFS File root CID; the SDI is the per-object indexable artifact published to the indexer, separate from the MST's index claim. Today's chunker (`pkg/ms3t/bucket/chunker.go:19-87`) in the MVP produces 1 MiB raw IPLD blocks pre-alignment, and `Content` points at a head-of-list block. The MST/manifest split is unaffected by either layout. Where this RFC says "chunks," read it generically as "the units of body data uploaded to piri" (1 MiB raw blocks today, 256 MB shards under [`001-forge-s3-flat-file-sharding-strategy.md`](https://github.com/fil-one/RFC/pull/2) RFC.

**MST node shape.** The existing `NodeData` and `TreeEntry` types at `pkg/ms3t/mst/mst.go:71-82` are unchanged — `Left` subtree pointer plus an ordered list of entries with key-prefix compression and per-entry `Tree` right-subtree pointers.

**Exit format.** A single CAR file containing the MST root + every reachable MST node + every reachable `ObjectManifest` + every reachable body chunk is a complete portable bucket. No service state is required to read it.

## Database schema

Postgres is the authoritative runtime store. **It holds metadata only** — CIDs, sizes, hashes, timestamps, user metadata, tags, policies, multipart upload state. Body bytes are content-addressed and live in piri; they never enter the relational store. The schema is heavily inspired by [supabase/storage](https://github.com/supabase/storage) (`migrations/tenant/`), which has solved most of these problems already and at scale. The core tables (column lists indicative, not literal SQL):

- **`buckets`**: `(id, name, owner_id, root_cid, forge_root_cid, public, file_size_limit, allowed_mime_types, created_at, ...)`. Superset of today's registry (`pkg/ms3t/registry/sqlite.go:14-20`). `root_cid` is the current MST root; `forge_root_cid` is the last root snapshotted to Forge.
- **`objects`**: `(id, bucket_id, name, version_id, manifest_cid, content_type, size, sha256, metadata jsonb, tags jsonb, created_at, modified_at, delete_marker, previous_manifest_cid, ...)`. One row per version. `manifest_cid` is the canonical handle into the MST.
- **`prefixes`**: materialized folder hierarchy with triggers, lifted from supabase. Powers `ListObjectsV2` prefix/delimiter semantics.
- **`s3_multipart_uploads`** and **`s3_multipart_uploads_parts`**: in-flight upload state. Service-only; never enters the MST.
- **`bucket_policies`, `bucket_lifecycle_rules`, `bucket_cors_rules`, `bucket_notifications`, `object_locks`, `object_retentions`, `object_legal_holds`**: service features over the bucket.
- **`snapshots`**: `(bucket_id, root_cid, committed_at, retained_until)`. Retained MST roots for time-travel and GC liveness calculation.

Migrations live under `internal/migrations/sql/` per existing sprue convention, applied by goose at startup. The supabase migrations to study, in order: `0002-storage-schema.sql` (initial buckets/objects), `0021-s3-multipart-uploads.sql` (multipart parts), `0026`–`0050` (prefixes, search\_v2, race-condition fixes).

We do not import supabase's row-level-security layer; UCAN handles authz.

## DB ↔ MST relationship

- **DB authoritative for runtime.** All S3 queries — `GetObject`, `HeadObject`, `ListObjectsV2`, multipart, IAM checks, lifecycle evaluation — read from Postgres. The MST is never on the read path.
- **MST authoritative for durability and exit.** The DB can be rebuilt from the MST root, with the explicit acknowledgement that service state (policies, in-flight multipart, audit logs) is lost in such a rebuild.
- **Async snapshot pipeline.** Object writes commit synchronously to Postgres. A background process batches DB-changes-since-last-snapshot into a new MST root, packs the changed nodes + new manifests into a CAR, ships it to piri, publishes the index claim, and advances `forge_root_cid`. The existing `Batched` uploader (`pkg/ms3t/uploader/uploader.go:171-277`) is the batching primitive.
- **Bidirectional invariant.** The committed object set — the closure of `(key, version-id, manifest contents, body bytes)` — is bidirectional between DB and MST. Service state is DB-only and lost on rebuild-from-MST.
- **Read-after-write.** Served by the DB. MST staleness is acceptable because the MST is no longer the read path.

## Data plane and byte handling

ms3t is **stateless about object bytes.** Body chunks, manifests, and MST nodes are all content-addressed and live in piri. Postgres holds metadata only. The MST holds canonical references but no payload. ms3t buffers a request's bytes only for the duration of a single PUT — long enough to chunk, hash, and upload them — then drops them.

**Write path** (PUT):

1. Client → ms3t (HTTP body).
2. ms3t chunks the body and uploads each chunk to piri synchronously; chunk CIDs are computed during chunking.
3. ms3t builds an `ObjectManifest`, uploads it to piri, and takes the manifest CID.
4. ms3t commits the metadata row to Postgres, transactional with version-id allocation. The row stores `manifest_cid` and the chunk CIDs.
5. The async snapshot pipeline folds the new manifest CID into the next MST commit.

**Read path** (GET):

1. Client → ms3t.
2. ms3t reads `(bucket, key, version) → manifest_cid` and the chunk CID list from Postgres. No network for metadata.
3. ms3t range-GETs the requested chunks from piri and streams them to the client.

The MST is on neither path. Reads are served entirely from `(Postgres, piri)`; the MST exists for snapshot, exit, and durability — not query. A local block cache is a known optimization for read throughput on hot keys; see Considerations.

## Multipart upload

S3's multipart upload protocol creates a question the single-PUT flow does not: where do part bytes live between `UploadPart` and the eventual `CompleteMultipartUpload` or `AbortMultipartUpload`? Until the upload commits, those bytes may never become part of a committed object. If they are already in piri, an abort produces orphan storage proportional to the upload's size — a much larger stream than per-write MST path-node orphans, and one piri may not even hold a proof of.

Multipart uploads can be very large: AWS raised the per-upload cap to 50 TB in 2025 (up from 5TB). Parallel uploads multiply that by tenant concurrency. Aborts are common — clients drop, retry, or rely on lifecycle policies that auto-abort uploads abandoned for 7 days.

At least two designs are on the table.

### Option 1 — service buffers parts locally; flushes on Complete

`UploadPart` writes bytes to ms3t's local disk; `CompleteMultipartUpload` chunks each part and uploads to piri; `AbortMultipartUpload` deletes the local buffer.

- Avoids piri orphans for aborted uploads entirely. piri storage maps 1:1 to committed objects.
- Service becomes durably stateful for in-flight multipart bytes. The data-plane principle "Service is stateless about object bytes" gets a carve-out for in-flight multipart.
- Disk sizing is unbounded without per-tenant quotas. We cannot support 50 TB single uploads under any realistic service deployment; even 5 TiB is fragile under concurrent load.
- DR boundary expands: services disk is now durable state, co-equal with Postgres for in-flight multipart.
- `CompleteMultipartUpload` is where the bytes flow to piri — long-running for large uploads.

### Option 2 — service streams parts to piri; defers Accept until Complete

piri's blob-allocation protocol has three phases: 

1. **Allocate** (provide size + hash, receive presigned URL)
2. **PUT** (upload bytes; piri returns 201 if checksum matches)
3. **Accept** (client claims commitment). Today piri retains un-Accepted data indefinitely without ever proving custody (i.e. a "bug" in piri).

`UploadPart` chunks the part body, calls Allocate + PUT for each chunk, and persists `(upload_id, part_number, etag, sha256, chunk_cids[])` to Postgres. **Accept is not called.** `CompleteMultipartUpload` issues the deferred Accepts for every chunk in order, builds the manifest, uploads it (Allocate + PUT + Accept), and commits the `objects` row. `AbortMultipartUpload` deletes the multipart rows; the un-Accepted chunks in piri expire on their own.

- ms3t stays stateless about bytes. Data-plane principle holds without carve-out.
- Scales to arbitrary object sizes, bounded only by piri.
- `CompleteMultipartUpload` is fast — Accept calls plus manifest construction.
- `UploadPart` latency is bounded by piri throughput, same shape as a single-PUT chunk.
- **Requires a piri change**: piri must expire un-Accepted data after a TTL. This is an overdue feature in piri regardless of multipart — without it, any failed or abandoned single-PUT also leaves un-Accepted bytes that piri retains forever without proof. The TTL is the un-Accepted analogue of the `assert/expire` mechanism for committed-but-unreachable data discussed in §"Orphan accounting and GC". (Note: Piri does not prove data that has not been accepted, so the coordination here if fairly simple: expire any data older than TTL which hasn't been accepted.)
- The piri operator pays for in-flight upload bytes during the upload window plus the TTL grace period. The customer's bill begins at Accept.

### Recommendation

**Option 2 is the target architecture.** It preserves the service's stateless-about-bytes principle and is the only design that handles 50 TB single uploads. The dependency is the piri un-Accepted-blob TTL, which is overdue regardless of multipart and is a smaller capability than the `assert/expire` work the GC story already requires.

Option 1 is acceptable as an interim if the piri change is far away. Under Option 1 we explicitly accept that we cannot support multi-TB single uploads, and that service's local disk becomes a sized, durable, replicated component. The choice is a function of the piri roadmap and is not made by this RFC.

## Snapshot cadence

**Cadence is a parameter, not a value.** Naming the constraints:

1. Cadence governs orphan accumulation rate. Faster snapshots produce more orphan blocks (see next section), which translates directly into customer storage cost.
2. Cadence governs federation and replication freshness. A snapshot is the unit at which two parties can agree on bucket state by content-addressed root.
3. Cadence is bounded above by the acceptable disaster-recovery window for DB loss. Anything not yet snapshotted is recoverable only from Postgres.
4. Cadence is bounded below by piri round-trip cost. Each snapshot is one CAR upload + one index blob upload + one indexer claim publication.

Possible models: time-based (every N seconds), write-count-based (every N PUTs per bucket), size-based (every N MiB of changes), or a hybrid with adaptive thresholds.

**The decision is deferred** until the orphan/GC mechanism is known. Cadence and GC are co-dependent and must be designed together.

## Orphan accounting and GC

**Every mutation orphans path-nodes.** This is structural to content-addressed Merkle trees: a write rewrites the path from leaf to root, and the prior path-nodes become unreferenced from the new root. ms3t inherits this property.

Per-write orphan profile:

- **Add**: ~`O(depth)` orphan MST nodes (the prior path).
- **Update**: ~`O(depth)` orphan MST nodes. With versioning disabled, the prior manifest and its body chunks are also orphaned. With versioning enabled, the prior manifest stays reachable via `Previous` and the chunks remain live.
- **Delete**: ~`O(depth)` orphan MST nodes. Body data orphans only when the entire version chain is expired.
- **Batch of K writes (one snapshot)**: ~`K · O(depth)` orphan MST nodes, minus the savings from shared-prefix paths between writes that touch nearby keys.

**Liveness model (Git GC).** The live set is the closure of the retained roots — the current `root_cid` plus any retained snapshots used for time-travel or audit. Anything piri holds for the space outside this closure is prunable.

**The customer billing problem.** Orphans accumulate continuously and proportionally to write volume. Customers do not want to pay for orphan state of their buckets. This RFC names the problem and acknowledges it explicitly; the mechanism is deferred.

A related but distinct concern is **multipart-abort cleanup** — bytes uploaded mid-multipart that never become part of a committed object. The design choice for that case lives in §"Multipart upload": under Option 1 the bytes never reach piri; under Option 2 they are un-Accepted in piri and rely on a per-blob TTL rather than reachability-based GC. Both are needed for a complete cost story, but they are different mechanisms over different state.

Prerequisites for actually freeing orphan storage:

1. A Forge-side `assert/expire`-style capability — currently absent.
2. A retention policy that defines how long prior snapshots are kept.
3. A reachability calculator that walks the live set from retained roots.

Upstream hedging work has begun: [FilOzone/filecoin-services#467](https://github.com/FilOzone/filecoin-services/issues/467) proposes a bundled `replacePieces` operation to make per-piece deletion in the Filecoin services smart contracts cheaper and atomic. The issue does not by itself give us `assert/expire`, but it addresses the smart-contract-level cost and atomicity problems any production GC story will hit. The un-Accepted-blob TTL needed for §"Multipart upload" Option 2 is a separate piri-side change.

Mechanism design is out of scope for this RFC and waits on the Forge capability surface to admit it. Readers should leave this section understanding that orphan accumulation is real, structural, and load-bearing on the eventual GC story.

## Fanout and sizing

The MST's role is "be the canonical snapshot," not "serve queries." That changes which fanout trade-offs matter:

- Higher fanout → flatter trees → fewer nodes per write path → smaller per-write CAR.
- Lower fanout → deeper trees → larger per-write CAR but smaller total tree footprint.
- Hash-keyed (current): good balance under non-uniform key distributions; loses prefix-locality, which we no longer need at the MST layer.

Tuning is deferred pending modeling against expected key distributions and write rates. The prototype keeps the current 4-bit hash-keyed fanout.

## Considerations

- **Bucket-tag portability.** Default DB-only; revisit if a portability use case appears.
- **Synchronous body upload.** Body chunks upload to piri before the metadata row commits, so the row never references absent bytes. ms3t holds object bytes only for the duration of a single PUT; payload always lives in piri. PUT latency is bounded by piri throughput.
- **Local read cache.** A block-level cache (in memory and/or on disk) reduces piri round-trips for hot keys and is the natural answer to read-throughput pressure. 
- **Bucket-level state hashing.** The `buckets` row is not in the MST. If federation later requires verifiable bucket-level settings, a small CAS structure could hash them; out of scope here.
- **GC mechanism.** Deferred entirely; awaits two distinct piri/Forge capabilities: (1) `assert/expire`-style expiry of committed-but-unreachable data (path-node and version-chain orphans), and (2) TTL-based expiry of Allocated-but-unaccepted data (failed PUTs and aborted-multipart-Option-2 cleanup). Both are required for a complete cost story.
- **Snapshot cadence.** Deferred; awaits the GC mechanism.
- **MST fanout.** Deferred; awaits workload modeling against real distributions.

## References

- [`001-forge-s3-flat-file-sharding-strategy.md`](https://github.com/fil-one/RFC/pull/2) — Forge S3 Facade sharding strategy
- [storacha/RFC #65](https://github.com/storacha/RFC/pull/65) — Filepack archive format
- [storacha/RFC #66](https://github.com/storacha/RFC/pull/66) — Virtual DAG in Sharded DAG Index
- [supabase/storage](https://github.com/supabase/storage) — schema prior art (`migrations/tenant/0002`, `0021`, `0026`–`0050`)
- [atproto MST](https://github.com/bluesky-social/indigo/tree/main/mst) — origin of the MST fork 
- [versity/versitygw](https://github.com/versity/versitygw) — planned S3 protocol layer
- [FilOzone/filecoin-services#467](https://github.com/FilOzone/filecoin-services/issues/467) — upstream issue: bundled `replacePieces` for cheaper, atomic piece deletion (deletion-story hedge)
