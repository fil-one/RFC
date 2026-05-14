# RFC: Bucket metadata — canonical MST and operational database

**Author**: Forrest (@frrist)

**Date**: 2026-04-30 (revised 2026-05-04)

**Status**: Draft

## Abstract

A Storacha S3 bucket is "Web3" only if a customer can walk away with their bucket — a content-addressed, self-describing artifact a recipient can reconstruct without Storacha-specific code. The bucket is therefore a Merkle Search Tree (MST), keyed by S3 object key with leaves pointing at `ObjectManifest` CIDs. The MST root names the whole bucket state at a point in time.

This RFC describes how that MST is materialized at runtime. The **recommended design** — and the one in `pkg/ms3t/` today — treats the MST as the source of truth for both reads and durability. Every S3 mutation lands synchronously in a local LSM-style segment log (CAR + per-batch fsync sidecar) and is async-shipped to Forge as a CAR + indexer claim once the segment seals. Postgres holds only a per-bucket `root_cid` pointer (CAS-advanced per op) and segment lifecycle metadata; the relational store does not participate in the read path.

An **alternative** — Postgres-authoritative, with the MST as a periodic async snapshot of an authoritative relational store — was considered before the prototype settled and remains a coherent design in workloads where prefix-listing throughput, IAM evaluation, and relational reporting dominate. It is documented as **Alternative considered: Postgres-authoritative** so the trade-off is preserved alongside what was actually built.

## Motivation

A Storacha S3 bucket is "Web3" only if a customer can walk away with their bucket. The falsifiable test: hand them a CAR file and they reconstruct an interoperable bucket on any IPFS-aware platform. That requires a content-addressed, self-describing representation of bucket state — which the MST provides.

The original draft of this RFC argued the MST was a poor fit for the read path and that S3 prefix listing, range reads, multipart, IAM checks, and lifecycle should be served from a relational store, with the MST relegated to an async snapshot for credible exit and federation. That argument led to the **Postgres-authoritative** alternative documented at the end of this RFC; supabase/storage was the prior art being adopted.

The implementation in `pkg/ms3t/` chose differently. Keeping the MST on the read path turned out to be acceptable when fronted by a local LSM tier — sealed-segment lookup absorbs hot reads, the open segment serves read-after-write directly out of an in-memory index, and only cold reads fall through to Forge. In exchange, the implementation gets one source of truth instead of two: no DB/MST drift, no batched-snapshot lag, no dual-write coordination. The MST root advances atomically with each S3 op via `CASRoot` in Postgres, and every committed write is immediately a credible-exit artifact. The MST is **always live** rather than a periodic projection.

But local LSM tiers do not make Forge optional. The MST is what makes the body chunks in piri *reachable from Forge*: without a continuously-maintained MST root shipped to Forge, the bytes are orphaned from the indexing-service's point of view. The MST root shipped to Forge is also the only verifiable identity for "the state of this bucket" — required for federation, replication, snapshots, and credible exit.

This RFC defines how that authority surface is wired in the recommended design, and contrasts it with the Postgres-authoritative alternative.

## What is an MST

A **Merkle Search Tree** is a content-addressed key-value tree. Every node is serialized, hashed, and identified by its CID; the root CID names the whole tree state at a point in time. Mutation is copy-on-write — any change rewrites the path of nodes from the affected leaf to the root, producing a new root CID. Unchanged subtrees keep their CIDs, so successive versions share structure.

For this RFC, MST keys are S3 object keys and values are CIDs of `ObjectManifest` blocks. The "current state of a bucket" is therefore a single CID — the MST root — from which every committed object is reachable.

The implementation in `pkg/ms3t/mst/` is forked from atproto, where the structure represents social-graph repos. The implications of that origin for S3 are discussed in §"Why MST" and §"Fanout and sizing".

## Why MST

The MST plays five roles, in priority order:

