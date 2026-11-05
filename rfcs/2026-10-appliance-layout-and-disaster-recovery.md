# RFC: Appliance Layout and Disaster Recovery

**Status:** Proposal

## Authors

- [Miroslav Bajtoš](https://github.com/bajtos)

## Introduction

In this RFC, we document what state the appliance needs to persist, describe the
storage layout on the host, and propose a backup strategy that allows us to
recover in various disaster scenarios.

Prior work:

- [Forge Instances & Pilot Regions](./2026-09-forge-instances.md)
- [Regional security principles and key management deployment proposal](./2026-08-regional-security-and-key-management.md)
- [Fil One Appliance Deployment Strategy](./2026-08-fil-one-appliance-deployment-strategy.md)
- [Fil One on Forge — Deployment Proposal: Co-located Guppy + Piri](./2026-05-filone-forge-deployment-proposal.md)
- [Fil One Object Encryption](./2026-06-fil-one-encryption-design.md)
- [Keep Piri's State in Postgres and Blob Store](./2026-10-piri-state-in-postgres.md)

## Scope

This RFC is proposing the storage layout that will work for the deployment
topology we are considering for the next 3-9 months:

The Blob Store is either a remote S3-compatible block storage or a custom
deployment using ZFS volumes to attach disk arrays to the appliance host.

We require the Blob Store to be resilient and store data redundantly (e.g.
erasure coded), and assume the blobs will not be lost. Each blob is the only
copy from Forge's point of view. Blobs of S3 objects may have a copy in Ingot
cache, but that's a cache, and such copies are not guaranteed to exist or stay
around.

All appliance services (Piri, Ingot) are running on the same machine and share
the same Postgres instance.

We have access to the private keys of wallets receiving payments for storage
services and keep them in appliance's OpenBao.

### Future direction (out of scope)

In 2027, we want to relax the durability requirement on blob store and allow
blobs to be occasionally lost without affecting the durability of customer data.
Anything written to the blob store will be erasure coded first and replicated
across Piri nodes.

Anywhere we would rather avoid that complexity, we would need to put data in the
database. In particular, we should rework Piri to build IPNI advertisements
on-demand by the HTTP request handler, using fragments of data stored in the
database.

Each Piri will have its own database server running alongside it, potentially
just a local SQLite. Piri nodes will live next to the disk(s) they write to, and
they may be managed by the regional provider.

## Persisted State

### Blob store

Blob payloads contain:

- Content of S3 objects stored by customers
- Ingot's MST nodes and manifests
- Piri's IPNI advertisements

In the current scope, we assume the blobs will not be lost.

### Control Plane

We must never lose:

1. The Postgres database cluster, which holds Ingot's database and one database per Piri instance.

   Ingot stores the region-wrapped content encryption key (CEK) of every
   encrypted blob in Postgres. Losing a row makes the blob unreadable through the
   region; only the tenant recipient in the encryption envelope remains.

   Notably, losing region-wrapped CEKs does not make the blobs permanently lost.
   It is, however, extremely inconvenient. We can recover CEKs from the tenant
   keys, but we haven't implemented it yet, so the recovery would be a manual,
   slow process. We really don't want it to happen.

   Additionally, Ingot stores the following data which we cannot afford to lose:
   - all bucket configuration: versioning, tenant, Object Lock, tagging,
     encryption, CORS
   - each bucket's current catalog root
   - the only map from a blob digest to the Piri instance that stores it (until
     we add IPNI & indexer)
   - the claim ledger used for deletion
   - multipart uploads in progress, whose parts are already on Piri
   - the `upload_registrations` outbox to Sprue

   Piri stores most of blob metadata and PDP pipeline state in Postgres. Losing
   state means losing the link between blobs and PDP Pieces. As a result, Piri
   stops proving data possession and FWSS stops payouts for the storage service.
   Losing the index of staged blobs makes these blobs immediately inaccessible.

2. OpenBao's raft storage. It holds the Piri and Ingot identity keys, the sprue
   and hilt delegations, the Piri owner wallet keys, the Postgres passwords, the
   chain RPC and Grafana push tokens, Ingot's region-key token, and the region
   key encryption key (KEK).
   Losing OpenBao makes every wrapped CEK in Postgres unusable, because the KEK
   is not exportable. (However, losing wrapped CEKs does not make the blobs
   permanently lost, as discussed above.)

   The appliance cannot regenerate the hilt proof or the third-party tokens;
   central and the providers must reissue them. Losing a Piri identity key
   means a new DID and re-onboarding.

   Piri does not talk to OpenBao, and Ingot only wraps and unwraps CEKs using
   the KEK, which writes nothing. Secrets stored in the appliance OpenBao change
   only when an operator runs a provisioning script or rotates a value.

Note: The OpenBao root token and recovery key live off the appliance, in
FilOne's 1Password.

### Loss-tolerant Data

We can afford to lose the following data:

- Ingot's `spool` files.

  Only in-flight uploads are in the spool and at risk, whether full objects or
  multipart parts. When lost, these in-flight uploads fail and S3 clients retry.
  Parts of multipart uploads must be uploaded again.

- Ingot's `cache` files.

  When lost, Ingot serves read requests by fetching the content from Piri.

  Note: At the moment, Ingot puts blobs in the cache only on the write path, and
  the read path never fills the cache. That's a performance optimisation we
  eventually need to implement.

- Ingot's `segments` files.

  When lost, Ingot rebuilds the unshipped segments from `ingot.catalog_journal`
  at startup ([FIL-1601](https://linear.app/filecoin-foundation/issue/FIL-1601)).
  A rebuilt segment ships after its bucket's next write, until
  [FIL-1446](https://linear.app/filecoin-foundation/issue/FIL-1446) makes ship
  authority survive a restart.

- Piri's wallet keystore and `piri-config.toml`.

  When lost, we re-render the key from OpenBao and force `piri init` to run
  again against the existing database, which finds the existing proof set.
  Running `piri init` against an empty database creates a new proof set, and
  the old one faults.

- Piri's egress journal.

  Piri's egress tracker is disabled today, so nothing writes the journal and
  losing it loses nothing. If Piri re-enables the tracker
  ([fil-forge/piri#14](https://github.com/fil-forge/piri/issues/14)), the
  journal holds the egress Piri has not reported yet, and that egress goes
  unreported. Alternatively, Ingot measures egress
  ([FIL-1558](https://linear.app/filecoin-foundation/issue/FIL-1558)), and
  Piri's journal stays empty.

- Caddy state & TLS certificates issued via Let's Encrypt.

  When lost, we re-issue new certificates.

- Alloy (telemetry collector) state.

  When lost, we lose telemetry data that was not written to Grafana Cloud yet.

- Deployment state.

  When lost, the next reconcile pass redeploys the appliance software stack
  against the existing Postgres cluster & OpenBao raft storage.

- The OpenBao seal token and the deploy token in `/etc/fil-one`.

  When lost, we mint a new seal token at central and a new deploy token with
  the appliance's OpenBao root token from 1Password. The seal token is bound to
  the node's IP address.

Appliance recovery depends on central in three places: central OpenBao holds
the transit key that unseals the appliance's OpenBao and the private key that
decrypts the appliance's backups, and `piri init` registers with central's
registrar. Central OpenBao stores its data in PostgreSQL with point-in-time
recovery: 35 days in production and 7 days in staging, where we run the restore
drills.

While central is down, a running appliance keeps serving, but an appliance
whose OpenBao restarts stays sealed, and no restore can complete. Central's own
durability is out of scope for this RFC.

## Disaster scenarios

Piri sets the recovery-time budget. Each day without a proof is a mainnet
proving period that pays nothing
([FWSS](https://github.com/fil-forge/filecoin-services/blob/08d9e619ca3c6cf5e764112445869e6c4f5492dd/service_contracts/src/FilecoinWarmStorageService.sol#L1626)).
The table estimates data loss and recovery time for each scenario.

| Scenario                           | Data lost                                               | Recovery time                                       |
| ---------------------------------- | ------------------------------------------------------- | --------------------------------------------------- |
| Silent data corruption due to bugs | None                                                    | No downtime, the repair can take days               |
| Bug or operator deletes data       | None                                                    | Hours                                               |
| One drive fails                    | None                                                    | No downtime                                         |
| Machine lost                       | None                                                    | Days to replace hardware, hours to rebuild the host |
| Both machines lost                 | The archive lag (see [Postgres](#postgres))             | Days to replace hardware, hours to rebuild the host |
| Unclean shutdown                   | None                                                    | Minutes                                             |
| Disk full                          | None                                                    | Hours                                               |
| Compromise or malicious deletion   | Blobs the attacker deletes, and writes not yet archived | Days, with downtime while we rebuild the host       |
| Lost OpenBao root credentials      | None                                                    | No downtime                                         |
| Appliance datacenter is destroyed  | Out of scope                                            | Out of scope                                        |

### Data corruption due to bugs in our code

Examples:

- We ship a change in Ingot or Piri that silently corrupts the data stored in
  Postgres. We discover the problem after several days, and need to restore the
  last good state plus re-apply changes made since then.

- We ship a broken database schema migration script that damages the data stored
  in Postgres in a non-reversible way.

- We ship a change in infra-nodes infrastructure configuration or management
  scripts that accidentally delete disk data (Postgres data, OpenBao secrets,
  Ingot segments). Such a change reaches every node at once.

- An operator runs `DROP DATABASE` on the wrong node.

A mistake that deletes files on the primary is recoverable:

- Postgres is recovered from the standby instance.
- OpenBao comes back from its snapshot.
- Ingot rebuilds its unshipped segments from `ingot.catalog_journal`.

A `DROP DATABASE` replicates to the standby. We restore the database from the
archive to just before the drop.

Corruption written through the application is harder, because it replicates to
the standby and lands in the WAL archive. Restoring the whole database to the
last good state and re-applying later changes is not possible in production:

- Rewinding the `ingot` database deletes the wrapped CEKs of every object
  written after the restore point, so those objects become unreadable through
  the region.

- Rewinding a Piri database rolls it back after the chain has moved on. Roots
  already added on chain are added again, the chain challenges roots the
  database no longer knows, and blobs accepted after the restore point are
  never proven. Piri has no path to resync its state from the chain (see
  [Open questions](#open-questions)).

- Rewinding either database brings back references to blobs whose bytes Piri
  already deleted.

Production recovery therefore restores to the latest recoverable point and
never chooses an earlier one. To fix corrupted state, we create a scratch cluster, load it with data at an earlier
moment using point-in-time recovery, inspect the damage and create deltas to
apply on the production cluster.

We need to keep blobs through the PITR window length, in case we want to roll
back a deletion.

This scenario will have no automated recovery until we implement [FIL-1591
Restore one bucket's catalog after a bug corrupts
it](https://linear.app/filecoin-foundation/issue/FIL-1591).

### Hard-drive failures

One of the hard-drives dies. There are two scenarios:

1. A single drive in a multi-drive volume with replication (e.g. RAID). A RAID,
   ZFS or SMART alert tells us the drive failed. The machine operator replaces
   the drive with a new one and the replication mechanism handles the rest. No
   data is lost.

2. The entire volume is lost. This is less likely, but can still happen, e.g. in
   case of a two-drive RAID in a single physical machine where an
   electricity-related problem "fries" everything in the box.

   If the NVMe pool holding the control-plane volume is lost, we can restore
   Postgres from the standby instance, restore OpenBao from its latest snapshot,
   and let Ingot rebuild its unshipped segments from `ingot.catalog_journal`.

   Losing Piri's storage volume is the same as losing the datacenter (see
   [Appliance datacenter is destroyed](#appliance-datacenter-is-destroyed)
   below).

### Machine loss

The machine running the appliance dies.

A synchronous Postgres standby runs on a second machine, so we are not losing
any acknowledged writes. We rebuild the appliance on new hardware: move the
storage array to the new host, copy the database from the standby, restore
OpenBao from its snapshot and mint new local tokens (see [Loss-tolerant
Data](#loss-tolerant-data)), point DNS at the new host, and redeploy Ingot and
Piri.

When the standby has been unavailable for a set time, the primary falls back to
asynchronous replication and alerts. If the host dies inside that window, the
new host copies the database from the standby and then replays the WAL archive
past the standby’s position, decrypting the segments with the region’s private
key from central OpenBao. We lose the writes that reached neither the standby
nor the archive: up to 60 seconds plus upload time while archiving works. If
both machines are lost, we restore Postgres from the WAL archive, which trails
by the same amount and rewinds Piri by that much.

Either restore from the archive rewinds the database, and the rewind needs
manual repair:

- Ingot loses writes it already acknowledged to S3 clients. Their bodies stay
  on Piri with no row referencing them.
- Piri's database falls behind the chain (see [Open questions](#open-questions)).
- Sprue, the indexer and IPNI keep the registrations, location claims and
  adverts of the lost writes.

### Unclean shutdown

A power loss or kernel crash stops the host without a clean shutdown.

Recovery:

- Postgres recovers from its WAL.
- Piri writes blobs and IPNI adverts to the flatfs volume or S3 storage before
  it commits the Postgres rows that reference them, so no row points at a
  missing file after recovery. On the flatfs volume, this requires
  [FIL-1611](https://linear.app/filecoin-foundation/issue/FIL-1611).
- A crash can leave bytes that nothing references, on Piri and in Ingot's
  spool, and can make Piri submit the same roots to the chain twice. These
  waste space or need manual repair. Their fixes are outside this RFC
  ([FIL-799](https://linear.app/filecoin-foundation/issue/FIL-799),
  [FIL-1456](https://linear.app/filecoin-foundation/issue/FIL-1456),
  [FIL-1391](https://linear.app/filecoin-foundation/issue/FIL-1391)).

### Disk full

Possible causes:

- Ingot overflows its configured spool/cache limit because of a burst in
  incoming data - higher upload rate that we calculate the limits for.
- Ingot's catalog log and `ingot.catalog_journal` grow while shipping stalls.
- Postgres data size grows more than we anticipated.
- Postgres keeps WAL for a missing standby or for an archive that cannot upload.
- OpenBao data size grows more than we anticipated.
- Docker container logs and old images fill the root filesystem.
- A Piri volume fills up.

In the proposed design, we have multiple volumes or ZFS data-sets to isolate
services from each other.

A burst in incoming traffic will prevent Ingot from accepting new uploads until
the periodic eviction reclaims the space. We run the sweep every 30 seconds with
no manual intervention needed.

Grafana alerts when any appliance filesystem has less than 40% free space, and
when Ingot's catalog shipping stalls.

### Compromise or malicious deletion

An attacker gains root on the appliance, through stolen operator credentials,
ransomware or a rogue datacenter employee, and tries to destroy or read our data
or backups.

We can recover metadata and keys. Backups go to a target where no credential on
the appliance can delete or overwrite them within the retention window, and the
appliance encrypts them with a public key whose private key it never holds.
We rebuild the appliance on a clean host, as after machine loss, and rotate
the secrets it held. We repair a malicious write the same way as a bug, from a
scratch cluster. After a `DROP DATABASE` or a wiped data directory, we restore
up to just before it. If the attacker stopped WAL archiving first, we lose the
writes that never reached the archive, the same as when both machines are lost.

The second machine holds a live copy of the database, wallet keys included, so
it needs the same protection as the appliance.

We cannot recover blobs the attacker deletes. Each blob is the only copy, and
root can destroy the 35-day undo window (see [Open questions](#open-questions)).
Protecting blobs against a compromise needs a copy outside the appliance's
reach, the same copy that protects them against losing the datacenter, so it
belongs to [Future direction](#future-direction-out-of-scope).

A compromised appliance can still serve plaintext while it runs; the [regional
security RFC](./2026-08-regional-security-and-key-management.md) covers that
limit.

### Lost OpenBao root credentials

Losing the root token or the recovery key of the appliance's OpenBao must not
lock us out.

The appliance host operator must not have access to the recovery key, because
that would allow them to migrate appliance's seal from central's transit key to
their own seal and thus remove appliance's dependency on central.

We propose keeping them both in two places: 1Password and central OpenBao's KV
store.

### Planned rebuild or migration

We move an appliance to new hardware.

This will follow the same runbook as host
loss, run on purpose, and it doubles as the drill for the standby path.

A separate drill restores Postgres from the base backups and the WAL archive.
Recovery from losing both machines depends on that path.

### Appliance datacenter is destroyed

In the extreme case, the entire datacenter is destroyed and we lose all data
stored in that location.

This is out of the scope of this RFC, see [Future
direction](#future-direction-out-of-scope) above.

## Proposed storage layout

### Current layout

The layout is configurable. At the moment, we use the following setups:

#### Dev (AWS EC2)

- One EBS volume for all control-plane data, shared by Postgres, OpenBao,
  Caddy, Alloy and deployment state.
- One EBS volume for Ingot data (spool, cache, segments)
- One EBS volume per Piri node for its whole data directory: blobs, IPNI
  advertisements, the wallet keystore, the egress journal and `piri-config.toml`

#### Staging (bare metal, single volume)

One ZFS pool holds Postgres, OpenBao, deployment state, Ingot and the Piris.

The host owns Caddy and Alloy. Its own Caddy serves the appliance and keeps its
state on the root filesystem. The host also runs workloads we do not own, which
may share disks with the appliance.

### Proposed layout

Each production appliance has two storage tiers, plus a second machine.

#### Fast pool

E.g. two NVMe drives in a mirror, one filesystem (or ZFS dataset) per row:

| Filesystem         | Contents                                                       | Backed up                                | Quota |
| ------------------ | -------------------------------------------------------------- | ---------------------------------------- | ----- |
| root volume        | The operating system, infra-nodes git clone, config files etc. | no                                       | no    |
| `control/postgres` | Postgres data directory, including WAL                         | standby, WAL archive and base backups    | no    |
| `control/openbao`  | OpenBao raft storage                                           | raft snapshots                           | no    |
| `control/state`    | Caddy, Alloy and deployment state                              | no                                       | yes   |
| `ingot/segments`   | Ingot catalog log                                              | no, rebuilt from Postgres                | yes   |
| `ingot/blobs`      | Ingot `spool` and `cache`                                      | no                                       | yes   |

Note: ZFS datasets share the pool's free space, so the loss-tolerant rows get a
quota, and `control/postgres` and `control/openbao` get a reservation that no
other row can take from. If we use fixed-size filesystems instead of datasets,
then we don't need quotas since each filesystem's size is its limit.

Ingot's spool and cache share one filesystem, because Ingot moves a blob from
the spool to the cache with a rename. The cache sits on NVMe because a cache on
the same drives as Piri's blobs would not make reads faster.

Piri's data directory holds the egress journal, the wallet key store and the
rendered config file. Since the journal's size is small and can be bounded, we
can keep it on the root filesystem, together with the wallet key store and the
config file.

#### S3 storage

A remote S3-compatible object storage. Each Piri node needs its own bucket,
because Piri supports a bucket prefix and no key prefix.

Blob bytes and IPNI advertisements go to its S3 bucket.

The egress journal, wallet keystore and `piri-config.toml` stay on the root
filesystem.

#### Storage array

An array of disk drives, attached to the single appliance host via one or more
ZFS volumes. (E.g. JBOD + Head Node with RAIDZ2 or RAID 6, but not
JBOD direct.)

The important constraint is that each Piri node is storing blobs in a single
directory (volume mount), Piri nodes cannot distribute blobs across multiple
locations.

Each Piri node will have one filesystem holding blob bytes and the IPNI
advertisements.

The egress journal, wallet keystore and `piri-config.toml` stay on the root
filesystem.

#### Second machine

We need another machine to run the synchronous Postgres standby instance.

It needs disk that's large enough to hold the whole Postgres cluster, but ZFS
and filesystem snapshots are not required.

This second machine can potentially run at a different site, or even in AWS on
EC2. At another site, every commit waits for the network round trip to the
standby.

The standby holds a live copy of the whole cluster, including `curio.eth_keys`
and the plaintext catalog, so the second machine needs the same access controls
and encryption at rest as the [Postgres backups](#postgres).

## Backup & restore

### Backup target

The target is configurable, either an S3-compatible bucket or a host we control,
and it must meet three requirements:

- It is off the appliance host. It can be at the same site or elsewhere.

- No credential on the appliance can delete or overwrite a backup within the
  retention window: S3 Object Lock in compliance mode, or a host where the
  appliance's account has no delete permission. The appliance pushes backups
  and never needs read access to old ones.

- Every backup is encrypted on the appliance with a public key. The private key
  lives in central OpenBao behind a restore-only policy. Deleting it at central
  makes every backup of the region unreadable, which keeps the regional
  security RFC's guarantee that a retired region loses access to all its data.

Retention is 35 days, the same window infra-central uses in production.

### Postgres

- **Replication.** A synchronous standby on the second machine, streaming
  through a replication slot
  ([FIL-1602](https://linear.app/filecoin-foundation/issue/FIL-1602)).
  Postgres waits for a missing synchronous standby indefinitely, so a watcher
  on the primary clears `synchronous_standby_names` once the standby has been
  gone for a set time, and restores it once the standby catches up. Each
  switch alerts. Commits stall for that set time before the fallback.

- **Retained WAL.** `max_slot_wal_keep_size` caps the WAL the slot keeps for a
  missing standby
  ([FIL-1602](https://linear.app/filecoin-foundation/issue/FIL-1602)). A standby
  that falls past the cap stops streaming, and we bring it back by hand. We
  either rebuild it from the primary with `pg_basebackup`, or replay the missing
  WAL from the archive after decrypting it with the region's private key from
  central OpenBao. WAL that fails to upload to the archive stays on the primary
  with no cap, so archive failures alert as well.

- **Archive.** Continuous WAL archiving with `archive_timeout = 60s`, plus
  scheduled base backups, to the backup target
  ([FIL-1599](https://linear.app/filecoin-foundation/issue/FIL-1599)). WAL-G can
  encrypt both with an OpenPGP public key. The archive lags the primary by up
  to 60 seconds plus upload time, and by more while uploads fail.

- **Restore.**
  - Host loss: copy the database from the standby to the new host. If the
    primary was running in asynchronous mode when it died, replay the archive
    past the standby’s position.
  - Both machines lost: restore the latest base backup and replay the archive to its end.
  - Forensics and drills: restore to any point in the last 35 days into a scratch cluster.

The 35-day retention sets the deletion guarantee. Deleting an object deletes its
wrapped CEK, and that deletion becomes irreversible once every backup taken
before it has expired, 35 days later. On the storage array, the blob undo
window keeps the deleted ciphertext equally long. The guarantee covers object
content only: object keys and user metadata stay on Piri in the shipped catalog
segments.

The backups contain secrets and customer data, so access to them must be as
tight as access to the wallet keys:

- `curio.eth_keys` holds each Piri owner's wallet key in plaintext.
- `ingot.upload_registrations.proofs` holds bearer UCAN delegations, and
  `ingot.blob_parks.put_invocation` embeds derived signer keys.
- Ingot's tables hold object keys, user metadata and fragments of multipart
  object bodies in plaintext.

### OpenBao

- We take a raft snapshot daily and after any script that writes to OpenBao,
  encrypt it with the backup public key, and push it to the backup target.

- Restoring a snapshot needs central: the snapshot is sealed with central's
  transit key, and the restored node needs a new seal token minted at central.

- We do not back up the seal token or the deploy token. A rebuild mints new
  ones: the seal token at central, the deploy token with the root token from
  1Password.

- The seal, deploy and region-key tokens lapse after 72 hours without renewal.
  After a longer outage, or a restore from an older snapshot, we mint new ones.
  A restore is complete once Ingot reads an encrypted object, because Ingot's
  health check does not detect a lapsed region-key token.

- After rotating the region KEK, we take a snapshot immediately. We never raise
  the key's `min_decryption_version` until every row is rewrapped and 35 days
  have passed, because Postgres backups within the retention window still hold
  CEKs wrapped with the older versions.

### Ingot

Nothing on Ingot's disk is backed up. Postgres holds every acknowledged write,
including the catalog blocks of unshipped segments. After a restore, Ingot
rebuilds those segments from `ingot.catalog_journal` and ships each one after
its bucket's next write. The spool and cache start empty.

### Piri

#### Database

Backed up with the Postgres cluster. It is restored to the latest recoverable
point and never to an earlier one.

#### Blob store

No off-host copy: the storage array relies on RAID, and the S3 endpoint relies
on its provider.

To recover from our own deletion bugs:

- Storage array: we keep 35 days of undo in daily filesystem snapshots.
- S3 endpoint: undo window would require a bucket versioning with 35-day expiry
  of noncurrent versions, this remains an open question ([see
  below](#open-questions)).

We use the undo window only to copy back individual blobs. Rolling a whole
volume back to a snapshot would leave committed rows pointing at missing bytes.

#### Local files

The wallet keystore and `piri-config.toml` are regenerated from OpenBao and
`piri init`. The egress journal is not backed up.

### Caddy

Not backed up. A new appliance re-issues its TLS certificates.

### Alloy

Not backed up. Telemetry not yet written to Grafana Cloud is lost.

## Prerequisites for production

- [FIL-1611](https://linear.app/filecoin-foundation/issue/FIL-1611): Piri syncs
  flatfs writes before committing rows that reference them.

- [FIL-1616](https://linear.app/filecoin-foundation/issue/FIL-1616): every
  appliance sets Ingot's byte budget for spool and cache.

- [FIL-1598](https://linear.app/filecoin-foundation/issue/FIL-1598),
  [FIL-1600](https://linear.app/filecoin-foundation/issue/FIL-1600) and
  [FIL-1602](https://linear.app/filecoin-foundation/issue/FIL-1602): a second
  machine for each appliance, at the same site or elsewhere, and the
  synchronous standby on it, replicating over an authenticated, encrypted
  connection.

- [FIL-1599](https://linear.app/filecoin-foundation/issue/FIL-1599): WAL archive
  and base backups.

- (FIL ticket TBD): alerts on WAL archive lag and on stale base backups and
  OpenBao snapshots.

- [FIL-1166](https://linear.app/filecoin-foundation/issue/FIL-1166): OpenBao
  snapshots and the control-state restore drill.

- [FIL-1745](https://linear.app/filecoin-foundation/issue/FIL-1745): daily and
  post-provisioning OpenBao raft snapshots, encrypted with the backup public key
  and pushed to the backup target.

- [FIL-1619](https://linear.app/filecoin-foundation/issue/FIL-1619): a key pair
  per region for encrypting backups, with the private key in central OpenBao
  behind a restore-only policy.

- (FIL ticket TBD): the blob undo window, with a snapshot-capable filesystem on
  the storage array, daily snapshots kept for 35 days, and spare capacity for
  the blobs deleted in those 35 days.

- (FIL ticket TBD): Ingot keeps deleted blobs longer than the WAL archive lag,
  so a restore from the archive never references freed bytes.

- (FIL ticket TBD): fast-pool filesystems or ZFS datasets with the quotas from
  the layout table. A candidate is
  [FIL-1343](https://linear.app/filecoin-foundation/issue/FIL-1343) "Configure
  the Spain hardware".

- [FIL-1166](https://linear.app/filecoin-foundation/issue/FIL-1166) and
  [FIL-1599](https://linear.app/filecoin-foundation/issue/FIL-1599): the backup
  target, provisioned with Object Lock in compliance mode or an account that
  cannot delete.

- [FIL-1678](https://linear.app/filecoin-foundation/issue/FIL-1678): a copy of the OpenBao root token and recovery key in
  central OpenBao's KV store.

- [FIL-492](https://linear.app/filecoin-foundation/issue/FIL-492): a tool that
  rewraps every CEK after a region KEK rotation.

- [FIL-1677](https://linear.app/filecoin-foundation/issue/FIL-1677) Piri and
  Ingot must start after a host reboot with no manual intervention needed

- [FIL-1446](https://linear.app/filecoin-foundation/issue/FIL-1446): durable or
  fetchable ship authority, so Ingot ships idle buckets' segments after a
  restart.

- [FIL-1622](https://linear.app/filecoin-foundation/issue/FIL-1622): services
  refuse to start on an unmounted volume.

- (FIL ticket TBD): RAID, ZFS and SMART health metrics with alerts.

- (FIL ticket TBD): cap the disk space used by Docker container logs and by old
  Docker images, so they cannot fill the root filesystem.

- (FIL ticket TBD): staging keeps only Piri's blobs on the ZFS pool. Each
  Piri's data directory moves to the root filesystem, with its blobs left in
  place through `repo.blob_dir`
  ([fil-forge/piri#212](https://github.com/fil-forge/piri/pull/212)), and
  Ingot's data directory moves to the root filesystem. Both moves copy the
  existing files while the service is stopped: if Piri's egress tracker is
  enabled, the egress journal holds egress not yet reported. Alternatively, we
  can reset the stage (wipe out all stored data) before executing the
  migration.

- (FIL ticket TBD): Piri caps the total size of its egress journal, so batches
  it cannot report do not fill the root filesystem. Today only each batch's
  size is limited. Needed before the egress tracker module is re-enabled
  ([fil-forge/piri#14](https://github.com/fil-forge/piri/issues/14)), unless
  [FIL-1558](https://linear.app/filecoin-foundation/issue/FIL-1558) moves
  egress tracking to Ingot first.

Higher-level tickets covering some of the above:

- [FIL-807](https://linear.app/filecoin-foundation/issue/FIL-807)
  Postgres replication and point-in-time recovery

- [FIL-1187](https://linear.app/filecoin-foundation/issue/FIL-1187) backups for all durable appliance state

## Open questions

- Restoring one bucket's catalog after a bug corrupts it
  ([FIL-1591](https://linear.app/filecoin-foundation/issue/FIL-1591)).

- S3-backed Piri has no undo window for our own deletion bugs.

  Bucket versioning with a 35-day expiry of noncurrent versions would provide
  one, but the [storage-qualification
  harness](https://github.com/fil-one/storage-qualification/blob/b46ffe5b10f4a4f21ae6ea0a335e31989e832a21/MANUAL.md#L541-L546)
  fails any bucket with versioning enabled and asks bidders to disable it,
  because without an expiry rule Piri's deletes leave versions behind and the
  bucket grows without bound.

  Adopting versioning means the harness requires versioning with a
  noncurrent-version expiration rule and checks that the store applies it. We
  would also need to verify that the selected IST-1 provider meets these new
  requirements.

  Without versioning, a deletion bug on an S3-backed node is unrecoverable.

- Piri cannot resync its database from the chain. The chain holds only each
  root's piece CID, and only Postgres knows which blobs make up a piece. Any
  rewind of a Piri database, including the rewind by the archive lag after
  losing both machines, needs manual repair.

- Nothing reclaims blobs on Piri that no database row references, and nothing
  reconciles Ingot's blob tables with what Piri holds after a restore.

- `piri-config.toml` is written with mode 0644 and holds the Postgres password
  and, on S3-backed nodes, the S3 secret key.

- `curio.eth_keys` holds the wallet key in plaintext. Where the key should live
  is an open question in the
  [Piri state RFC](./2026-10-piri-state-in-postgres.md#keystore).

- A running appliance keeps serving after its seal token is revoked, until it
  restarts. A central-triggered seal would close that window.

- An attacker with root on the appliance can destroy blobs for good.
  Control-plane backups are safe because no credential on the appliance can
  shorten their retention, but the blob undo window does not have that property:

  - Storage array: root can destroy the daily snapshots. Keeping them out of
    reach needs snapshots taken and held on a machine the appliance cannot
    administer.

  - S3 endpoint: Piri's S3 credentials can delete noncurrent versions. A bucket
    policy that denies Piri `s3:DeleteObjectVersion`, leaving lifecycle expiry
    as the only way versions go away, would close this. Object Lock alone does
    not, because its retention counts from when a version was written: the
    noncurrent version of a blob written more than 35 days earlier is no longer
    locked.

  Until we close both gaps, the undo window protects against our own bugs but
  not against a compromise.

## Alternatives considered

_This section was written by Claude. I am sorry. 🙈_

### Postgres

- **Nightly `pg_dump` or hourly filesystem snapshots.** Simpler than WAL
  archiving, but a restore loses up to a day or an hour of metadata and orphans
  every blob written in that window.
- **WAL archive only, without a standby.** Needs no second machine, but losing
  the host rewinds every database by the archive lag, including Piri's state
  that has to follow the chain.
- **Amazon RDS or Aurora as the standby.** Neither accepts physical replication
  from an external server
  ([AWS](https://aws.amazon.com/blogs/database/best-practices-for-migrating-postgresql-databases-to-amazon-rds-and-amazon-aurora/)),
  so the new host could not copy the database from either. Postgres on EC2 can run
  the standby.
- **One Postgres cluster for Ingot and one for the Piris.** Each could be
  restored to a different point. Production never chooses an earlier restore
  point, so one cluster is enough, and a scratch cluster serves forensics.
- **Rewinding the whole database after a bug corrupts data**, or repairing the
  damaged rows from a scratch copy. A rewind deletes the wrapped CEKs of every
  object written since the restore point and rolls Piri back after the chain
  has moved on. [FIL-1591](https://linear.app/filecoin-foundation/issue/FIL-1591)
  designs the repair of a damaged bucket.
- **A retention window of 14 or 90 days.** A shorter window makes deletion
  final sooner and leaves less time to notice corruption; a longer one stores
  much more WAL. 35 days matches infra-central.

### Backup target and encryption

- **Backups in another region or provider**, to survive the loss of the
  datacenter. Postgres and OpenBao without the blobs cannot bring a region
  back, and protecting blobs across sites belongs to cross-site replication.
- **The Piri disk array as the backup target, made append-only with
  `chattr +a` or ZFS delegated permissions.** Root on the appliance can undo
  both, and losing the array takes the backups with it.
- **A backup host that pulls from the appliance.** It keeps credentials off the
  appliance, but pushing works with every target (S3, another host, an
  attached volume) and the target enforces retention.
- **The backup key in the appliance's OpenBao.** Restoring OpenBao would need a
  key stored inside it.
- **Encryption of the target's disks only.** Anyone with access to the target
  can read backups, there is no way to revoke access from the central side.
- **A transit data key from central for every backup.** The private key never
  leaves central either way, but every backup would need central to be
  reachable, in addition to every restore.

### Storage layout

- **ZFS on every appliance, with snapshots taken atomically across Postgres
  and the data volumes.** A snapshot cannot span the NVMe pool and the storage
  array, and point-in-time recovery lands between snapshots anyway. Ingot
  rebuilding its segments from Postgres
  ([FIL-1601](https://linear.app/filecoin-foundation/issue/FIL-1601)) gives the
  consistency instead.
- **Fixing Ingot's segment numbering and orphan recovery** so the segment files
  survive a Postgres restore. FIL-1601 makes the files disposable, which also
  covers segments that shipped and were pruned after the restore point.
- **Keeping every shipped catalog segment on Ingot's disk, or copying each one
  to the backup target.** Segments are shipped to Piri as regular blobs. Instead
  of designing a custom backup solution for catalog segments only, it's better
  to improve how Ingot stores blobs (see [future
  direction](#future-direction-out-of-scope)) and lean on that for catalog
  segments too. For now, we accept that losing a Piri volume can break read
  operations for buckets storing MST data on the lost volume.
- **A full copy of the blobs on a backup host, or replication to a second
  Piri.** Either doubles raw storage, and replication needs work in sprue. We
  rely on RAID and treat the loss of a whole volume like the loss of the
  datacenter.

### Recovery tooling

- **A reconcile tool for Ingot's blob tables, Piri's records and the chain,
  required before production.** It stays an open issue; manual repair covers
  the rewind by the archive lag that remains possible.
- **Deletion grace periods as long as the retention window.** Metadata restored
  to any point in the window would never point at freed bytes. Production never
  chooses an earlier restore point, so a grace longer than the archive lag is
  enough. A 35-day grace would also keep deleted blobs claimed, in the on-chain
  data set and paid for during those 35 days.

### OpenBao

- **No standing root token, with recovery shares split 3-of-5 among FilOne
  staff or wrapped at central.** Losing one person's credentials could not lock
  us out, but every admin task would need a quorum or central. We keep the root
  token and recovery key in two vaults and never give them to the region
  operator.
- **Encrypting the root token and recovery key to a PGP key at init**, with
  `bao operator init -root-token-pgp-key=… -recovery-pgp-keys=…`, so they never
  appear in plain text on the appliance host. Our staff run provisioning over
  SSH, so both values reach only the operator's own terminal and nothing stores
  them on the node.
- **Backing up the seal token and the deploy token.** A rebuild on the same IP
  address would be faster, but the backups would then hold a credential that
  unseals the appliance.

### Verification

- **A nightly automated restore of the latest backup into a scratch cluster.**
  Alerts on backup lag and a quarterly drill cover the same ground at lower
  cost.
- **The same backup stack on dev nodes.** Dev takes no backups; staging runs
  the drills.
