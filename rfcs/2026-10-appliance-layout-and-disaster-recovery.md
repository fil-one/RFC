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

## Persisted state

We must never lose:

1. Blob payloads, which contain:

   - Content of S3 objects stored by customers
   - Piri's IPNI advertisements

   Each blob is the only copy, we don't replicate blobs at the Piri storage
   layer. We potentially may have an Ingot cache copy, but it's a cache that's
   not guaranteed to exist or stay around.

2. The Postgres database cluster, which holds Ingot's database and one database per Piri instance.

   Ingot stores the region-wrapped content encryption key (CEK) of every
   encrypted blob in Postgres. Losing a row makes the blob unreadable through the
   region; only the tenant recipient in the encryption envelope remains.

   Notably, losing region-wrapped CEKs does not make the blobs permanently lost.
   It is, however, extremely inconvenient. We can recover CEKs from the tenant
   keys, but we haven't implemented it yet, so the recovery would be a manual,
   slow process. We really don't want it to happen.

   Piri stores most of blob metadata and PDP pipeline state in Postgres. Losing
   state means losing the link between blobs and PDP Pieces. As a result, Piri
   stops proving data possession and FWSS stops payouts for the storage service.

3. OpenBao's raft storage. It holds the Piri and Ingot identity keys, the sprue
   and hilt delegations, the Piri owner wallet keys, the Postgres passwords, the
   chain RPC and Grafana push tokens, and the region key encryption key (KEK).
   Losing OpenBao makes every wrapped CEK in Postgres unusable, because the KEK
   is not exportable.

   Piri does not talk to OpenBao, and Ingot only wraps and unwraps CEKs using
   the KEK, which writes nothing. Secrets stored in the appliance OpenBao change
   only when an operator runs a provisioning script or rotates a value.

Note: The OpenBao root token and recovery key live off the appliance, in
FilOne's 1Password and in central OpenBao's KV store.

We can afford to lose the following data:

- Ingot's `spool` files.

  When lost, in-flight uploads fail and S3 clients retry. Parts of multipart uploads must be uploaded again.

- Ingot's `cache` files.

  When lost, we re-fetch the content from Piri.