1. **Credible exit.** A CAR file containing the MST root, every reachable node, every reachable `ObjectManifest`, and every reachable body chunk is a complete, portable bucket. A recipient with no Storacha-specific code can reconstruct it.
2. **Forge-native durability.** The MST root is the discoverable, indexer-resolvable handle from which all bucket bytes are reachable. Without continuous MST commits shipped to Forge, body chunks in piri are orphaned from Forge's perspective. The local LSM holds in-flight state; the MST root is the durable artifact in Forge.
3. **Federation and replication.** A content-addressed root makes "sync to root R" verifiable across operator boundaries. Postgres logical replication does not.
4. **Free incremental snapshots.** Every commit is a snapshot. Structural sharing makes N retained snapshots ≈ 1 in storage cost. Under the recommended design, where every S3 op produces a new root, this property compounds: history is dense and free.
5. **Tamper-evident history.** Retained roots form a verifiable commitment chain.

The current MST in `pkg/ms3t/mst/` is forked from atproto's repo MST: 4-bit fanout, hash-keyed via `sha256(key)` leading-zero count (`pkg/ms3t/mst/mst_util.go:20-49`). atproto's design point is bounded social-graph repos with no prefix-listing requirement. S3 buckets violate all of those: keys can be 1KB, deeply hierarchical, prefix-listing is a first-class operation, and buckets can grow unbounded.

