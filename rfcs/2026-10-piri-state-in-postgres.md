# RFC: Keep Piri's State in Postgres

**Status:** Proposal

## Authors

- [Miroslav Bajtoš](https://github.com/bajtos)

## Motivation

Piri and its dependencies should keep all database-like data in Postgres.
Postgres is already required, and Piri refuses to start with anything else. One
database gives one backup, one transaction boundary and one place to look when a
node misbehaves.

Three Piri stores still live in LevelDB under `data_dir`:

- PDP aggregates; this RFC moves this store into Postgres.
- The IPNI publisher store; this RFC split it: metadata goes into Postgres and advertisement blocks go into the blob store.
- The wallet keystore stays an open question.

## Background

### Where Piri uses LevelDB

| Store      | Directory                       | Contents                                                                                     |
| ---------- | ------------------------------- | -------------------------------------------------------------------------------------------- |
| Aggregator | `data_dir/aggregator/datastore` | PDP aggregates: root, ordered pieces, inclusion proofs                                       |
| Publisher  | `data_dir/publisher`            | IPNI advertisement chain: adverts, entry chunks, head, per-context maps                      |
| Keystore   | `data_dir/wallet`               | The owner's secp256k1 wallet key; also opened by `piri wallet import` and `piri wallet list` |

The S3 storage mode keeps all three on local disk.

Piri's dependencies compile LevelDB code into the binary but never open a store
at runtime. Curio reaches Lotus's `node/repo` (LevelDB and badger) only through
`curio/deps`, which Piri does not call. go-f3's certstore is used only by Lotus
snapshot code. go-dsqueue is constructed only by boxo's reprovider, which Piri
does not use. go-ethereum's database packages are not compiled in.
`go-ds-leveldb` will stay in `go.mod` through Lotus, but no dependency writes
LevelDB data.

### Why aggregates are in LevelDB

Nobody chose LevelDB over SQL. The aggregate store was written in November 2024
([storacha/piri#22](https://github.com/storacha/piri/pull/22)), when the node
had no SQL database and every store was LevelDB. SQLite arrived in August 2025
([storacha/piri#192](https://github.com/storacha/piri/pull/192)) for the job
queues only.

When S3 support landed
([storacha/piri#417](https://github.com/storacha/piri/pull/417)), the aggregator
stayed on local disk with the comment "no S3 implementation available". A week
later ([storacha/piri#442](https://github.com/storacha/piri/pull/442)) it moved
into `LocalOnlyModule`, labelled "high-frequency state for PDP aggregation". The
label was accurate then: the same LevelDB also held the aggregator's per-blob
buffer and the batching manager's buffer.
[fil-forge/piri#30](https://github.com/fil-forge/piri/pull/30) moved both
buffers to Postgres tables (`pdp_blob_pipeline`, `pdp_root_submissions`). What
remains in LevelDB is one write per aggregate, by default once per 128 MiB of
accepted data.

No issue or PR in either repository proposed or rejected moving aggregates to SQL.

### Aggregate store problems

**It grows without bound.**

Each aggregate holds one inclusion proof per blob, about 150–250 bytes per blob,
and nothing ever deletes it. At 12 PiB that is 30 GB to 0.8 TB, depending on
blob size.

**Its writes are not transactional.**

`AggregateTask` writes the aggregate to LevelDB inside a Postgres transaction
that LevelDB does not take part in. Only the Postgres half commits atomically.

**A lost record stalls the pipeline.**

If LevelDB loses an aggregate (a restore from an older snapshot, a damaged
disk), `AddRootsTask` fails with "reading aggregate" on every retry and the
blobs in that aggregate never reach the chain. Nothing heals it.

## Design: PDP aggregates

### Schema

Aggregate data moves onto the pipeline rows that already describe each blob. A new migration script adds two columns to `pdp_blob_pipeline`:

- `agg_position integer`: the piece's position in its aggregate.
- `inclusion_proof bytea`: the CBOR-encoded `merkletree.ProofData` for the piece.

Each aggregate piece is one row's `commp`, and the row already carries
`aggregate_root`. The fold matches each piece to its row by blob digest, because
two blobs with the same bytes and different multihashes share a `commp`. An
aggregate is the set of rows `WHERE aggregate_root = $1 ORDER BY agg_position`.
Piece order and each proof's index must survive exactly, because AddRoots checks
that sub-roots are in descending size order and recomputes the root from them.

### Lifecycle

`AggregateTask` sets `aggregate_root`, `aggregated_at`, `agg_position` and
`inclusion_proof` in the transaction that folds the pieces. The aggregate and
the pipeline state now commit together.

`AddRootsTask` is the only reader. It loads each aggregate from these rows once
and passes it to `PieceAcceptor`, which issues the `/pdp/accept` receipts.
`/pdp/info` serves inclusion proofs from the `/pdp/accept` receipts, which are
already in Postgres, and the prover works from Curio's `pdp_data_set_pieces`
tables.

When AddRoots succeeds, the existing retirement transaction deletes the batch's
`pdp_blob_pipeline` and `pdp_root_submissions` rows. The aggregate data goes
with them. The table holds only aggregates between aggregation and AddRoots, so
its size is bounded by the in-flight window.

Piri stops opening `data_dir/aggregator`. The aggregator storage config, its fx
providers and the `aggregator_datastore` name are removed.

### Upgrade

No production node exists. Development and staging nodes may hold aggregates
written to LevelDB but not yet submitted. The database schema migration does not
import them. Instead, it clears `aggregate_root`, `aggregated_at` and
`agg_task_id` on every unretired pipeline row and empties
`pdp_root_submissions`. Clearing `agg_task_id` lets the aggregation task claim
the rows on its next idle poll instead of waiting for a new blob to trigger a
fold. Those blobs fold into new aggregates, which are written to Postgres.

One case loses data: AddRoots landed on chain but the node stopped before the
retirement transaction committed. Those blobs are aggregated again and added to
the data set a second time, so the node proves and is paid for them twice. On
development and staging this is acceptable.

The leftover `data_dir/aggregator` directory is unused after the upgrade.
Operators can delete it.

## Design: IPNI publisher store

The publisher store holds the IPNI advertisement chain. Each published blob adds:

| Record         | Contents                                                                                           | Size   |
| -------------- | -------------------------------------------------------------------------------------------------- | ------ |
| Advert         | dag-cbor: previous advert link, provider, addresses, context ID, metadata, entries link, signature | ~590 B |
| Entries chunk  | dag-cbor `{Entries: [digest]}`; Piri advertises one multihash per advert                           | 46 B   |
| Chunk-link map | context ID → entries-chunk link                                                                    | ~36 B  |
| Metadata map   | context ID → metadata (the location claim CID)                                                     | ~51 B  |

The node also keeps one signed head, rewritten once per publish batch. Nothing
deletes any of these. At 12 PiB of 128 MiB blobs, about 100.7M blobs, the store
holds about 110 GB. Postgres already holds the queue of adverts waiting to be
published (`ipni_pending_adverts`).

An indexer reads the chain by fetching the head, then each advert and its
entries chunk by CID through `GET /ipni/v1/ad/<cid>`, walking back to the last
advert it has seen. Each block is fetched about once per indexer. Piri itself
reads only the head and the two maps, while publishing.

### Where each record goes

- **The head** is one row in Postgres, updated with compare-and-swap.
- **The two maps** move unchanged into one Postgres table,
  `ipni_contexts(context_id, chunk_link, metadata)`, behind go-ipni-tools'
  existing table interface. Publishing behaves as it does today: an unchanged
  context is skipped, a new claim for the same context publishes a metadata
  update, and a release deletes nothing. The table takes about 21–23 GB at
  100.7M blobs. Each node has its own database, so the table does not store the
  provider peer ID.
- **Adverts** go to the blob store, one object per advert keyed by its CID:
  `ipni/<cid>` in the S3 bucket, or a separate flatfs directory `data_dir/ipni`
  on a local volume. 100.7M objects of ~590 B add under 1 TB of raw disk even
  with a 4 KiB allocation unit and erasure-coding overhead, against 12 PiB of
  blobs.
- **Entries chunks are not stored.** The advert links to its entries chunk with
  an identity CID, which carries the 46-byte chunk inside the 50-byte CID. The
  link costs 55 bytes inside the advert instead of 41. storetheindex still
  requests the chunk by CID; Piri answers by decoding the CID, and the indexer's
  hash check passes. Nothing in go-libipni or storetheindex rejects an identity
  link. The cid.contact CAR mirror skips identity blocks and falls back to
  fetching from Piri.

### Publishing a batch

The publish task takes up to 1,000 queued rows and publishes them in one Postgres transaction:

1. Read the maps for every context ID in the batch with one query.
2. Build and sign the adverts.
3. Write the adverts to the blob store in parallel.
4. Write the map changes with one batched upsert, move the head with compare-and-swap, and retire the queue rows.
5. Commit. Announce the new head to IPNI after the commit.

If any step fails, the transaction rolls back, the head stays where it was and
the queue rows are published again later. Adverts already written to the blob
store stay there with nothing linking to them, and nothing deletes them. A
failed batch strands at most 1,000 adverts, about 4 MB with a 4 KiB allocation
unit, which is small enough to accept. Announcing after the commit means an
indexer never hears about a head whose adverts are missing or that a rollback
discarded.

go-ipni-tools gains batch read and write interfaces for the maps and blocks, so
a batch costs a fixed number of SQL statements instead of about four per blob.

### Backups and restore

Only Postgres is backed up; the blob store, sized at petabytes, is not. After a
restore, Postgres is the older side. Every head and map row in it was committed
after its adverts were written, so each points at objects that still exist.
Adverts written after the backup are unreferenced. The restored queue rows
publish again on top of the restored head, and indexers keep what they already
ingested.

A blob store that loses data also loses the blobs it holds, so that case is a
disaster recovery for the whole node rather than for the chain.

### Upgrade

No development or staging node announces to an IPNI node, so the move needs no
migration. Nodes start with an empty chain, and operators can delete
`data_dir/publisher`. Blobs accepted before the upgrade reach the new chain when
Piri gains a backfill.

## Alternatives considered

### Aggregate store alternatives

- **A table with one row per aggregate**, holding the CBOR-encoded `Aggregate`.
  It keeps the current store interface but needs its own delete at retirement,
  and a record grows with the number of blobs in the aggregate (into megabytes
  for many small blobs).
- **A table with one row per piece.** Normalized, but it repeats the key that
  `pdp_blob_pipeline` already holds.
- **Import LevelDB aggregates at startup.** It preserves in-flight work but
  keeps LevelDB code for one more release. No node holds data that needs it.
- **Read from Postgres, fall back to LevelDB for one release.** The same cost as
  an import, spread across the runtime path.

### IPNI store alternatives

- **The whole publisher store in Postgres.** One backend, and every write can
  share a transaction. It adds about 110 GB to Postgres at 100.7M blobs for as
  long as the chain is never trimmed.
- **Entries chunks as blob-store objects.** A second object of 46 bytes per
  blob, which on self-hosted S3 or flatfs occupies a whole allocation unit or
  file.
- **Pack each publish batch into one object**, with a Postgres index from CID
  to object, offset and length. It cuts the object count by up to 1,000 times
  but adds about 15 GB of index, a range-read path and a cache.
- **Render adverts on request**, as Curio does, from a row per advert. It still
  needs about 20 GB of rows, and any change in encoding or addresses would
  change the CIDs indexers already hold.
- **Record "advertised" as a flag on `ucan_acceptances`.** About one byte per
  blob instead of `ipni_contexts`, but it changes what happens on release and
  re-accept. A later migration can derive it from `ipni_contexts`.
- **One advert for many blobs.** It would save about 590 bytes per blob, but an
  advert carries one metadata value for all its entries, and Piri's metadata is
  the per-blob claim. It needs a batch claim type, a new metadata codec and
  indexing-service changes.

## Open Questions

### Keystore

The LevelDB keystore holds the owner's wallet key as plaintext. At startup,
`provisionPDPState` copies it into the Postgres table `eth_keys`, and Curio's
transaction sender and `RegisterProvider` sign with that copy. After
provisioning, the LevelDB keystore serves only a startup existence check and the
`piri wallet` CLI. The key is therefore already in Postgres, which contradicts
the `LocalOnlyModule` comment "private keys must never leave disk".

Options:

- **`eth_keys` as the only store.** `piri init`, `piri wallet import` and `piri
  wallet list` read and write `eth_keys`, and the keystore package goes away. It
  matches where signing already reads the key. Database backups and database
  access then carry the key, so we would want encryption at rest and tight
  access controls.
- **Curio's file keystore** (`curio/lib/keystore`). One file per key, mode 0600,
  refusing keys with looser permissions. It keeps a local source of truth but
  still copies the key into `eth_keys`.
- **Keep LevelDB.** No change, and the plaintext copy in `eth_keys` remains.

## References

- [storacha/piri#22](https://github.com/storacha/piri/pull/22): PDP, introduces the aggregate store
- [storacha/piri#417](https://github.com/storacha/piri/pull/417): S3-compatible storage backend
- [storacha/piri#442](https://github.com/storacha/piri/pull/442): introduces `LocalOnlyModule`
- [fil-forge/piri#30](https://github.com/fil-forge/piri/pull/30): moves the aggregation pipeline onto harmonytask
- [fil-forge/piri#183](https://github.com/fil-forge/piri/pull/183): moves UCAN stores to Postgres
- [FIL-1577](https://linear.app/filecoin-foundation/issue/FIL-1577): Piri should store IPNI-related data in Postgres and the blob store
