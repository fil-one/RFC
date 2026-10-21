# RFC: Keep Piri's State in Postgres

**Status:** Proposal

## Authors

- [Miroslav Bajtoš](https://github.com/bajtos)

## Motivation

Piri and its dependencies should keep all database-like data in Postgres. Postgres is already required: the PDP pipeline runs on Curio's harmonytask, whose task claims use `FOR UPDATE SKIP LOCKED`, and Piri refuses to start the pipeline on anything else. One database gives one backup, one transaction boundary and one place to look when a node misbehaves.

Three Piri stores still live in LevelDB under `data_dir`. This RFC moves two of them. PDP aggregates go into Postgres. The IPNI publisher store splits: its metadata goes into Postgres and its advertisement blocks go into the blob store. The third store, the wallet keystore, stays an open question.

The aggregate store has three problems today:

- **It grows without bound.** Each aggregate holds one inclusion proof per blob, about 150–250 bytes per blob, and nothing ever deletes it. At 12 PiB that is 30 GB to 0.8 TB, depending on blob size.
- **Its writes are not transactional.** `AggregateTask` writes the aggregate to LevelDB inside a Postgres transaction that LevelDB does not take part in. Only the Postgres half commits atomically.
- **A lost record stalls the pipeline.** If LevelDB loses an aggregate (a restore from an older snapshot, a damaged disk), `AddRootsTask` fails with "reading aggregate" on every retry and the blobs in that aggregate never reach the chain. Nothing heals it.

## Background

### Why aggregates are in LevelDB

Nobody chose LevelDB over SQL. The aggregate store was written in November 2024 ([storacha/piri#22](https://github.com/storacha/piri/pull/22)), when the node had no SQL database and every store was LevelDB. SQLite arrived in August 2025 ([storacha/piri#192](https://github.com/storacha/piri/pull/192)) for the job queues only.

When S3 support landed ([storacha/piri#417](https://github.com/storacha/piri/pull/417)), the aggregator stayed on local disk with the comment "no S3 implementation available". A week later ([storacha/piri#442](https://github.com/storacha/piri/pull/442)) it moved into `LocalOnlyModule`, labelled "high-frequency state for PDP aggregation". The label was accurate then: the same LevelDB also held the aggregator's per-blob buffer and the batching manager's buffer. [fil-forge/piri#30](https://github.com/fil-forge/piri/pull/30) moved both buffers to Postgres tables (`pdp_blob_pipeline`, `pdp_root_submissions`). What remains in LevelDB is one write per aggregate, by default once per 128 MiB of accepted data.

No issue or PR in either repository proposed or rejected moving aggregates to SQL.

### Where Piri uses LevelDB

| Store | Directory | Contents |
|---|---|---|
| Aggregator | `data_dir/aggregator/datastore` | PDP aggregates: root, ordered pieces, inclusion proofs |
| Publisher | `data_dir/publisher` | IPNI advertisement chain: adverts, entry chunks, head, per-context maps |
| Keystore | `data_dir/wallet` | The owner's secp256k1 wallet key; also opened by `piri wallet import` and `piri wallet list` |

The S3 storage mode keeps all three on local disk.

Piri's dependencies compile LevelDB code into the binary but never open a store at runtime. Curio reaches Lotus's `node/repo` (LevelDB and badger) only through `curio/deps`, which Piri does not call. go-f3's certstore is used only by Lotus snapshot code. go-dsqueue is constructed only by boxo's reprovider, which Piri does not use. go-ethereum's database packages are not compiled in. `go-ds-leveldb` will stay in `go.mod` through Lotus, but no dependency writes LevelDB data.

## Design: PDP aggregates

### Schema

Aggregate data moves onto the pipeline rows that already describe each blob. Migration `00000008` adds two columns to `pdp_blob_pipeline`:

- `agg_position integer`: the piece's position in its aggregate.
- `inclusion_proof bytea`: the CBOR-encoded `merkletree.ProofData` for the piece.

Each aggregate piece is one row's `commp`, and the row already carries `aggregate_root`. An aggregate is the set of rows `WHERE aggregate_root = $1 ORDER BY agg_position`. Piece order and each proof's index must survive exactly, because AddRoots checks that sub-roots are in descending size order and recomputes the root from them.

### Lifecycle

`AggregateTask` sets `aggregate_root`, `aggregated_at`, `agg_position` and `inclusion_proof` in the transaction that folds the pieces. The aggregate and the pipeline state now commit together.

`PieceAcceptor` and `AddRootsTask` read the aggregate from these rows. They are the only readers. `/pdp/info` serves inclusion proofs from the `/pdp/accept` receipts, which are already in Postgres, and the prover works from Curio's `pdp_data_set_pieces` tables.

When AddRoots succeeds, the existing retirement transaction deletes the batch's `pdp_blob_pipeline` and `pdp_root_submissions` rows. The aggregate data goes with them. The table holds only aggregates between aggregation and AddRoots, so its size is bounded by the in-flight window.

Piri stops opening `data_dir/aggregator`. The aggregator storage config, its fx providers and the `aggregator_datastore` name are removed.

### Upgrade

No production node exists. Development and staging nodes may hold aggregates written to LevelDB but not yet submitted. Migration `00000008` does not import them. It clears `aggregate_root` and `aggregated_at` on every unretired pipeline row and empties `pdp_root_submissions`. Those blobs fold into new aggregates, which are written to Postgres.

One case loses data: AddRoots landed on chain but the node stopped before the retirement transaction committed. Those blobs are aggregated again and added to the data set a second time, so the node proves and is paid for them twice. On development and staging this is acceptable.

The leftover `data_dir/aggregator` directory is unused after the upgrade. Operators can delete it.

## Design: IPNI publisher store

The publisher store holds the IPNI advertisement chain:

- content-addressed blocks: adverts and entry chunks
- the chain head, updated with compare-and-swap
- two per-context maps, from context ID to entry-chunk link and to metadata

It grows by one advert and one entry chunk per blob, and nothing deletes them. Postgres already holds the queue of adverts waiting to be published (`ipni_pending_adverts`).

The store splits by kind of data:

- **Postgres** holds the head and the per-context maps. These are small, change on every publish, and need compare-and-swap. They can share a transaction with the advert queue.
- **The blob store** (flatfs on a local volume, or S3-compatible object storage) holds the adverts and entry chunks. These are immutable, content-addressed and written once, and the blob store is already sized for the node's data.

A publish writes its blocks to the blob store first, then moves the head in Postgres. A failure between the two leaves unreferenced blocks, which do no harm because nothing links to them. Indexers keep fetching blocks through the existing `/ipni/v1/ad/` endpoint.

The schema and blob-store layout belong to the implementation ([FIL-1577](https://linear.app/filecoin-foundation/issue/FIL-1577)).

No development or staging node has published to IPNI, so the move needs no migration. Nodes start with an empty chain.

## Alternatives Considered

- **A table with one row per aggregate**, holding the CBOR-encoded `Aggregate`. It keeps the current store interface but needs its own delete at retirement, and a record grows with the number of blobs in the aggregate (into megabytes for many small blobs).
- **A table with one row per piece.** Normalized, but it repeats the key that `pdp_blob_pipeline` already holds.
- **Import LevelDB aggregates at startup.** It preserves in-flight work but keeps LevelDB code for one more release. No node holds data that needs it.
- **Read from Postgres, fall back to LevelDB for one release.** The same cost as an import, spread across the runtime path.
- **The whole publisher store in Postgres.** One backend, and every write can share a transaction. The table grows by roughly 1–2 KB per published blob for as long as the chain is never trimmed, so Postgres would carry data the blob store is better sized for.
- **Keep the publisher store in LevelDB.** Backups would still need a consistent snapshot of LevelDB and Postgres together.

## Open Questions

### Keystore

The LevelDB keystore holds the owner's wallet key as plaintext. At startup, `provisionPDPState` copies it into the Postgres table `eth_keys`, and Curio's transaction sender and `RegisterProvider` sign with that copy. After provisioning, the LevelDB keystore serves only a startup existence check and the `piri wallet` CLI. The key is therefore already in Postgres, which contradicts the `LocalOnlyModule` comment "private keys must never leave disk".

Options:

- **`eth_keys` as the only store.** `piri init`, `piri wallet import` and `piri wallet list` read and write `eth_keys`, and the keystore package goes away. It matches where signing already reads the key. Database backups and database access then carry the key, so we would want encryption at rest and tight access controls.
- **Curio's file keystore** (`curio/lib/keystore`). One file per key, mode 0600, refusing keys with looser permissions. It keeps a local source of truth but still copies the key into `eth_keys`.
- **Keep LevelDB.** No change, and the plaintext copy in `eth_keys` remains.

## References

- [storacha/piri#22](https://github.com/storacha/piri/pull/22): PDP, introduces the aggregate store
- [storacha/piri#417](https://github.com/storacha/piri/pull/417): S3-compatible storage backend
- [storacha/piri#442](https://github.com/storacha/piri/pull/442): introduces `LocalOnlyModule`
- [fil-forge/piri#30](https://github.com/fil-forge/piri/pull/30): moves the aggregation pipeline onto harmonytask
- [fil-forge/piri#183](https://github.com/fil-forge/piri/pull/183): moves UCAN stores to Postgres
- [FIL-1577](https://linear.app/filecoin-foundation/issue/FIL-1577): Piri should store IPNI-related data in Postgres and the blob store