The recommended design lives with these properties by absorbing prefix-listing latency in the local LSM tier (the open segment's in-memory index + sealed segments on local disk handle the great majority of block fetches; only cold blocks reach Forge). Fanout becomes a snapshot-efficiency knob (size of diff per write batch), not a query-efficiency knob. Tuning is deferred — see §"Fanout and sizing".

## Canonical state vs service state

The dividing rule is the **credible-exit test**: state belongs in the MST if a customer would want it on exit to another platform; otherwise it is service state and lives only in the database. The rule applies under both designs.

| Feature | MST | DB | Notes |
|---|---|---|---|
| Object body content (chunks) | ✓ | refs only | bytes always in piri; DB and MST hold CIDs, not bytes |
| Object manifest (Content-Type, user-meta, ETag, size, timestamps) | ✓ | ✓ mirror (alt only) | intrinsic to "what the object is" |
| Object tags | ✓ | ✓ mirror (alt only) | per-object user metadata; travels |
| Object versioning history | ✓ | ✓ mirror (alt only) | versions are user data |
| Multipart upload state (in-flight) | | ✓ | service-only; on `CompleteMultipartUpload` the assembled object enters the MST |
| Bucket policy / IAM / ACL | | ✓ | service-enforced; no enforcer after exit |
| Bucket CORS, lifecycle, replication, notifications, website | | ✓ | service features over the bucket |
| Object Lock, retention, legal hold | | ✓ | service-enforced immutability contract |
| Bucket tags | | ✓ | operational unit metadata, not content |
| `bucket → root_cid` pointer | | ✓ | operational cursor |
| Owner mapping, audit logs, metrics | | ✓ | platform state |

Under the **recommended design**, the "✓ mirror" columns above are *not* populated — the MST is the only home for object metadata, and the relational store does not mirror it. Under the **alternative**, they are.

Bucket tags are a borderline call. They describe the operational unit (`cost-center=engineering`) rather than user content; on exit, a recipient can re-tag the destination bucket. This RFC defaults them to DB-only and revisits if a portability use case appears.

## Recommended design — MST-authoritative LSM

The implementation in `pkg/ms3t/` (described in detail in [`pkg/ms3t/architectural.md`](https://github.com/storacha/sprue/blob/main/pkg/ms3t/architectural.md)) treats the MST as canonical state and uses an LSM-style local log to absorb the latency and ordering work that a relational tier would otherwise do.

### Authority model

- **MST is authoritative** for both reads and durability. The bucket's current state is exactly its `root_cid`.
- **Postgres is a coordination layer.** It holds the per-bucket `root_cid` pointer (CAS-advanced per op for serializability across concurrent S3 requests) and the lifecycle of LSM segments. It does **not** participate in the read path and does **not** mirror object metadata. If Postgres is wiped, the segment files on disk plus the indexer + piri are sufficient to rehydrate it.
- **The local LSM log is the synchronous durability boundary.** A PUT returns 200 only after the segment's CAR + .ops sidecar are fsynced and Postgres has CAS-advanced the bucket root.
- **Forge is the asynchronous durability boundary.** Sealed segments are shipped to Forge by a background flusher; `forge_root_cid` is advanced atomically with the segment's `flushed` state transition.

### Data model

The leaf value is the CID of an `ObjectManifest`. The implementation's manifest at [`pkg/ms3t/bucket/manifest.go:10-43`](https://github.com/storacha/sprue/blob/main/pkg/ms3t/bucket/manifest.go) is intentionally lean:

```go
type ObjectManifest struct {
    Key         string `cborgen:"k"`
    ContentType string `cborgen:"ct"`
    Created     int64  `cborgen:"t"`
    Body        Body   `cborgen:"b"`
}

type Body struct {
    Size    int64   `cborgen:"s"`
    SHA256  []byte  `cborgen:"h"`     // ETag source today
    Content cid.Cid `cborgen:"c"`     // body-DAG root
    Format  string  `cborgen:"f"`     // routes to BodyCodec
}

const FormatFixed = "fixed-v1"

type FixedChunkerIndex struct {
    ChunkSize int64     `cborgen:"cs"`
    Chunks    []cid.Cid `cborgen:"c"`
}
```

Body framing is polymorphic via `Body.Format`. The only codec today is `FormatFixed` — flat 1 MiB raw blocks indexed by a `FixedChunkerIndex`. The `BodyCodec` seam (`pkg/ms3t/bucket/chunker.go`) keeps adding a new codec to a new constant + new implementation; the manifest shape stays stable. The target end state for `Body.Content` under [`001-forge-s3-flat-file-sharding-strategy.md`](https://github.com/fil-one/RFC/pull/2) is a UnixFS File root over 256 MB Filepack shards, with an SDI v0.2 inlined for one-roundtrip retrieval.

**Not yet modeled.** Versioning chains, deletion markers, S3 user metadata (`x-amz-meta-*`), per-object tags, and `Modified` (S3 Last-Modified) are absent from the implemented manifest. They are extensions, not removals — the original RFC's proposed shape (`Previous` chain, `DeleteMarker`, `UserMetadata`, `Tags`) is the target end state. Their absence is tracked by failing smoke cases (`PutObject_with_metadata`, version-related cases) in `pkg/ms3t/testing/smoke_test.go`.

The MST node shape (`NodeData`, `TreeEntry` in `pkg/ms3t/mst/`) is the atproto fork with relaxed key validation; key-prefix compression and `Tree`/`Left` subtree pointers are unchanged.

**Exit format.** A single CAR file containing the MST root + every reachable MST node + every reachable `ObjectManifest` + every reachable body chunk is a complete portable bucket. No service state is required to read it.

### Postgres schema

The recommended-design schema is intentionally tiny — Postgres is a coordinator, not a query store. From `pkg/ms3t/migrations/sql/`:

```sql
CREATE TABLE ms3t.buckets (
    name           TEXT   PRIMARY KEY,
    root_cid       BYTEA,           -- current MST root, NULL for empty bucket
    forge_root_cid BYTEA,           -- last MST root whose blocks are durable in Forge
    created_at     BIGINT NOT NULL
);

CREATE TABLE ms3t.segments (
    seq         BIGINT PRIMARY KEY,
    state       TEXT   NOT NULL CHECK (state IN ('open','sealed','flushed')),
    sealed_at   BIGINT,
    flushed_at  BIGINT,
    size_bytes  BIGINT NOT NULL DEFAULT 0,
    car_sha256  BYTEA
);

CREATE TABLE ms3t.segment_op_roots (
    seq        BIGINT NOT NULL REFERENCES ms3t.segments(seq) ON DELETE CASCADE,
    seq_within INT    NOT NULL,
    bucket     TEXT   NOT NULL,
    root_cid   BYTEA  NOT NULL,
    PRIMARY KEY (seq, seq_within)
);
CREATE INDEX segment_op_roots_bucket_seq_idx ON ms3t.segment_op_roots (bucket, seq);

CREATE SEQUENCE ms3t.segment_seq;
```

`forge_root_cid` is the per-bucket high-water mark of "what's durably in Forge." When a flush succeeds, the flusher advances `forge_root_cid` for every op-root the segment carried, in the same Postgres transaction that flips the segment's state to `flushed`. Anything reachable from `root_cid` but not from `forge_root_cid` is durable on local disk only.

Service-feature tables (policies, lifecycle, multipart upload state, IAM, lock) are additive extensions on this schema. They are **service state**, distinct from canonical bucket state, and they do not change the authority model — they live only in Postgres because there is nothing in them a customer wants on credible exit.

### Storage tiers

```
HOT    open segment       seg-NNN.car  + .ops sidecar  (per-batch fsync of both)
WARM   sealed segments    + .idx sidecar (atomic tmp+rename)
COLD   shipped to Forge   CAR + SDI + indexer claim;
                          forge_root_cid advanced atomically with state flip
```

- `.car` — CAR v1, blocks appended via `cars.WriteBlocksAt`. Per-batch fsync.
- `.ops` — append-only sidecar of `[bucket, root]` CBOR records, length-prefixed. One record per `AppendBatch` (one S3 op).
- `.idx` — written atomically at seal time. JSON: `{seq, size_bytes, sha256_hex, sealed_at, blocks[], op_roots[]}`. Source of truth for sealed segments.

Default seal triggers ([`pkg/ms3t/logstore/config.go:62-75`](https://github.com/storacha/sprue/blob/main/pkg/ms3t/logstore/config.go)): seal at 64 MiB or 5 s, whichever first. `Retain` defaults to 6 — that many flushed segments stay on disk as a local read tier; older ones are unlinked.

### Write path

Per S3 mutation:

1. `bucketop.Coordinator.Begin(bucket)` — clone the bucket name (defends against fiber's recycled request buffer), acquire the per-bucket lock, snapshot the current root from Postgres.
2. `BodyCodec.Chunk(ctx, tx, body)` — write body chunks + `FixedChunkerIndex` through the per-tx staging buffer (which feeds the segment log on Commit).
3. `tx.Put(manifest)` — write the `ObjectManifest`.
4. MST mutate (`Add` / `Update` / `Delete` + `GetPointer`) — serialize new MST nodes through the same staging buffer, return the new root CID.
5. `tx.Commit(newRoot)`:
   - `staging.Commit` → `log.AppendBatch(blocks, OpRoot{bucket, root})`. The segment fsyncs both `.car` and `.ops` before returning.
   - `reg.CASRoot(bucket, expect, next)` advances the bucket root in Postgres.
   - Release the per-bucket lock.
6. Return 200 to the client.

ms3t holds object bytes only for the duration of a single PUT — long enough to chunk, hash, and stage them — then drops them. The service is stateless about object bytes; payloads always live in (segment files, Forge).

### Read path

The read path is `blockstore.Layered` — a composite tier:

1. **Open segment's in-memory index** — CIDs of blocks just appended in this process.
2. **Sealed segments on local disk**, newest-first by `seq`.
3. **Forge** — only on local miss. `blockstore.Forge` queries the indexer for the block's `(CAR multihash, offset, length)`, self-issues a scoped retrieval delegation, and does a ranged GET against piri.

Read-after-write is served from the open segment's in-memory index — no Forge round-trip. Hot keys served from sealed segments. Cold keys fall through to Forge once.

Listings (`ListObjectsV2`) walk the MST via `s3frontend.listWalk`. There is no precomputed prefix index; prefix-listing latency is bounded by MST traversal over the Layered tier. Improving this is in scope for future work — most likely a derived prefix table populated alongside the bucket-root advance — but the recommended design does **not** require it for correctness.

### Flush path

Background goroutine in `logstore.Store`:

1. Pick a sealed segment off the queue.
2. Build a `uploader.CARSource` from segment metadata (`{Path, Size, SHA256, Positions}` — every field already on the segment, no rescan).
3. `uploader.Forge.SubmitCAR`: allocate + PUT + Accept the data CAR via piri (routing-selected); build a `ShardedDagIndexView` from the segment's block positions; allocate + PUT + Accept the index blob; self-issue a `space/content/retrieve` delegation scoped to the index blob; publish the index claim against the indexing-service.
4. `meta.MarkSegmentFlushed` in one Postgres transaction — flips state to `flushed`, writes `flushed_at`, advances `forge_root_cid` for every op-root the segment carried.
5. Retention: if more than `Retain` flushed segments are on disk, retire the oldest.

### Recovery on startup

`logstore.Open` reconciles `<DataDir>/segments/` against `Meta.ListUnflushedSegments` before accepting writes:

- **File + DB open** → rebuild as open via CAR scan + ops replay; force-seal at startup. We never resume an open segment from a previous process.
- **File + DB sealed** → load from `.idx`, re-enqueue for flush.
- **File + .idx, no DB row** → rehydrate the DB row; the `.idx` is authoritative for sealed state.
- **File only (torn seal)** → rebuild as open, seed DB, force-seal.
- **DB row, no file** → log error, delete the DB row.

The on-disk segment + sidecars are the source of truth at recovery; Postgres is rebuildable from them.

### Identity and Forge wiring

ms3t owns an ed25519 keypair persisted at `<DataDir>/space.key` — this is the **space**, the root UCAN authority. ms3t self-issues `space/content/retrieve` delegations to sprue's identity, which is the audience for retrievals (sprue talks to piri "as itself"). One ms3t ↔ one space; multi-tenancy is out of scope today.

### Trade-offs

**Pros**
- One source of truth — no DB/MST drift, no dual-write coordination, no batched-snapshot lag.
- The MST is always live; every committed write is immediately a credible-exit artifact.
- Per-op fsync gives durability without putting Postgres on the data path.
- DR boundary on local disk is exactly the segment files; on Forge it's the most recent flushed segment.

**Cons**
- `ListObjectsV2` traverses the MST through `Layered`; prefix-listing throughput is bounded by block-fetch cost, not by an index scan. Acceptable today; improvable via a derived prefix table.
- Service features that benefit from relational queries (lifecycle evaluation, IAM evaluation over many objects, audit reporting) are MST-walk-bound today.
- Multi-writer scaling needs cross-process coordination beyond the in-process per-bucket lock — out of scope today.
- `OpStaging` buffers an entire op's blocks in memory until Commit; multi-GB PUTs bound peak memory at ≈ payload size. Tracked TODO at `pkg/ms3t/blockstore/staging.go`; a file-backed staging swap caps per-tx footprint at one chunk + index.

## Multipart upload

S3's multipart upload protocol creates a question the single-PUT flow does not: where do part bytes live between `UploadPart` and the eventual `CompleteMultipartUpload` or `AbortMultipartUpload`? Until the upload commits, those bytes may never become part of a committed object. If they are already in piri, an abort produces orphan storage proportional to the upload's size — a much larger stream than per-write MST path-node orphans, and one piri may not even hold a proof of.

Multipart uploads can be very large: AWS raised the per-upload cap to 50 TB in 2025 (up from 5TB). Parallel uploads multiply that by tenant concurrency. Aborts are common — clients drop, retry, or rely on lifecycle policies that auto-abort uploads abandoned for 7 days.

Multipart upload is unimplemented in `pkg/ms3t/` today; the design discussion below applies under either authority model. Under the **recommended design**, multipart-upload-state tables (`s3_multipart_uploads`, `s3_multipart_uploads_parts`) are an additive extension to Postgres — they are **service state** by the §"Canonical state vs service state" rule and do not change the authority model. On `CompleteMultipartUpload`, the assembled object enters the MST in exactly the same way a single-PUT object does. Under the **alternative**, the same tables sit alongside the rest of the relational store.

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

`UploadPart` chunks the part body, calls Allocate + PUT for each chunk, and persists `(upload_id, part_number, etag, sha256, chunk_cids[])` to Postgres. **Accept is not called.** `CompleteMultipartUpload` issues the deferred Accepts for every chunk in order, builds the manifest, uploads it (Allocate + PUT + Accept), and commits the bucket root advance (recommended design: via the same `bucketop.Tx` an ordinary PUT uses). `AbortMultipartUpload` deletes the multipart rows; the un-Accepted chunks in piri expire on their own.

- ms3t stays stateless about bytes. Data-plane principle holds without carve-out.
- Scales to arbitrary object sizes, bounded only by piri.
- `CompleteMultipartUpload` is fast — Accept calls plus manifest construction.
- `UploadPart` latency is bounded by piri throughput, same shape as a single-PUT chunk.
- **Requires a piri change**: piri must expire un-Accepted data after a TTL. This is an overdue feature in piri regardless of multipart — without it, any failed or abandoned single-PUT also leaves un-Accepted bytes that piri retains forever without proof. The TTL is the un-Accepted analogue of the `assert/expire` mechanism for committed-but-unreachable data discussed in §"Orphan accounting and GC". (Note: Piri does not prove data that has not been accepted, so the coordination here is fairly simple: expire any data older than TTL which hasn't been accepted.)
- The piri operator pays for in-flight upload bytes during the upload window plus the TTL grace period. The customer's bill begins at Accept.

### Recommendation

**Option 2 is the target architecture.** It preserves the service's stateless-about-bytes principle and is the only design that handles 50 TB single uploads. The dependency is the piri un-Accepted-blob TTL, which is overdue regardless of multipart and is a smaller capability than the `assert/expire` work the GC story already requires.

Option 1 is acceptable as an interim if the piri change is far away. Under Option 1 we explicitly accept that we cannot support multi-TB single uploads, and that service's local disk becomes a sized, durable, replicated component. The choice is a function of the piri roadmap and is not made by this RFC.

## Cadence: segment seal and Forge flush

Under the recommended design, every committed S3 op produces a new MST root — there is no separate snapshot cadence at the MST layer. The relevant cadence is **segment seal + flush to Forge**. Naming the constraints:

1. Seal cadence governs how often Forge receives new state. Faster seals → finer-grained shipping → smaller per-seal CAR but more indexer claims and Forge round-trips. Slower seals → larger CARs but worse Forge-side freshness and a longer tail of unflushed-on-Forge state.
2. Seal cadence governs federation and replication freshness. A flushed segment is the unit at which two parties can agree on bucket state by content-addressed root over Forge.
3. Seal cadence bounds the disaster-recovery window for unflushed segments. Anything in the open segment is recoverable only from local disk; anything in a sealed-but-not-flushed segment is recoverable from local disk until the flush succeeds.
4. Per-seal cost is one CAR upload + one index blob upload + one indexer claim publication.

The implementation chose a hybrid model: seal on bytes (`SealBytes` default 64 MiB) or on age (`SealAge` default 5 s), whichever first. Both are tunable via `ServerConfig`. Whether those defaults stay is a workload-tuning question that depends on the orphan/GC mechanism — see next section.

Under the **alternative** (Postgres-authoritative), cadence is what the original RFC called *snapshot cadence*: how often the async pipeline batches DB-changes-since-last-snapshot into a new MST root and ships the resulting CAR to Forge. The same four constraints apply, with the addition of (5) DB/MST drift acceptable to the operator — a parameter the recommended design eliminates by construction.

## Orphan accounting and GC

**Every mutation orphans path-nodes.** This is structural to content-addressed Merkle trees: a write rewrites the path from leaf to root, and the prior path-nodes become unreferenced from the new root. ms3t inherits this property under either design.

Per-write orphan profile:

- **Add**: ~`O(depth)` orphan MST nodes (the prior path).
- **Update**: ~`O(depth)` orphan MST nodes. With versioning disabled, the prior manifest and its body chunks are also orphaned. With versioning enabled, the prior manifest stays reachable via `Previous` and the chunks remain live.
- **Delete**: ~`O(depth)` orphan MST nodes. Body data orphans only when the entire version chain is expired.
- **Batch of K writes (one snapshot/seal)**: ~`K · O(depth)` orphan MST nodes, minus the savings from shared-prefix paths between writes that touch nearby keys.

Under the **recommended design**, orphan production is per-op (every commit is a new root); there is no batching window that amortizes path-node duplication across writes that touch nearby keys. The segment-seal cadence governs how that orphan stream packages into Forge CARs but not whether it occurs. Under the **alternative**, the snapshot batching window does provide some cross-write amortization at the cost of staleness.

**Liveness model (Git GC).** The live set is the closure of the retained roots — the current `root_cid` plus any retained snapshots used for time-travel or audit. Anything piri holds for the space outside this closure is prunable.

**The customer billing problem.** Orphans accumulate continuously and proportionally to write volume. Customers do not want to pay for orphan state of their buckets. This RFC names the problem and acknowledges it explicitly; the mechanism is deferred.

A related but distinct concern is **multipart-abort cleanup** — bytes uploaded mid-multipart that never become part of a committed object. The design choice for that case lives in §"Multipart upload": under Option 1 the bytes never reach piri; under Option 2 they are un-Accepted in piri and rely on a per-blob TTL rather than reachability-based GC. Both are needed for a complete cost story, but they are different mechanisms over different state.

Prerequisites for actually freeing orphan storage:

1. A Forge-side `assert/expire`-style capability — currently absent.
2. A retention policy that defines how long prior snapshots are kept.
3. A reachability calculator that walks the live set from retained roots.

Upstream hedging work has begun: [FilOzone/filecoin-services#467](https://github.com/FilOzone/filecoin-services/issues/467) proposes a bundled `replacePieces` operation to make per-piece deletion in the Filecoin services smart contracts cheaper and atomic. The issue does not by itself give us `assert/expire`, but it addresses the smart-contract-level cost and atomicity problems any production GC story will hit. The un-Accepted-blob TTL needed for §"Multipart upload" Option 2 is a separate piri-side change.

Mechanism design is out of scope for this RFC and waits on the Forge capability surface to admit it. Readers should leave this section understanding that orphan accumulation is real, structural, and load-bearing on the eventual GC story under either authority model.

## Fanout and sizing

The MST's role is "be the canonical content-addressed snapshot." Under the recommended design it also serves reads, but the LSM tier absorbs the latency-sensitive part of that work. That changes which fanout trade-offs matter:

- Higher fanout → flatter trees → fewer nodes per write path → smaller per-write CAR.
- Lower fanout → deeper trees → larger per-write CAR but smaller total tree footprint.
- Hash-keyed (current): good balance under non-uniform key distributions; loses prefix-locality, which we no longer need at the MST layer because prefix-listing semantics are computed at walk time over the Layered tier (today) or via a derived prefix index (future).

Tuning is deferred pending modeling against expected key distributions and write rates. The prototype keeps the current 4-bit hash-keyed fanout.

## Alternative considered — Postgres-authoritative

The original draft of this RFC proposed a **Postgres-authoritative** design: a relational store inspired by [supabase/storage](https://github.com/supabase/storage) serves all S3 reads, and the MST is rebuilt asynchronously from DB-changes-since-last-snapshot. It is documented here so the trade-off is visible alongside what was actually built.

### Shape

- **Postgres is authoritative for runtime.** All S3 queries — `GetObject`, `HeadObject`, `ListObjectsV2`, multipart, IAM checks, lifecycle evaluation — read from Postgres. The MST is never on the read path.
- **MST is authoritative for durability and exit.** The DB can be rebuilt from the MST root, with the explicit acknowledgement that service state (policies, in-flight multipart, audit logs) is lost in such a rebuild.
- **Async snapshot pipeline.** Object writes commit synchronously to Postgres. A background process batches DB-changes-since-last-snapshot into a new MST root, packs the changed nodes + new manifests into a CAR, ships it to piri, publishes the index claim, and advances `forge_root_cid`.
- **Read-after-write.** Served by the DB. MST staleness is acceptable because the MST is no longer the read path.

The DB schema mirrors the supabase/storage prior art: `buckets`, `objects` (one row per version), `prefixes` (materialized folder hierarchy with triggers, powering `ListObjectsV2` prefix/delimiter semantics), `s3_multipart_uploads` and `s3_multipart_uploads_parts`, plus service-feature tables (`bucket_policies`, `bucket_lifecycle_rules`, `bucket_cors_rules`, `bucket_notifications`, `object_locks`, `object_retentions`, `object_legal_holds`) and a `snapshots` table for retained MST roots. Migrations from supabase to study, in order: `0002-storage-schema.sql` (initial buckets/objects), `0021-s3-multipart-uploads.sql` (multipart parts), `0026`–`0050` (prefixes, search\_v2, race-condition fixes). UCAN handles authz; supabase's row-level-security layer is not imported.

The proposed `ObjectManifest` under this alternative is richer — versioning chain via `Previous`, `DeleteMarker`, `UserMetadata`, `Tags`, `Modified`. A delete is a new manifest with `DeleteMarker: true` and `Previous` linking the prior manifest. The MST always uses the versioned shape; a bucket with versioning disabled simply means the service never surfaces anything past `current`.

### Why this wasn't chosen

The recommended design removes a class of bugs (DB/MST drift, dual-write coordination, snapshot lag) at the cost of weaker query power for prefix listing and relational reporting. The supabase schema can be re-introduced as a **derived index** — a read-only relational projection rebuildable from the MST — if and when query workloads demand it. That keeps the MST as the single source of truth while admitting relational query power for the workloads that need it.

The argument for the alternative remains valid for any future ms3t deployment where prefix-listing throughput, IAM evaluation over millions of objects, lifecycle rule fan-out, or relational reporting dominates the workload — and where the operator is willing to pay the dual-write coordination cost in exchange. This RFC does not foreclose that path; it documents that the implementation chose differently for now.

## Considerations

- **Bucket-tag portability.** Default DB-only; revisit if a portability use case appears.
- **Synchronous body upload.** Body chunks land in the segment log before AppendBatch returns and before Postgres CAS-advances the bucket root, so the published root never references absent bytes. ms3t holds object bytes only for the duration of a single PUT; payload always lives in the segment files (until shipped) and Forge (after). PUT latency is bounded by local fsync + Postgres CAS; not by piri throughput.
- **In-memory block cache.** The Layered tier already absorbs hot reads from local disk; an explicit in-memory cache is a further optimization for ultra-hot keys and is orthogonal to authority model. Worth measuring before building.
- **GC mechanism.** Deferred entirely; awaits two distinct piri/Forge capabilities: (1) `assert/expire`-style expiry of committed-but-unreachable data (path-node and version-chain orphans), and (2) TTL-based expiry of Allocated-but-unaccepted data (failed PUTs and aborted-multipart-Option-2 cleanup). Both are required for a complete cost story under either authority model.
- **Segment-flush cadence.** Defaults are 64 MiB / 5 s; the right values depend on workload and on the GC mechanism, since faster flushes increase the orphan stream rate to Forge.
- **MST fanout.** Deferred; awaits workload modeling against real distributions.
- **Derived prefix index.** A relational projection of `(bucket, prefix, key, manifest_cid)` populated alongside `CASRoot` would keep MST authority while restoring O(log) prefix listing; designing it is future work.

## References

- [`pkg/ms3t/architectural.md`](https://github.com/storacha/sprue/blob/main/pkg/ms3t/architectural.md) — implementation source-of-truth for the recommended design
- [`pkg/ms3t/bucket/manifest.go`](https://github.com/storacha/sprue/blob/main/pkg/ms3t/bucket/manifest.go) — current `ObjectManifest`/`Body`/`FixedChunkerIndex`
- [`pkg/ms3t/migrations/sql/`](https://github.com/storacha/sprue/blob/main/pkg/ms3t/migrations/sql) — `ms3t.buckets`, `ms3t.segments`, `ms3t.segment_op_roots` schema
- [`pkg/ms3t/logstore/`](https://github.com/storacha/sprue/blob/main/pkg/ms3t/logstore) — LSM-style segment log
- [`pkg/ms3t/blockstore/`](https://github.com/storacha/sprue/blob/main/pkg/ms3t/blockstore) — `Layered`, `Forge`, `OpStaging`
- [`001-forge-s3-flat-file-sharding-strategy.md`](https://github.com/fil-one/RFC/pull/2) — Forge S3 facade sharding strategy
- [storacha/RFC #65](https://github.com/storacha/RFC/pull/65) — Filepack archive format
- [storacha/RFC #66](https://github.com/storacha/RFC/pull/66) — Virtual DAG in Sharded DAG Index
- [supabase/storage](https://github.com/supabase/storage) — schema prior art for the alternative (`migrations/tenant/0002`, `0021`, `0026`–`0050`)
- [atproto MST](https://github.com/bluesky-social/indigo/tree/main/mst) — origin of the MST fork
- [versity/versitygw](https://github.com/versity/versitygw) — S3 protocol layer
- [FilOzone/filecoin-services#467](https://github.com/FilOzone/filecoin-services/issues/467) — upstream issue: bundled `replacePieces` for cheaper, atomic piece deletion (deletion-story hedge)