- Ingot's `segments` files.

  When lost, Ingot rebuilds segments from Postgres and ships them again.

  This requires us to implement [FIL-1601 Rebuild unshipped catalog segments
  from the journal after losing the
  host](https://linear.app/filecoin-foundation/issue/FIL-1601) first. Until
  FIL-1601 lands, losing segments breaks every bucket with an unshipped segment.

- Piri's wallet keystore and `piri-config.toml`.

  When lost, we re-render the key from OpenBao and run `piri init` against the
  existing database, which finds the existing proof set. Running `piri init`
  against an empty database creates a new proof set, and the old one faults.

- Piri's egress journal.

  In [FIL-1558 Forge regions measure each space's egress, so the trial egress
  cap applies there](https://linear.app/filecoin-foundation/issue/FIL-1558), we
  will implement egress measuring in Ingot.

- Caddy state & TLS certificates issued via Let's Encrypt.

  When lost, we re-issue new certificates.

- Alloy (telemetry collector) state.

  When lost, we lose telemetry data that was not written to Grafana Cloud yet.

- Deployment state.

  When lost, we reinstall the appliance software stack and attach it to the
  existing Postgres cluster & OpenBao raft storage.

- The OpenBao seal token and the deploy token in `/etc/fil-one`.

  When lost, we mint new ones at central. The seal token is bound to the node's IP address.

Appliance recovery depends on central OpenBao in two places: it holds the
transit key that unseals the appliance's OpenBao, and the private key that
decrypts the appliance's backups. Central OpenBao stores its data in RDS
Postgres with a 35-day point-in-time recovery window.

## Disaster scenarios

### Data corruption due to bugs in our code

Examples:

- We ship a change in Ingot or Piri that silently corrupts the data stored in
  Postgres. We discover the problem after several days, and need to restore the
  last good state plus re-apply changes made since then.

- We ship a broken database schema migration script that damages the data stored
  in Postgres in a non-reversible way.

- We ship a change in infra-nodes infrastructure configuration or management
  scripts that accidentally delete directories (Postgres data, OpenBao secrets,
  Ingot segments).

A mistake that deletes a directory is recoverable:

- Postgres is recovered from the standby instance.
- OpenBao comes back from its snapshot.
- Ingot rebuilds its segments from data stored in Postgres.

Corruption written through the application is harder, because it replicates to
the standby and lands in the WAL archive. Restoring the whole database to the
last good state and re-applying later changes is not possible in production:

- Rewinding the `ingot` database deletes the wrapped CEKs of every object
  written after the restore point, so those objects become unreadable through
  the region.

- Rewinding a Piri database rolls it back after the chain has moved on. Roots
  already added on chain are added again, the chain challenges roots the
  database no longer knows, and blobs accepted after the restore point are
  never proven. Piri has no path to resync its state from the chain.

- Rewinding either database brings back references to blobs whose bytes Piri
  already deleted.

Production recovery therefore always restores to the moment of failure. To fix
corrupted state, we create a scratch cluster, load it with data at an earlier
moment using point-in-time recovery, inspect the damage and create deltas to
apply on the production cluster.

This scenario will have no automated recovery until we implement [FIL-1591
Restore one bucket's catalog after a bug corrupts
it](https://linear.app/filecoin-foundation/issue/FIL-1591).

### Hard-drive failures

One of the hard-drives dies. There are two scenarios:

1. A single drive in a multi-drive volume with replication (e.g. RAID). The
   machine operator replaces the drive with a new one and the replication
   mechanism handles the rest. No data is lost.

2. The entire volume is lost. This is less likely, but can still happen, e.g. in
   case of a two-drive RAID in a single physical machine where an
   electricity-related problem "fries" everything in the box.

   If the NVMe pool holding the control-plane volume is lost, we can restore
   Postgres from the standby instance, restore OpenBao from its latest snapshot,
   and let Ingot rebuild its local state from Postgres & data stored in Piri.

   Losing Piri's storage volume is the same as losing the datacenter (see
   [Appliance datacenter is destroyed](#appliance-datacenter-is-destroyed)
   below).

### Machine loss

The machine running the appliance dies.

A synchronous Postgres standby runs on a second machine at the same site,
so we are not losing any acknowledged writes. We promote the standby, rebuild
the appliance on new hardware from the OpenBao snapshot, mint a new seal token
at central, and start Ingot and Piri against the promoted database.

When the standby has been unavailable for a set time, the primary falls back to
asynchronous replication and alerts. Losing the host inside that window loses
the writes since the standby fell behind. If both machines are lost, we restore
Postgres from the WAL archive, which trails by up to 60 seconds and rewinds Piri
by that much.

### Unclean shutdown

A power loss or kernel crash stops the host without a clean shutdown.

Recovery:

- Postgres recovers from its WAL.
- Piri commits Postgres rows referencing blobs only after the blobs were written
  to the flatfs volume or S3 storage, there is no inconsistency after recovery.
  (This requires [FIL-1611 Sync flatfs writes before Postgres commits rows that reference them](https://linear.app/filecoin-foundation/issue/FIL-1611).
- Piri deletes blobs before marking them as deleted in Postgres. After restore,
  we may end up with orphaned blobs left behind. Fixing this is future work.

### Disk full

A full NVMe pool stops Postgres, OpenBao and Ingot together. Possible causes:

- Ingot overflows its configured storage limit because of a burst in incoming
  data - higher upload rate that we calculate the limits for.
- Postgres data size grows more than we anticipated.
- OpenBao data size grows more than we anticipated.

### Compromise or malicious deletion

An attacker gains root on the appliance, through stolen operator credentials,
ransomware or a rogue datacenter employee, and tries to destroy or read our data
or backups.

Backups go to a target where no credential on the appliance can delete or
overwrite them within the retention window. The appliance encrypts backups with
a public key whose private key it never holds.

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
loss, run on purpose, and it doubles as the restore drill.

### Appliance datacenter is destroyed

In the extreme case, the entire datacenter is destroyed and we lose all data
stored in that location.

I am arguing this scenario is out of the scope of our appliance disaster
recovery, it must be addressed at a higher level, e.g. by replicating data
across multiple location.

## Proposed storage layout

### Current layout

The layout is configurable. At the moment, we use the following setups:

#### Dev (AWS EC2)

- One EBS volume for all control-plane data, shared by Postgres, OpenBao,
  Caddy, Alloy and deployment state.
- One EBS volume for Ingot data (spool, cache, segments)
- One EBS volume per Piri node for storing blobs

#### Staging (bare metal, single volume)

One ZFS pool shared by all services: Postgres, OpenBao, Caddy, Alloy, Ingot, Piris.

### Proposed layout

Each production appliance has two storage tiers, plus a second machine at the
same site.

#### Fast pool

E.g. two NVMe drives in a mirror, one filesystem (or ZFS dataset) per row:

| Filesystem         | Contents                               | Backed up                                | Quota |
| ------------------ | -------------------------------------- | ---------------------------------------- | ----- |
| `control/postgres` | Postgres data directory, including WAL | standby, WAL archive and base backups    | no    |
| `control/openbao`  | OpenBao raft storage                   | raft snapshots                           | no    |
| `control/state`    | Caddy, Alloy and deployment state      | no                                       | yes   |
| `ingot/segments`   | Ingot catalog log                      | no, rebuilt from Postgres after FIL-1601 | yes   |
| `ingot/blobs`      | Ingot `spool` and `cache`              | no                                       | yes   |

Ingot's spool and cache share one filesystem, because Ingot moves a blob from
the spool to the cache with a rename. The cache sits on NVMe because a cache on
the same drives as Piri's blobs would not make reads faster.

In addition to the filesystems above, we need a volume to hold the root
filesystem, where we keep all Forge binaries, configuration files and Piri's
wallet keystore.

#### S3 storage

A remote S3-compatible object storage. If we need to run more than one Piri
node, then each Piri node will be configured with a different bucket or key
(path) prefix.

#### Storage array

The storage array (JBOD) option: one Piri node per each mounted volume. A volume
is sized in hundreds of TBs to PBs.

One filesystem per Piri node, sized in hundreds of
terabytes to petabytes, holding blob bytes, advertisements, the egress journal,
the wallet keystore and `piri-config.toml`. On an S3-backed Piri, blob bytes and
advertisements go to its S3 bucket and the small local files stay on the fast
pool.

#### Second machine

We need another machine to run the synchronous Postgres standby instance.

It needs disk that's large enough to hold the whole Postgres cluster, but ZFS
and filesystem snapshots are not required.

## Backup & restore

### Backup target

The target is configurable, either an S3-compatible bucket or a host we control,
and it must meet three requirements:

- It is off the appliance host. It may be at the same site.

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
  ([FIL-1602](https://linear.app/filecoin-foundation/issue/FIL-1602)). When the
  standby has been gone for a set time, the primary drops to asynchronous
  replication and alerts, and resumes synchronous replication once the standby
  catches up.

- **Archive.** Continuous WAL archiving with `archive_timeout = 60s`, plus
  scheduled base backups, to the backup target
  ([FIL-1599](https://linear.app/filecoin-foundation/issue/FIL-1599)). WAL-G can
  encrypt both with an OpenPGP public key.

- **Restore.**
  - Host loss: recover from the standby.
  - Both machines lost: restore the latest base backup and replay the archive to its end.
  - Forensics and drills: restore to any point in the last 35 days into a scratch cluster.

The 35-day retention sets the deletion guarantee. Deleting an object deletes its
wrapped CEK, and that deletion becomes irreversible once every backup taken
before it has expired, 35 days later.

The backups contain secrets, so access to them must be as tight as access to
the wallet keys:

- `curio.eth_keys` holds each Piri owner's wallet key in plaintext.
- `ingot.upload_registrations.proofs` holds bearer UCAN delegations, and
  `ingot.blob_parks.put_invocation` embeds derived signer keys.
- `ingot.catalog_journal` holds object keys and metadata in plaintext.

### OpenBao

- We take a raft snapshot daily and after any script that writes to OpenBao,
  encrypt it with the backup public key, and push it to the backup target.

- Restoring a snapshot needs central: the snapshot is sealed with central's
  transit key, and the restored node needs a new seal token minted at central.

- We do not back up the seal token or the deploy token. A rebuild mints new
  ones.

- After rotating the region KEK, we take a snapshot immediately. We never raise
  the key's `min_decryption_version` until every row is rewrapped and 35 days
  have passed, because Postgres backups within the retention window still hold
  CEKs wrapped with the older versions.

### Ingot

Nothing on Ingot's disk is backed up. Postgres holds every acknowledged write,
including the catalog blocks of unshipped segments. After a restore, Ingot
rebuilds and ships those segments from `ingot.catalog_journal` (FIL-1601), and
the spool and cache start empty.

### Piri

#### Database

Backed up with the Postgres cluster. It is restored only to the moment of
failure, never to an earlier point.

#### Blob store

No off-host copy: the storage array relies on RAID, and the S3 endpoint relies
on its provider.

To recover from our own deletion bugs, we keep 35 days of undo: daily filesystem
snapshots on local storage, and bucket versioning with 35-day expiry of
noncurrent versions on S3. We use the undo window only to copy back individual
blobs. Rolling a whole volume back to a snapshot would leave committed rows
pointing at missing bytes.

Blobs are encrypted, so keeping deleted ciphertext for 35 days does not weaken
the deletion guarantee.

#### Local files

The wallet keystore and `piri-config.toml` are regenerated from OpenBao and
`piri init`. The egress journal is not backed up.

### Caddy

Not backed up. A new appliance re-issues its TLS certificates.

### Alloy

Not backed up. Telemetry not yet written to Grafana Cloud is lost.

## Prerequisites for production

- [FIL-1601](https://linear.app/filecoin-foundation/issue/FIL-1601): Ingot
  rebuilds unshipped segments from `ingot.catalog_journal`.

- [FIL-1611](https://linear.app/filecoin-foundation/issue/FIL-1611): Piri syncs
  flatfs writes before committing rows that reference them.

- [FIL-1616](https://linear.app/filecoin-foundation/issue/FIL-1616): every
  appliance sets Ingot's byte budget for spool and cache.

- [FIL-1598](https://linear.app/filecoin-foundation/issue/FIL-1598),
  [FIL-1600](https://linear.app/filecoin-foundation/issue/FIL-1600) and
  [FIL-1602](https://linear.app/filecoin-foundation/issue/FIL-1602): the second
  machine at each site and the synchronous standby on it.

- [FIL-1599](https://linear.app/filecoin-foundation/issue/FIL-1599): WAL archive
  and base backups.

- [FIL-1166](https://linear.app/filecoin-foundation/issue/FIL-1166): OpenBao
  snapshots and the control-state restore drill.

- [FIL-1619](https://linear.app/filecoin-foundation/issue/FIL-1619): a key pair
  per region for encrypting backups, with the private key in central OpenBao
  behind a restore-only policy.

- (FIL ticket TBD): the blob undo window, with a snapshot-capable filesystem on
  the storage array and bucket versioning with noncurrent-version expiry on S3.
  A candidate for the S3 part is
  [FIL-1403](https://linear.app/filecoin-foundation/issue/FIL-1403) "Stand up
  the destination appliance".

- (FIL ticket TBD): fast-pool filesystems or ZFS datasets with the quotas from
  the layout table. A candidate is
  [FIL-1343](https://linear.app/filecoin-foundation/issue/FIL-1343) "Configure
  the Spain hardware".

- [FIL-1166](https://linear.app/filecoin-foundation/issue/FIL-1166) and
  [FIL-1599](https://linear.app/filecoin-foundation/issue/FIL-1599): the backup
  target, provisioned with Object Lock in compliance mode or an account that
  cannot delete.

- (FIL ticket TBD): a copy of the OpenBao root token and recovery key in
  central OpenBao's KV store.

- [FIL-492](https://linear.app/filecoin-foundation/issue/FIL-492): a tool that
  rewraps every CEK after a region KEK rotation.

- [FIL-1183](https://linear.app/filecoin-foundation/issue/FIL-1183): no secrets
  rendered into a tmpfs, so Piri and Ingot start after a reboot without a manual
  `deploy-apps.sh`.

- [FIL-1446](https://linear.app/filecoin-foundation/issue/FIL-1446): durable or
  fetchable ship authority, so Ingot ships idle buckets' segments after a
  restart.

- [FIL-1622](https://linear.app/filecoin-foundation/issue/FIL-1622): services
  refuse to start on an unmounted volume.

- (FIL ticket TBD): RAID, ZFS and SMART health metrics with alerts.

Higher-level tickets covering some of the above:

- [FIL-807](https://linear.app/filecoin-foundation/issue/FIL-807)
  Postgres replication and point-in-time recovery

- [FIL-1187](https://linear.app/filecoin-foundation/issue/FIL-1187) backups for all durable appliance state

## Open issues

- Restoring one bucket's catalog after a bug corrupts it
  ([FIL-1591](https://linear.app/filecoin-foundation/issue/FIL-1591)).

- Piri cannot resync its database from the chain. Any rewind of a Piri database,
  including the rewind of up to 60 seconds after losing both machines, needs
  manual repair.

- Nothing reclaims blobs on Piri that no database row references, and nothing
  reconciles Ingot's blob tables with what Piri holds after a restore.

- `piri-config.toml` is written with mode 0644 and holds the Postgres password
  and, on S3-backed nodes, the S3 secret key.

- `curio.eth_keys` holds the wallet key in plaintext. Where the key should live
  is an open question in the
  [Piri state RFC](./2026-10-piri-state-in-postgres.md#keystore).

- A running appliance keeps serving after its seal token is revoked, until it
  restarts. A central-triggered seal would close that window.

## Alternatives considered

_This section was written by Claude. I am sorry. 🙈_

### Postgres

- **Nightly `pg_dump` or hourly filesystem snapshots.** Simpler than WAL
  archiving, but a restore loses up to a day or an hour of metadata and orphans
  every blob written in that window.
- **WAL archive only, without a standby.** Needs no second machine, but losing
  the host rewinds every database by up to 60 seconds, including Piri's state
  that has to follow the chain.
- **One Postgres cluster for Ingot and one for the Piris.** Each could be
  restored to a different point. Production never rewinds a database, so one
  cluster is enough, and a scratch cluster serves forensics.
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
  to the backup target.** Ingot keeps only the newest segments, to aggregate
  them before shipping. We accept that losing a Piri volume can break listing
  for buckets with segments on it.
- **A full copy of the blobs on a backup host, or replication to a second
  Piri.** Either doubles raw storage, and replication needs work in sprue. We
  rely on RAID and treat the loss of a whole volume like the loss of the
  datacenter.

### Recovery tooling

- **A reconcile tool for Ingot's blob tables, Piri's records and the chain,
  required before production.** It stays an open issue; manual repair covers
  the rewind of up to 60 seconds that remains possible.
- **Deletion grace periods as long as the retention window.** Restored metadata
  would never point at freed bytes. Production restores only to the moment of
  failure, so the exposure is limited to deletions in the last 60 seconds
  before losing both machines, and the 35-day undo window for blobs already
  holds those bytes for a manual copy-back. A long grace in Ingot would also
  keep deleted blobs claimed, in the on-chain data set and paid for during
  those 35 days.

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
