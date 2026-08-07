# RFC: FilOne Appliance Deployment Strategy

**Status:** Proposal

## Authors

- [Miroslav Bajtoš](https://github.com/bajtos)

## Motivation

FilOne Appliance is a set of services operating FilOne node on infrastructure provided by a regional
provider. Two main goals are shortening time needed to set up a new region and guaranteeing high
operational rigour even if the regional provider does not have the necessary skills or experience.

Our selling point to regional providers: you provide hard drives and a VM, we bring our software and
customers.

This document proposes how to deploy and operate the appliance.

## Requirements & Constraints

1. Setting up a new region must not require any engineering or advanced system administration on the
   side of the region provider/operator.
2. FilOne must be able to deploy security and bug fixes in a timely manner (hours, not days).
3. FilOne must have visibility into operational metrics and logs.
4. Upgrades must cause as short downtime as possible. We should aim for zero-downtime upgrades.
5. Upgrades must honours timing constraints, e.g. we cannot upgrade in the window where the Piri
   node is required to submit a PDP proof.
6. The deployment should be managed using infrastructure-as-code.
7. We must pin versions of all services, so that we always deploy a combination of
   versions that was tested in staging and is known to work together correctly.

The following services do not support more than one instance running concurrently, therefore upgrades must be implemented as in-place restarts:

- Ingot
- Piri
- Postgres
- Vault
- Caddy

Zero-downtime upgrades are not possible now, this remains an aspirational future goal.

## Components & Dependencies

Our stack can be organised into the following layers:

1. Hardware
1. Operating System
1. Infra Services
1. Forge Services
1. External Dependencies

It is not yet clear who will operate which layer - FilOne or the region operator.

### Hardware

This is the bedrock on which we build the rest of the stack.

1. The machine: virtual or bare-metal
2. Network connectivity: a public IP under a stable domain name, and open port 443
3. Storage - control plane: a persisted volume mounted as local FS in the machine
4. Storage - data plane: an S3-compatible object storage, in the same datacenter, accessible via S3 (HTTPS)

### Operating System

A minimal Linux-based distro with a container runtime (Podman) and systemd.

Updated infrequently, primarily to apply bugfixes and security patches.

Easy to recreate, disposable.

### Infra Services

- A Postgres-compatible database, with point-in-time backup & recovery
- A secure secret manager (OpenBao, unsealed using FilOne's central OpenBao instance)
- Caddy (TLS termination, cert management)

Updated infrequently, primarily to apply bugfixes and security patches.

Backed up to an external location (not the control-plane volume).

### Forge Services

- Piri
- Ingot

Updated frequently to ship new features.

### External Dependencies

- Filecoin JSON RPC API like chain.love

## Proposal

1. Podman + Quadlet for running each infra & app service as a systemd unit.
1. Config files and pinned image versions tracked in git.
1. systemd-timer with git-pull script to reconcile.
1. Rollback is implemented as `git revert` to the previous set of configs & image versions and another deploy.
1. We can run on any Linux distro with Podman & systemd. If we own the OS, then let's use an immutable image-based OS like Fedora CoreOS or bootc.

### We operate the appliance

The flow - how are changes deployed:

1. A pull request modifies pinned image versions and/or config versions for the staging node
2. The pull request is merged
3. The staging node pulls the new commit (timer-based job) and reconciles the services
4. The staging node dispatches GHA verify workflow
5. The workflow performs automated end-to-end smoke tests
   - create a new tenant, create a new access key, create a new bucket, upload/download object, etc.
6. If the tests pass, the workflow creates a new pull request to update the per-region infra definition files
   - One pull request per region, so that we can roll out changes incrementally
   - If there is an already open pull request for the same region, the workflow closes it.
7. A developer merges the PR for the production region.
8. The production node pulls the new commit and reconciles the services
9. (Optionally: the production node dispatches a workflow to run non-destructive end-to-end tests)

### Provider operates the appliance

We need a slightly different approach in case the appliance services are operated by the region provider.

Straw-man proposal 1:

1. We add a new directory with image versions & base config files for provider-operated nodes, treat it as a new virtual region.
1. After the staging deployment passed the tests, the workflow opens a pull request to update that virtual region.
1. It's up to the region operator to decide when they want to pull the changes & apply them

Straw-man proposal 2:

1. Provider-operated nodes don't use git-based IaaC, they use Docker tags instead.
2. After the staging deployment passed the tests, if Piri or Ingot has a new image version, the workflow creates a new Docker tag for both Piri & Ingot images, sharing the same version number.
3. It's up to the appliance operator to watch new Docker image version and apply the updates, e.g. using podman auto-update.

### Why the immutable OS

- Prevents a sloppy operator from breaking the box
- Prevents [snowflake servers](https://martinfowler.com/bliki/SnowflakeServer.html)
- Unattended security updates
- A botched OS update is automatically reverted

## Scenarios / Runbook

### Upgrade Piri or Ingot image version or update their config

Lock-step: one commit bumps both digests and includes any config changes.

**Reconciliation**

1. Pre-pull both images (abort before any restart on failure)
2. `pdp-gate wait` (paying Piri's drain, up to 60 min)
3. restart `piri.service` then `ingot.service`, health-gated

**Downstream impact**

- Piri's restart is externally invisible but slow.
- Ingot drops in-flight requests (no graceful shutdown; SDK retries absorb most, browser pre-signed URLs don't)

**TODOs**

- Improve Ingot to support graceful shutdowns
- Investigate if Caddy can queue connections while Ingot is restarting

### Update Caddy config

**Reconciliation**

1. Sync configs (directory bind-mount — never a single file)
2. Check the config hash diff to detect changes
3. `systemctl reload caddy`; Caddy validates during load, and an invalid config fails the reload leaving the old config serving.

**Downstream impact**

Zero-downtime everywhere; `:443` never closes.

### Upgrade Caddy image version

1. Dependabot PR against the `deps/` shim; `make render` (CI) propagates the pin into the staging
   unit; GitHub auto-merge enabled for Caddy PRs — the human gate is removed, the staging gate is
   not.
2. Staging restarts Caddy (no pdp-gate), health-gated; verify workflow runs.

**Promotion to prod regions is manual, like every other tier**

- A prod Caddy restart closes `:443` briefly, and that downtime is only ever incurred on a human's decision.
- For a _security_ bump, patch latency now equals human response time, so add an alert on "staging
  carries a Caddy digest ahead of prod for > N hours" — the promotion stays a choice, but an
  unpromoted security
  patch must not be silent;

On merge of the promotion PR, prod nodes restart Caddy on their next tick.

**Downstream impact:**

- ~1–3 s of `:443` closed per node;

Socket activation is what would dissolve the downtime: with systemd owning `:443`, an image
swap queues connections in the accept backlog instead of refusing them, turning the restart
from downtime into added latency. This requires verification whether Caddy consumes `LISTEN_FDS`.

Post-restart checks include cert validity headroom, since TLS-ALPN-01 renewal failures are silent
until expiry.

### Update OpenBao config

Staging reconciler runs `bao-validate`, then `systemctl reload openbao` (SIGHUP).

### Upgrade OpenBao version

Rare, announced, human-gated per node. A restart seals the vault; recovery requires the unseal round-trip to central — a total regional read outage for the duration.

The reconciler does not pull the changes automatically. Instead, we implement a metric & alert to
let us know when a node is not up to date.

Per node, inside an announced window:

1. Pre-flight cheapest checks first:
2. Central OpenBao reachable and healthy (never overlap a central maintenance window)
3. Image pre-pulled;
4. Raft snapshot taken;
5. Downgrade/storage-format compatibility already checked in release notes _during staging_.
6. Optionally stop Ingot for a clean edge.
7. Human triggers the gated upgrade unit
8. Wait: start → unseal round-trip → healthy, where healthy = successful transit encrypt/decrypt.
9. Verify Ingot end-to-end reads; close the window;

Repeat per node — never fleet-parallel, since every one is a full regional read outage.

**Rollback**

Revert the digest in staging → verify → promote the revert; each node's human
re-triggers the upgrade unit, paying the seal/unseal cost again. If the new version migrated raft
storage format, rollback is restore-from-snapshot — which is why compatibility is a staging-time
check, not a prod-incident discovery.

### Update Postgres config

The reconciler does not pull the changes automatically. Instead, we implement a metric & alert to
let us know when a node is not up to date.

Per node, inside an announced window:

1. Edit in `nodes/staging/`; merge; staging runs `pg_ctl reload` — unconditionally safe;
   `postmaster`-context params simply don't apply and set `pending_restart`.
2. The verify suite includes `SELECT count(*) FROM pg_settings WHERE pending_restart;` — **a nonzero
   count is now visible at verify time, before promotion**, not just as a standing alert. Promote
   knowing whether a restart window is owed.
3. Prod nodes reload on their tick; the standing `pending_restart` alert remains the backstop per
   node.

## Upgrade Postgres patch/minor version

The reconciler does not pull the changes automatically. Instead, we implement a metric & alert to
let us know when a node is not up to date.

Per node, inside an announced window:

1. pre-flight:

   1. image pre-pulled,
   2. base backup verified,
   3. WAL current,
   4. no long transactions.

2. `systemctl restart postgres.service` from the new digest,
3. health gate,
4. Ingot reconnect verified.

**Downstream impact**

- Seconds-to-a-minute of Piri & Ingot metadata errors per node;
- No cascade (no `Requires=`);
- Optional clean stop of Ingot for zero error noise — pick one policy fleet-wide so partner-visible behaviour is consistent.

**Rollback**

Patch/minor formats are compatible in both directions in practice — revert in staging, verify,
promote, re-trigger per node. Release notes still checked for the rare on-disk change, at staging
time.

**TODOs**

- Verify that Piri and Ingot PG clients handle dropped connections and automatically reconnect after
  the PG server comes online again

### Upgrade Postgres major version

The one scenario where git revert is not the rollback: the data directory is rewritten; the
revert path is restore-from-backup.

**The staging run is a full dress rehearsal of the migration, including the restore rehearsal.**

1. **Plan at staging time:** release notes, extension compatibility, `pg_upgrade` vs dump/restore
   (for a single-node metadata DB, dump/restore is often the honest choice and doubles as a backup
   test).

2. **Stage:** digest + volume-path change (new data dir `/var/mnt/filone/postgres-<ver>`) in one commit to
   `nodes/staging/`. Human runs the purpose-built oneshot migration unit on staging:

   1. stop Ingot & Piri,
   2. old cluster stopped,
   3. migrate into the new data dir,
   4. start from the new unit,
   5. `vacuumdb --analyze-in-stages`,
   6. real-query verification,
   7. start Ingot,
   8. smoke suite.

3. **Promote the files.** Per prod node, inside a long announced pdp-clear window:

   1. fresh base backup restore-rehearsed as part of the window plan;
   2. disk headroom for two data dirs;
   3. stop Ingot & Piri deliberately (a window this long as ride-it-out is an error storm);
   4. run the same migration unit;
   5. verify;
   6. start Piri & Ingot only after verification — the point of no return is chosen, not stumbled past.

4. Keep each node's old data dir through an agreed soak; remove via follow-up commit/cleanup.

**Downstream impact:**

Full Piri & Ingot outage per node for the window (tens of minutes, size-dependent).

**Rollback**

Retained old data dir + backups. Writes accepted post-migration are lost on rollback, which is exactly why Ingot & Piri stays down until verification passes.

### Provision a new region

- Enrollment-based provisioning, two OS paths.
- Nothing per-node is baked into any image.
- All provisioning logic lives in one idempotent, repo-versioned script, `filone-enroll`; the only per-node input is a **single one-time enrollment token**.
- Two supported OS paths converge on the same enrolled state:

#### Enrollment

1. Path A — immutable OS

One _generic_ FCOS/bootc qcow2/ISO for the whole fleet, plus a per-node Ignition file:

- hostname/node-id,
- the `/var/mnt/filone` mount
- and a oneshot that runs `filone-enroll <token>` on first boot.

Keeps atomic updates, rollback, read-only `/usr`.

2. Path B — mutable OS, already-provisioned VM.

- Ops runs the same `filone-enroll <token>` on the existing box.
- The script performs what Ignition would have and its prereq check bites harder: **systemd +
  Podman ≥ 5.0 (`Notify=healthy`), hard-fail otherwise** — Debian 13 (5.4.2) passes, Ubuntu LTS
  archives do not without a backport.

  Accepted costs, stated so they are chosen and not discovered:
  - no atomic OS rollback or A/B updates;
  - OS patching becomes a mutable-OS process with an explicit owner;
  - bootc bound-images and `/usr`-baked options are unavailable.

#### Provisioning

Everything downstream of enrollment is byte-identical on both paths: units are distro-agnostic and
all state was already under `/var/mnt/filone`.

**What `filone-enroll` does** (the whole provisioning surface, in one place):

1. prereq check
2. data directory
3. `/etc/filone/node-id`
4. install reconcile script + timer + `filone-bao-bootstrap` oneshot
5. redeem the enrollment token against central OpenBao (OpenBao response-wrapping: single-use, short-TTL, minted per node)
6. obtain the secrets from the central OpenBao and seal them into `systemd-creds` (TPM-backed where
   available, host-key otherwise):

   - git read credential (unless the repo is public?)
   - registry pull credential (unless the GHCR images are publicly accessible?)
   - OpenBao seal credential

7. enable the timer.

Nothing app-specific; apps arrive from git on the first tick.

A used or expired token fails loudly; an intercepted token is single-use and therefore noisy, not quiet.

The steps:

1. **Central prep (Terraform):**
   - Transit seal key + policy + per-node credentials in/around central OpenBao, and mint the **wrapped enrollment token**.
   - The bootstrap set the token redeems into stays host-bound in `systemd-creds` — never in the local OpenBao (self-bricking rule).
2. **Repo prep:**
   - Add `nodes/<newnode>/` on `main` via the promotion script targeted at just this node — the node
     is born at the current verified-in-staging state, not at whatever staging has half-baked.
3. **Enroll:**

- Path A — boot the generic image with the node's Ignition + token.
- Path B — run `filone-enroll <token>` on the provisioned box.

4. **Node phones home outbound (first deploy metric, first journal batch); DNS pointed.**

   No inbound ports, ever, on either path.

5. **First convergence:**
   - fetch `main`,
   - validate
   - pre-pull
   - install units
   - start in dependency order.

     OpenBao first: unseal round-trip to central — confirm central reachability (and that the
     enrollment token hasn't expired) before scheduling.
6. **Acceptance:**

   - deploy timestamp fresh
   - OpenBao unsealed and passing a real transit op-
   - Ingot health exercising transit unwrap
   - Piri healthy
   - Caddy serving with first ACME issuance verified.

   Then announce.

## Detecting and troubleshooting a failed deployment

### Detection — three layers, cheapest first

1. **The restart itself fails.**

   `Notify=healthy` + `HealthCmd` + explicit `TimeoutStartSec` means
   `systemctl restart` returns non-zero if the service never reaches healthy (set the timeout
   explicitly — bug #27290 — and generously for Piri given the drain).

   The reconciler sees the failure synchronously.

2. **`OnFailure=` on every service unit**

   Fires an alert unit → textfile metric + journald event → Alloy → central Grafana.

   Catches crashes _between_ deploys too, not just during them.

3. **Deadman/staleness alert** on `deploy_last_success_timestamp_seconds`.

   A dead timer, a wedged git fetch, or a quarantine loop all look identical from central: the
   timestamp goes stale. Threshold must exceed poll interval + worst-case Piri drain (the 15-minute
   rev-1 value fires on every Piri deploy; size it above ~poll + 60 min until the drain question is
   resolved).

**TODOs**

We didn't want to run Alloy with access to our Grafana instance. How else can we get alerts?

### Automatic response in the reconciler

_This was suggested by Claude, I am not sure if this is a good idea. Almost certainly not needed for the initial version._

On any failed apply the reconciler:

1. records the bad SHA in a quarantine file so subsequent ticks skip it — no flapping;
2. reverts the working tree to the last-good SHA and re-applies, restoring the previous units and (because digests are pinned and images aren't aggressively pruned) the previous containers;
3. emits `deploy_failed{sha=…}` and leaves last-success stale so the deadman fires if nobody acts.
   Quarantine clears when a _new_ commit lands — the fix is always a new commit (revert or
   forward-fix), never a manual poke on the node.

Known limits, by design:

- forward-only DB migrations make app-revert insufficient on its own
- a partial multi-service failure (Piri restarted fine, Ingot didn't) reverts both to keep lock-step
- and a healthy-but-wrong deployment sails through — health-check quality is the ceiling on everything this layer can catch.

### Troubleshooting sequence (human, after the alert)

#### Where did it stop?

```
journalctl -u reconcile.service -n 200
```

The reconciler logs each phase (fetch, validate, pre-pull, per-unit apply). Check the quarantine file for the offending SHA and the exact failed step.

**TODOs**

This requires SSH access to the box. Should we send logs to Grafana instead?

#### Why did the unit fail?

```
systemctl status <unit>
journalctl -u <unit> --since <deploy time>
```

If the container started but failed health:

```
podman logs <container>
```

The usual buckets:

- image pull/digest mismatch (registry or credential)
- config invalid (should have been caught by `quadlet -dryrun` / `caddy validate` / `bao-validate` — if it wasn't, fix the validator too)
- health-check timing (came up slower than `TimeoutStartSec` — distinguish "broken" from "slow")
- or a dependency down (OpenBao sealed → Ingot can never go healthy; check unseal status _first_ whenever Ingot fails — it's the common cause with an uncommon-looking symptom)

#### Confirm current state is last-good

- `git -C /var/lib/filone/config rev-parse HEAD` vs the quarantined SHA
- `systemctl list-units 'filone-*'` all active
- running digests vs unit digests (`podman inspect --format '{{.ImageDigest}}'`)

#### Fix forward in git

1. Revert or fix on `main`
2. merge
3. CI renders
4. quarantine clears on the new SHA
5. node converges.

Only if the node itself is wedged (dead timer, corrupted checkout) does anyone touch the box — and
then only to restart the reconcile timer or re-clone, never to hand-edit units.

#### If the deadman fired without a deploy

- timer dead (`systemctl status reconcile.timer`)
- git credential expired/revoked
- egress broken
- (on bootc only) an OS rollback re-converging against an old app state: check `bootc status` for a rollback whenever node behaviour looks "impossibly old."

## Open Questions

### RPC API provider

Do we want to run a Filecoin node ourselves (in every region? one central instance?) or rely on a 3rd-party RPC API provider like chain.love?

On one hand, running a node is not trivial:

- HW requirements: ChainSafe's baseline is "16 GiB, 4 cores" vs Lotus's 32 GiB / 8 cores. Forest requires ~250 GiB storage, Lotus 300-400 GiB.
- We must deal with network upgrades
- Provisioning wall-clock: Forest - well under an hour, Lotus - tens of minutes.
- The recommended pattern is N+1 redundancy — two independent nodes behind a reverse proxy/load balancer.

On the other hand, 3rd-party providers have occasional downtimes (Chain.love reports 99.95%+ uptime) and can be quite expensive.

- Chain.love [pricing](https://chain-love.gitbook.io/chain-love-docs/blockchains/filecoin): 10M free CU per month, then $1/1M CU; dedicated mainnet node is $300/month.

### How much we can trust the region operator?

The region operator has hypervisor access and can read the SSH host key and any authorized-keys material.

Alternative to consider: Trusted Executed Environment, e.g. [Confidential Metal](https://confidential.ai/products#confidential-metal).

### Where to hold Region KEK

Per [RFC: Fil One Object Encryption](https://github.com/fil-one/RFC/blob/main/rfcs/2026-06-fil-one-encryption-design.md),
Ingot stores the region KEK in a secure secrets manager as a non-exportable key and asks the manager
to wrap/unwrap individual CEKs.

This is putting a ceil on the maximum throughput we can achieve. Analysis by Claude, comparing Vault and YubiHSM 2:

|                                 | YubiHSM 2  | Vault Transit         |
| ------------------------------- | ---------- | --------------------- |
| Per-op latency                  | ~10 ms     | 0.3–1 ms              |
| Write path ceiling              | ~100 PUT/s | 3k–10k PUT/s          |
| Read path ceiling (single-part) | ~100 GET/s | 3k–10k GET/s          |
| Read path (5 GiB, 51 parts)     | ~2 GET/s   | ~100–200 GET/s        |
| Batching                        | None       | Yes, 20k–100k items/s |
| Scales with                     | Nothing    | vCPU, nodes           |
| Region rotation (10M parts)     | ~2.3 days  | ~1 hour               |

Alternative to consider: don't put the non-exportable key on the per-part path. Insert an
intermediary key instead.

```
HSM (region root KEK, non-exportable)
  └─ wraps → intermediate KEK (per-bucket, or per-day, or per-region-epoch)
       └─ Ingot holds unwrapped in memory, does A256KW locally in Go
            └─ wraps → per-part CEK
```

HSM ops drop from one-per-part to one-per-intermediate-key-cache-miss — effectively zero
steady-state — while local AES-KW on 32 bytes runs at millions of ops/sec. This is the standard KMS
envelope pattern (cf. [AWS KMS data key caching](https://docs.aws.amazon.com/encryption-sdk/latest/developer-guide/data-key-caching.html)).

This has an additional benefit: the appliance does not need to operate a secure secrets manager like Vault.

- We can keep the single per-region KEK in a local file.
- If we move to per-bucket KEKs, we can store the wrapped per-bucket KEKs in the Postgres database.

### Is a single VM feasible?

The single VM is an unmitigated single point of failure. VM loss = regional outage. The fix would
require us to drop the "just give us a VM" pitch.

## Prior art

### Impossible Cloud

Impossible Cloud's requirements for node operators ([link](https://docs.icn.global/network-architecture/scalernode-network#scalernode)):

1. Physical host maintenance
2. Boot configuration
3. Internal network configuration and management
4. External network connectivity to the Internet

They mention Daemon in the docs, but it's not clear whether that's a single binary or a set of services. And everything is closed-source :disappointed:

### StorJ

Storj Node is single Docker image
([docs](https://storj.dev/node/get-started/install-node-software/cli/storage-node)), using
watchtower for auto-updates. They also offer MSI installer for Windows and QNAP Storage Node App.

Storj Commercial Node ([docs](https://storj.dev/node/commercial-node)) is for datacentre-based
nodes; they recommend running one node per hard drive. It's the same Docker image, plus the
documentation shows a sample Ansible script.

## References

- https://www.thelinuxvault.net/blog/how-to-run-podman-containers-under-systemd-with-quadlet/

## Alternatives Considered

### Close runners-up

**Quadlet + ansible-pull on a timer**

- Requires Ansible on every box, which is awkward on an immutable /usr (layer it or containerize it) and creates version drift across a 30-box fleet
- `ansible.builtin.systemd_service` daemon-reload is not idempotent, so restart gating needs care to avoid double-restarts
- More moving parts than a shell reconciler for essentially the same convergence semantics

**Docker Compose + Portainer CE GitOps polling**

- Requires the root Docker daemon plus Portainer itself as persistent moving parts; a real penalty on `bootc`
- Mixed cadence is project-level, so protecting Postgres/OpenBao from an unintended recreate takes ongoing discipline
- "Force redeployment / git-wins" is documented as Business-Edition-only in the 2.27 LTS docs — unverified on CE
- Loses the systemd `OnFailure=` alerting path and per-unit cgroup directiveso

### Evaluated but rejected

**Komodo v2 (Core + Periphery)**

- Compose/Docker-only — does not manage Quadlet, conflicting with the bootc/Podman-native design
- Core + MongoDB compromise equals fleet-wide arbitrary execution across partner boxes
- Loses systemd fail-closed semantics and Notify=healthy health gating
- Key behaviors unverified from primary sources (pre_deploy abort-on-nonzero, critical for pdp-gate) plus issue #1381: config-only changes don't trigger redeploy via TOML sync

**Kamal**

- Mandatory inbound SSH violates the outbound-only trust boundary with partner hardware
- Its headline zero-downtime advantage dissolves for singleton Piri/Ingot services
- Weaker immutability story — assumes a mutable host

**podman auto-update + timer**

- Mutually exclusive with digest pinning: `Image=...@sha256:...` never triggers an update, so you'd need floating tags, making the registry — not git — the source of truth
- Its automatic rollback (re-tag to previous on restart failure) actively fights the git-revert model
- Ignores config changes entirely
- Kept only as a rejected CD engine; cosign verification layered independently instead

### Ruled out on category fit

- **Flux / Argo CD**: Kubernetes controllers with no supported non-K8s mode; we had ruled out K8s/K3s.
- **Harbormaster**: Compose-only, niche, low activity; maintenance status itself a risk. Strictly dominated by Portainer for the same runtime.
- **bootc bound images / `bootc upgrade` as CD**:binds app images to the OS lifecycle, so it can't express per-service cadences; useful as baseline pinning only.
- **Watchtower**: archived December 2025; dead upstream.

### Push family

- Long-lived SSH key in a CI secret — inbound port plus a forever-credential in GitHub reaching every box
- Tailscale ephemeral push — cleanest push option, but node state = last push (weaker git-as-truth), no coalescing for Piri's long drain, and the box joins a central tailnet with the OAuth secret in GitHub
- OIDC short-lived SSH certs — most secure push, most moving parts (you run a CA)
- Self-hosted runner on the node — trust inversion: CI identity executing arbitrary code on the partner's box; GitHub's own docs warn against it
- Webhook receiver on node — needs an inbound port (or tunnel), and per-push firing became a liability once coalescing mattered for Piri's drain
- Ansible push over SSH — same inbound concerns; ansible-pull dominates it

---

**THE TEXT BELOW CONTAINS RAW NOTES AND CLAUDE OUTPUT. PROBABLY NOT WORTH READING, BUT A USEFUL CONTEXT FOR CLAUDE.**

---

# 2026-08-05 notes from chatting with Forrest

Unresolved question: who operates the appliance? Us or the partner? Maybe we can support both.

We can have three ways:

1. Docker-based (raw docker compose, K8 cluster)
2. Run a binary as a systems service file (or nohup in tmux?!)
3. Golden Images (ISOs) - people must have hypervisors and know how to run an ISO image. People who
   have the HW to run this typically have Intel Xeon chips from 2014 era, which don't have SHA256
   extensions, and everything becomes super slow.

We cannot have zero-downtime setup by end of September.

- Piri does not support two instances at the same time reading from the same DB. This needs more testing.
- Ingot - does not allow horizontal scaling either, we cannot have two Ingot instances running at the same time (behaviour is undefined).

The RFC should outline how we go from what we have in Staging right now, to CI/CD, to the future described in this doc.
How we meet the stability requirements (things don't break when we have two versions). How a commit landed on Piri's main ends up in the staging environment.

If want Golden Images, we can produce them using Terraform, the purpose of TF is to deploy infra, not apps. Hashicorp produces some tool that's designed to produce Golden Images - "[packer](https://developer.hashicorp.com/packer)".

Piri attempts a graceful shutdown, it will (by default, configurable) take up to 60 minutes to shut
down. (Finish existing connections, wait until the task queue is drained & finished.) See
https://github.com/filecoin-project/curio/pull/1302

Ingot does not have graceful shutdown, but it is written in a way that it's expecting to be killed
anytime and will restart gracefully. We need to implement graceful shutdown to wait until
in-progress requests are handled. There is a go library for that -
https://github.com/facebookarchive/grace but we need to support Fiber HTTP server used by VersityGW.

We need to export Piri on a stable domain name, so that `~/.well_known/did.json` are public -> double check with Alan.

No need to run Grafana Alloy on the Appliance to ship logs & telemetry to our Grafana. Piri is exporting OTEL logs to collector operated by
Forge, the collector forwards them to our Grafana. Caddy supports OTEL out of the box too, see https://caddyserver.com/docs/metrics.

---

# 2026-08-05 Claude Research Report

**FilOne Appliance: Deployment Platform Comparison and Recommendation**

## TL;DR

- **Adopt Podman Quadlet + systemd on an immutable image-based OS (Fedora CoreOS/bootc or Flatcar), plus a small `pdp-gate` wrapper around `piri status upgrade-check`.** Still the recommendation. Simpler fallback: plain systemd units running the Go binaries.
- **But requirement 4 now has a caveat that cuts against this choice.** `podman auto-update` restarts a unit in place — that is a stop-start, not a cutover. Getting true zero-downtime app upgrades under Quadlet needs a proxy unit plus a blue/green unit pair plus a flip script: modest custom work, in a project that explicitly prefers off-the-shelf. **Kamal's `kamal-proxy` does exactly this natively and is the most off-the-shelf answer to requirement 4.** If zero-downtime app upgrades are a hard requirement rather than an aspiration, that is a legitimate reason to reconsider. See "Requirement 4" below.
- **Zero downtime is achievable for app upgrades and impossible for OS reboots.** Reboots are 30–90s on FCOS/Flatcar. **Live kernel patching is not available on Fedora CoreOS** (verified), and kexec is not worth its failure mode on partner hardware. So reboot _frequency_ and _duration_ are the only levers: keep the frequent security-fix path in containers and reboot-free, and batch host reboots into rare `pdp-gate`d windows.
- **Achievable downtime is partly an application property, not a platform property.** In-box cutover means two app versions coexist against one database during the overlap. If Piri or Ingot ship a non-backward-compatible migration, zero-downtime is off the table for that release no matter what the deployment layer does. **This is the load-bearing unverified assumption in the whole report.**
- **The real SLO is "outage shorter than client retry tolerance."** S3 SDK clients retry with backoff, PUTs are idempotent, multipart uploads are resumable. A sub-minute reboot is largely invisible to well-behaved clients; a cached NXDOMAIN is not.
- **The single VM remains an unmitigated single point of failure.** True zero-downtime and VM-failure tolerance both require two VMs plus a movable IP, which changes the commercial offer. That is a product decision worth making deliberately rather than discovering when a partner's VM dies.
- **Two platform-independent constraints still dominate:** the partner owns the hypervisor, so no durable high-value secret — above all Piri's funded wallet key — belongs on the VM; and the appliance is internet-facing on hardware you do not trust.
- **TLS: TLS-ALPN-01 on port 443, issued on-box by the proxy.** Needs no port beyond the 443 already required, no DNS credentials, and no acme-dns. Challenge type is not a security decision here — a partner controlling the network path can always mint a certificate for their own hostname — so **prompt DNS record removal on offboarding is the actual control**, and it currently has no owner or target. The one regression is MPIC: validation now depends on the partner's box being reachable from multiple global vantage points, and the quorum tightens through December 2026.

---

## Requirement 4: what zero-downtime actually means here

### Why DNS draining does not apply

Draining presupposes an alternative target. In this design:

- one region = one partner = one VM = one public IP
- each region exposes Ingot at `{region}.s3.fil.one`
- S3 clients are configured against that specific regional endpoint

Removing or repointing `us-east-1.s3.fil.one` does not shift traffic anywhere. It makes the endpoint unresolvable. And clients pinned to a regional endpoint do not fail over to another region — nor should they, since the data is regional by design and the objects live on that box.

**Record removal is worse than doing nothing.** Negative answers are cached per RFC 2308, with the negative TTL governed by the zone's SOA minimum — commonly 300–3600s. Pulling a record to cover a 60-second reboot can therefore make the endpoint unresolvable for five to sixty minutes, long after the box is back. **Keep the A record pointed at the box at all times and let TCP fail fast.** A refused connection recovers on the client's next retry; a cached NXDOMAIN does not.

### The mechanism that does work: in-box cutover

This is in fact the primary and only real mechanism, and it needs no second host and no DNS involvement.

1. **A long-lived reverse proxy owns :443 as its own unit** — nginx, Caddy, Traefik, or kamal-proxy — deployed as a separate systemd/Quadlet unit, **never bundled with the application**. Backends come and go behind it; the listening socket never closes. This is the single most important structural detail: if the proxy restarts alongside the app, every cutover is a visible connection reset and the whole exercise is pointless.
2. **Start the new app version alongside the old.** Health-check it. Shift the proxy to it. Let the old one finish its in-flight requests. Stop it. nginx and Caddy both reload configuration without dropping established connections.
3. **Alternative without a proxy:** systemd socket activation or `SO_REUSEPORT` handoff between process generations. Entirely viable in Go (`coreos/go-systemd/activation`, `signal.Notify` + `http.Server.Shutdown`), but harder to reason about, and you likely want a proxy anyway for TLS termination.

### Two constraints that follow

- **The VM needs headroom to run ~2× the application briefly.** If it is sized for exactly one copy of Ingot, cutover is impossible and you are back to stop-start. **This belongs in the partner hardware specification** alongside disks and the public IP, and it is easy to omit and painful to retrofit.
- **Schema migrations must be backward-compatible (expand/contract).** Cutover means two application versions run against one database during the overlap window. Ingot's overlap can be long — a large multipart upload can hold a connection for a considerable time; the old backend keeps serving in-flight requests while the new one takes all new ones, which stretches the two-version window. **If Piri or Ingot ever ship a breaking migration, zero-downtime is off the table for that release regardless of deployment tooling.**

### What cannot be zero-downtime

State this plainly rather than engineering around it:

- **OS reboots.** Nothing in the guest survives them; realistically 30–90s on FCOS/Flatcar. The two mitigations suggested in revs. 3–4 have now been checked and **both are withdrawn** — see "Host OS updates" below. The remaining mitigation is the one that matters most anyway: **keep the frequent security-fix path in containers so it never touches the host**, and batch host reboots into rare `pdp-gate`d windows.
- **Postgres major-version upgrades.** Single instance, write outage. Rare and schedulable.
- **The PDP proving singleton.** Not client-facing; gated separately by `pdp-gate`.

### The reframe: retry tolerance, not zero downtime

Given S3 semantics, "zero downtime" is probably the wrong target. S3 SDK clients retry with exponential backoff, PUTs are idempotent, and multipart uploads are resumable. The SLO worth engineering to is **"outage shorter than client retry tolerance,"** under which a sub-minute reboot is largely invisible to well-behaved clients — whereas a cached NXDOMAIN is not.

Measure your actual clients' configured retry windows rather than trusting SDK defaults, which vary by SDK and version and are frequently overridden in customer configuration.

This reframe yields a story that is both defensible and honest: **the frequent path (application security fixes, requirement 2) is genuinely zero-downtime; the rare path (OS reboots) is a brief, retry-absorbed blip.** No overclaiming to partners or customers.

### The only route to true zero-downtime

Two VMs plus an IP that can move between them — a floating/failover IP, or VRRP/keepalived. That is real blue/green, and it also supplies the currently missing answer to **VM-level failure, which today is an unmitigated single point of failure**.

The cost is commercial, not technical: the offer changes from "you provide hard drives and a VM" to "…and a second VM and a movable IP," cutting against the onboarding simplicity that is the selling point. Some hosters offer floating IPs readily; many do not. **This is a product decision and should be made deliberately.**

---

## Requirement 5: PDP-gated upgrades

The hardest requirement.

**PDP is not WindowPoSt.** Piri is a warm-storage node verified by PDP — a distinct proof system whose cadence is contract-configurable (hourly is typical in general guidance), not WindowPoSt's fixed 48-deadlines-per-24h. Research found Piri's `proofset state` reporting a 2880-epoch period with a 60-epoch (~30 min) window, which is a daily cadence; treat the actual figure as **unverified and contract-dependent**.

**Delegate to the application; never compute windows.** `piri status upgrade-check` returns documented exit codes (0 safe, 1 not safe, 2 unable to determine) and is explicitly intended for automation. Because cadence is configurable and may change with contract upgrades, any gate that derives safe windows from epoch math is a latent bug. This is why delegation is correct, not merely convenient.

**PDP failure is a slashing event.** Two consequences:

1. **Exit code 2 must fail closed.** Failing closed costs a delayed upgrade; failing open risks financial loss.
2. **A hard timeout is not a policy.** If the unsafe duty cycle is high, "cap at 2h then proceed" could mean security fixes never land — or land during a proving window. Escalate to a human with the tradeoff explicit: accept slashing risk now, or accept continued vulnerability exposure. **Name the owner before the first incident.**

Piri's `chain_current_epoch` and `next_challenge_window_start_epoch` metrics are for observing and forecasting windows, and for alerting — not for gating.

### Minimal gate per option

- **Quadlet + systemd (recommended):** `ExecStartPre=/usr/local/bin/pdp-gate` on the upgrade unit, driven by a `.timer`; systemd retries on the next tick. Same guard on the OS reboot unit. ~30 lines.
- **Plain systemd:** identical, without containers.
- **Compose + Kamal:** `.kamal/hooks/pre-deploy` runs the check, exits nonzero to abort.
- **K3s + Flux:** CronJob/controller toggling `flux suspend`/`resume`, plus PreStop hook and PodDisruptionBudget. For node reboots, **Kured** with `--blocking-pod-selector` / `--prometheus-url` and `--reboot-days/--start-time/--end-time` holds the reboot lock until safe.
- **Talos:** `talosctl upgrade` has **no application-aware gate**; must be wrapped externally. A mark against Talos.
- **Balena:** **supervisor update locks** (`/tmp/balena/updates.lock`) let the application hold a lock while proving. Cleanest native fit of any managed platform.

---

## Reachability

Partner VMs are **not** behind NAT. Piri serves retrievals and Ingot is an internet-facing S3 gateway; both need public DNS and inbound 443.

**Pull is still recommended, on operational rather than impossibility grounds.** Inbound 443 for the data plane does not imply inbound 22 for the control plane — separate rules, separate risk decisions.

- Pull does not depend on the partner maintaining a firewall rule or allowlist for FilOne, and survives policy changes and management-IP churn.
- A box offline during a push stays stale; a polling box self-heals. At 30+ regions, "which boxes missed the last push?" is a real cost.
- The partner can read the SSH host key and authorized-keys material off disk, so SSH-as-control-plane is weak on its own terms.
- Outbound-only management keeps the internet-facing surface to 443 alone.

Kamal over an overlay is now defensible if the team prefers push immediacy — and see the requirement-4 discussion, which strengthens Kamal's case considerably.

---

## DNS and TLS

**Onboarding.** FilOne owns `fil.one`. The partner supplies a public IP and opens 443; FilOne creates `{region}.s3.fil.one`. Nothing for the partner to renew or debug. Keep TTLs modest (60–300s) for agility.

**Where owning DNS earns its keep** — not for upgrades, but for: repointing a region when the partner's IP changes or the VM is rebuilt on new infrastructure; per-region certificate issuance without partner involvement; and permanently decommissioning or migrating a region. **Do not use record removal as a maintenance tool.**

**Certificates: TLS-ALPN-01 on port 443, issued on-box by the proxy.**

### Correcting the CAA account-pinning argument

Rev. 3 called CAA account-pinning (RFC 8657 `accounturi`) "the decisive mitigation," on the reasoning that an attacker holding only a delegated challenge credential could not issue without the pinned ACME account key. The mechanism is real — Let's Encrypt documents that `accounturi` stops someone who hijacks your domain but lacks your ACME account key, and `validationmethods` accepts `dns-01`, `http-01`, and `tls-alpn-01`.

**But it does not protect against this attacker.** In _any_ on-box issuance model — HTTP-01, TLS-ALPN-01, or on-box DNS-01 via a delegated credential — the ACME account key must live on the VM, and the partner reads it off disk. Pinning issuance to an account key the adversary holds accomplishes nothing against that adversary.

CAA pinning defends against the hostile partner **only** if issuance is central and the account key never reaches the box. Rev. 2 (central push + CAA pinning) was internally coherent; rev. 3 kept the CAA argument while moving issuance on-box, which silently broke it. Keep CAA regardless — it is free, and it does block third-party CAs and a DNS hijack by someone without box access — but do not count it as partner containment. Note the CA/B Forum is moving to make RFC 8657 processing mandatory rather than optional (changes dated September 2026), which strengthens it against third parties over time.

### Security equivalence, and where the real control lives

As long as TLS terminates on partner hardware, the partner can obtain a valid publicly-trusted certificate for their own region's hostname by at least three independent routes:

1. read the leaf private key off disk;
2. read the ACME account key off disk;
3. intercept inbound traffic to their own IP and self-answer HTTP-01/TLS-ALPN-01 — **without touching the VM at all**, since they control the network path.

Route 3 cannot be closed by any credential-scoping scheme. Therefore **challenge type is not a security decision in this threat model** and should be settled on operational simplicity.

The real control is that **FilOne owns DNS**. The partner's ability to self-issue lasts exactly as long as `{region}.s3.fil.one` resolves to their IP; their leaf key expires on its own. So the revocation lever on partner termination is _prompt DNS record removal_, and the marginal exposure from on-box issuance is bounded by how quickly FilOne pulls the record — entirely within FilOne's control. That is a far better place for the property to live than a CAA record whose premise the architecture violates.

### Why TLS-ALPN-01 specifically

- **HTTP-01 would add a second inbound port.** It can only begin on port 80 — arbitrary ports are disallowed by the ACME standard, and redirects (followed up to 10 deep, to ports 80/443 only) do not change where validation _starts_. Partner onboarding currently requires only 443; HTTP-01 would widen that.
- **TLS-ALPN-01 needs only port 443**, which is already open. RFC 8737 requires the validation connection to use TCP 443 with an `acme-tls/1` ALPN extension and SNI for the name being validated. Caddy enables it by default with no configuration.
- **It removes a silent-failure mode.** With HTTP-01, a partner blocking only port 80 breaks renewal while the service keeps serving — surfacing weeks later as an expiry. TLS-ALPN-01 shares the service port, so a port-level block takes the region down visibly instead.
- Caveats that do not apply here: certbot still lacks TLS-ALPN-01 support as of early 2026, and TLS-ALPN-01 breaks behind anything that terminates TLS upstream and strips ALPN. Caddy is the terminator and there is no CDN.

### What this deletes

Relative to rev. 3: no self-hosted acme-dns (a DNS server, database, per-region account provisioning, and its own TLS bootstrap trap); no `_acme-challenge` CNAME records; no DNS credentials on partner hardware; **and no dangling-CNAME subdomain-takeover vector, so rev. 3's hard decommissioning gate ceases to exist.**

### Required controls

- **`*.fil.one` wildcard keys must never reach a partner VM.** Per-region certificates only. Note that TLS-ALPN-01 _cannot_ issue wildcards — only DNS-01 can — so this limitation is aligned with the security posture rather than a cost.
- **CAA at the apex anyway**, e.g. `fil.one. CAA 0 issue "letsencrypt.org; validationmethods=tls-alpn-01"` plus `issuewild ";"`. Blocks other CAs and pins the challenge type. Add `accounturi` only if issuance ever moves central, where it would actually bite.
- **DNSSEC on `fil.one`.**
- **Prompt DNS removal is the termination control.** Make record deletion the first step of any partner offboarding, and treat time-to-removal as the security metric it is.
- **Persist certificates across rebuilds** on a data volume — not baked into the immutable image, not re-issued on boot. Let's Encrypt caps new certificates at 50 per registered domain per 7 days, globally across accounts; `fil.one` is one bucket for every region, and a fleet-wide reprovision of 300 regions would exceed it roughly 6×. Rate limits are counted on orders and certificates, **not** on challenge type, so this requirement is unchanged from rev. 3.
- **Use ARI-based renewals**, which are exempt from all rate limits. File a rate-limit override before 30 regions. Point all CI and image builds at staging.
- **Keep 90-day certificates; do not adopt the 6-day short-lived profile yet.** On-box TLS-ALPN-01 couples renewal to serving health: a bad deploy that breaks the proxy also blocks renewal. At 90 days with renewal at ~30 days remaining there are weeks of slack; at 6 days there are hours.
- **Require globally reachable 443 in the partner agreement** — see the MPIC risk below.
- **Do not use public Web PKI certs as internal service identity.** Use a separate private CA or SPIFFE-style identity so a freshly issued public cert grants no internal trust.

### The one genuine regression: MPIC exposure

This is the strongest argument against on-box HTTP-01/TLS-ALPN-01 in this setting, and it is worth stating plainly because it cuts against the recommendation.

Multi-Perspective Issuance Corroboration has been mandatory since 15 September 2025 (CA/B Forum ballot SC-067), and the quorum is ratcheting upward: 2 remote perspectives with 1 matching from September 2025; 3 remote with ≥2 matching from March 2026; 4 remote with ≥3 from June 2026; 5 remote with ≥4 from December 2026.

The asymmetry with DNS-01 matters here. Under TLS-ALPN-01, validation must reach **the partner's box** from multiple geographically separated vantage points. Under DNS-01, it reaches **FilOne's authoritative DNS**, which FilOne controls and knows is globally reachable. Partner-side GeoIP filtering, aggressive DDoS scrubbing, or regional blocking silently breaks issuance — and the tolerance for non-corroborating perspectives shrinks through December 2026. Operators running geo-restricted estates have hit exactly this at scale.

Mitigations: state global reachability of 443 without geo-filtering as a partner obligation; test from several regions during onboarding; and keep a **per-region DNS-01 escape hatch** (see below) rather than treating the challenge type as a fleet-wide commitment.

### Per-region DNS-01 escape hatch

Design the appliance so challenge type is a per-region setting. A region whose partner cannot make 443 globally reachable, or whose network breaks ALPN, falls back to DNS-01 — at which point that single region needs a delegated credential and the associated CNAME plus decommissioning hygiene. Keeping this as an exception rather than the default is what makes the acme-dns infrastructure optional instead of mandatory.

---

## The single-VM constraint

On one node you cannot reschedule: if the box is down, the region is down. No orchestrator changes that. The honest availability story is "zero-downtime app upgrades, sub-minute reboots absorbed by client retries, and full regional outage if the VM dies."

Running two copies of a stateless service behind a proxy gives cutover for the HTTP surface. It does nothing for the proving loop, which is a singleton bound to the wallet.

**What real single-node appliance vendors do.** GitLab's dominant on-prem form factor is **Omnibus** — one package, embedded supervisor, single VM, explicitly not Kubernetes; GitLab's docs warn the Helm path "requires more resources… Administration and troubleshooting require Kubernetes knowledge." Umbrel/Start9/CasaOS ship **Docker Compose** bundles. Balena ships a **container OS with A/B host updates plus a supervisor**. Home Assistant OS is an **image-based OS with a supervisor**. The recurring pattern is _supervisor + image-based OS_, not a cluster of one.

---

## Trust boundary and secrets

Piri references a `service.pem` key and a **funded delegated wallet (`wallet.hex`) that pays PDP contract gas**. On partner-controlled disk, that key is exfiltratable and the funds drainable.

- **Do not run a durable Vault inside the partner VM.** Transit-seal auto-unseal removes the seal key from the box, but unsealed secrets remain in partner-readable memory. Auto-unseal protects against disk theft, not a malicious hypervisor.
- **Prefer central Vault/OpenBao + Vault Agent** rendering short-lived credentials into tmpfs. SPIFFE/SPIRE can supply workload identity, but node attestation is weak when the attestor is the adversary.
- **Keep the wallet off the box.** Storacha ships **`piri-signing-service`**, documented such that "Keys never leave the signing service" while letting "Multiple piri nodes… request signatures without key access." The highest-leverage security decision in the design; make it a hard requirement.
- **A full Vault per region is not warranted.**
- **TLS keys and the ACME account key** belong on this list too.

---

## Stateful components: Postgres

**Piri defaults to SQLite** — the docs state it "is the default database backend, requiring no additional configuration." Postgres is needed only for multi-instance Piri or **Ingot in forge mode**. Options, best to worst:

1. **Avoid on-box Postgres.** If Ingot can use a FilOne-side Postgres, prefer that. Anything on partner disk is partner-readable.
2. **On-box Postgres container** with WAL archiving to the partner's S3 via **pgBackRest or wal-g**, nightly base backups, rehearsed PITR. On K3s, **CloudNativePG** with Barman Cloud — justified only if already on K3s.
3. **External managed Postgres** is cleanest operationally but adds a hop in Piri's hot path.

Major upgrades are **not zero-downtime**; schedule in a rare window. Note the interaction with cutover: **backward-compatible migrations are a hard prerequisite for zero-downtime app upgrades.**

---

## Immutable OS foundation

- **Fedora CoreOS / bootc (recommended).** Ignition first-boot, immutable `/usr`, atomic updates with automatic rollback, configurable reboot windows via zincati / `bootc-fetch-apply-updates`. Native Podman. bootc ships the **whole OS as a digest-pinned OCI image**, unifying OS and app pinning.
- **Flatcar Container Linux.** Maintained Container Linux successor; Ignition, A/B atomic updates. Strong alternative.
- **Talos Linux.** Most locked-down (no shell, no SSH, API-only), A/B with automatic rollback on failed boot. But Talos exists solely to run Kubernetes, so choosing it commits you to single-node K8s.
- **NixOS.** Fully declarative, atomic generation rollback. Cost: team-wide Nix adoption.
- **Ubuntu Core / snaps.** Transactional and appliance-oriented, but packaging and store dependency add friction for Go + containers.

**Partner install path:** a **pre-baked qcow2/OVA/ISO** with Ignition/cloud-init embedded. Partner boots it, it phones home, FilOne points DNS at the supplied IP. Concrete answer to requirement 1.

### Host OS updates: what is and isn't possible (verified)

Revs. 3–4 suggested checking live kernel patching and kexec. Both have now been checked, and **both suggestions are withdrawn.** The framing was also wrong in a more basic way: FCOS and bootc have no "update individual OS packages" operation to cherry-pick from. They do whole-image atomic transitions — which is exactly why they were chosen (A/B rollback, drift-free, digest-pinned). Asking which host packages can update without a reboot is a package-manager question aimed at a system that deliberately isn't one.

**Live kernel patching is not available on Fedora CoreOS.**

- Live patches are a _vendor-produced artifact per specific kernel build_, not a capability you enable. Red Hat builds `kpatch-patch` RPMs delivered via the Red Hat CDN, gated behind an active subscription, and only for **selected** Important and Critical CVEs — Red Hat's own documentation states live patching cannot address all critical or important CVEs, and that not all kernels receive live patches.
- **Fedora ships no such patches**, so there is nothing to install on FCOS regardless of tooling.
- rpm-ostree's kpatch integration is an **open enhancement issue since 2015**, with the maintainers noting the administrator UX and provider support remain unresolved.
- Upstream `kpatch` is now marked **deprecated**.
- Consequence: on FCOS, a kernel CVE means a reboot. **RHEL image mode with a subscription has been considered and rejected** — it is the only route to live kernel patching, and the subscription cost is not justified for coverage limited to selected CVEs. Reboots are therefore the accepted mechanism for the entire host tier, which makes reboot duration and the two-tier policy below the load-bearing controls.

**kexec is not worth it here.**

- `kexec-tools` is **not in the FCOS base image**; using it requires package layering, which contradicts the clean-image discipline recommended below.
- ostree's own `prepare-kexec` support has been an **open enhancement request since 2016**.
- `systemctl kexec` depends on the Boot Loader Specification interface to know which kernel to load, and Fedora's grub BLS implementation makes this awkward rather than turnkey.
- There is a **catastrophic failure mode on partner hardware**: a 2025 FCOS tracker issue reports kexec hanging with a permanent black screen, recoverable only by a hard power cycle. On a VM you do not own, recovery means asking the partner to intervene — precisely what requirement 1 exists to avoid.
- The payoff is small in your topology anyway. **kexec's saving comes from skipping firmware/POST**, which is already minimal in a VM; it does not skip kernel init, initramfs, or systemd startup, which is where most of a VM's boot time goes. Reported bare-metal savings (~50s → ~13s) do not transfer.

**`rpm-ostree apply-live` exists but does not solve this.**

- It supports live-applying **pure package additions** only, assuming no other pending changes; **uninstalls are unsupported** and upgrades are constrained.
- It works by overlaying `/usr` and writing the commit diff in, with state in transient `/run` that vanishes on reboot.
- The documentation warns it is experimental, may pull in other pending changes, and that services may crash or misbehave in ways that surface later and are hard to debug — recommending a reboot into the new tree as the safe path.
- **Decisively: replacing a file on disk does not change what is mapped into a running process.** A patched library on disk does nothing for a running Piri until Piri restarts. "Patch without restart" is close to incoherent for shared libraries, since the code being replaced is the code currently executing. You would still restart the service, gaining nothing over a container update.
- bootc rests on the same ostree machinery, so the same constraint applies. No supported bootc live-apply path was found; verify if it becomes load-bearing.

### Why this matters less than it appears

**Your services' libraries come from their container images, not the host.** Piri, Ingot, Caddy, Postgres, and Alloy each ship their own userspace, so a glibc or OpenSSL CVE in the _host_ image does not touch them. The **container base image** is the real surface, and updating it is already the fast path: digest bump, pull, cutover — no host change, no reboot.

Two properties compound this favourably:

- **The FCOS host is deliberately tiny** — systemd, podman, NetworkManager, sshd. Nothing on it is internet-facing; the only publicly listening process is Caddy, which is a container.
- **Piri and Ingot are confirmed pure-Go, statically linked, cgo-free** (verified against the repositories — evidence below). They do not link a system C library at all, and Go ships its own crypto stack. **Host glibc and OpenSSL CVEs therefore cannot affect them.** This is not a partial mitigation; for these two services it removes the entire host C-library CVE class.

Verification evidence, for future re-checking:

| Check                          | Piri                                                                                                                                          | Ingot                                     |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| `CGO_ENABLED=0` in build paths | Makefile, `.github/workflows/go-check.yml`, Dockerfile, `.goreleaser.yaml` — all four                                                         | Dockerfile (3 stages), `.goreleaser.yaml` |
| `import "C"` in sources        | none                                                                                                                                          | none                                      |
| SQLite driver                  | `glebarez/go-sqlite`, `ncruces/go-sqlite3` (WASM), `modernc.org/sqlite` — all pure Go. Notably **not** `mattn/go-sqlite3`, which requires cgo | n/a                                       |
| Postgres driver                | n/a                                                                                                                                           | `jackc/pgx/v5` — pure Go                  |
| CommP utilities                | `go-commp-utils/nonffi` — deliberately the non-FFI variant                                                                                    | n/a                                       |
| `filecoin-ffi`                 | present as `// indirect` but imported by **no source file**                                                                                   | absent                                    |

On that last row: `filecoin-ffi` is Rust FFI and cannot compile without cgo, so **the fact that `CGO_ENABLED=0` builds pass in CI is itself proof it is not in the import graph.** It is a module-graph artifact pulled in by another Filecoin dependency, not a linked component. This self-proving property is worth remembering — if a future dependency change made cgo genuinely necessary, the existing CI would fail loudly rather than silently producing a dynamically linked binary.

### Container base images: a live gap found during verification

Checking the above surfaced something the two-tier policy needs to account for. Both Dockerfiles use **floating base tags**, which contradicts this report's own "pin by digest, never by tag" rule:

- **Piri** runtime: `alpine:latest` — chosen, per a comment in the Dockerfile, for `wget` healthchecks.
- **Ingot** runtime: `debian:bookworm-slim` — a considerably larger surface including glibc and a full apt userspace.

Since both binaries are static, **neither needs a libc at all.** The opportunity is concrete: rebase both onto `gcr.io/distroless/static` or `scratch`, digest-pinned, replacing the `wget` healthcheck with a Go-native health command or a statically-linked probe. That would shrink the fast-tier CVE surface to approximately _the two Go binaries themselves_ — which is close to the theoretical floor for a containerised service, and makes the "hours not days" commitment in requirement 2 dramatically easier to honour because there is almost nothing left to patch.

Note the current risk asymmetry runs the wrong way: Ingot is the customer-facing S3 endpoint and carries the _larger_ base image of the two.

What genuinely requires a host reboot: **kernel CVEs**, **container-runtime CVEs** (podman/crun — these matter, being the container-escape path), and **systemd**. Few are remotely exploitable pre-auth on a box whose only exposed port fronts a Go reverse proxy, which is what makes batching them defensible rather than negligent.

### Two-tier CVE response policy

This replaces the vaguer "deploy fixes in hours" framing and makes requirement 2 concretely satisfiable:

| Tier        | Surface                                                                                                                                                                   | Path                                                                                       | Target          |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | --------------- |
| **Fast**    | Container images: the Piri and Ingot Go binaries, plus Caddy, Postgres, Alloy, and all base images                                                                        | Digest bump → signed image → pull → `pdp-gate` → proxy cutover. No host change, no reboot. | Hours           |
| **Batched** | Host OS only: kernel, podman/crun, systemd. **Confirmed excluded: host glibc, OpenSSL, and every other host C library** — Piri and Ingot are static and link none of them | New OS image digest → staged by zincati/bootc → `pdp-gate`d reboot in a scheduled window   | Days, scheduled |

With a documented **exception path**: a remotely-exploitable pre-auth kernel or container-runtime CVE triggers an unscheduled `pdp-gate`d reboot, accepting the 30–90s outage. Name the owner who can invoke this.

Operational rules that follow:

1. Configure zincati (FCOS) or `bootc-fetch-apply-updates` to **stage but not auto-reboot**; gate the reboot on `pdp-gate` plus the window.
2. **Do not layer packages** with `rpm-ostree install` unless forced. Layering complicates every subsequent rebase and is the main way teams lose the clean image model. (This is also the second reason to reject kexec.)
3. **Track container base images separately and aggressively** — that is the genuine hours-not-days surface.
4. **Treat reboot duration as the lever you actually control**, since neither live patching nor kexec is available. Measuring it is already in the bake-off plan; this raises its priority from nice-to-know to load-bearing.

---

## Image distribution and supply chain

Private FilOne registry with an edge pull-through cache for constrained sites; short-lived per-region pull credentials rather than a durable baked-in secret. **Sign with cosign and verify on the box before activation** — Podman via policy, Kubernetes via Sigstore Policy Controller or Kyverno. This is what stops a partner with disk access from swapping images: they cannot forge a signature. **Pin by digest, never by tag.**

---

## 3am failure handling

- **A/B OS rollback** reverts a bad OS image on failed boot, unattended.
- **Health-gated cutover** means a failed new version never receives traffic; the old one keeps serving.
- **`podman auto-update`** rolls back to the previous image if the new one fails its health check.
- **systemd `WatchdogSec` + `Restart=always`** recovers hung processes.
- **Piri's `/health`** feeds all of the above.
- **DNS repointing** lets FilOne migrate a region whose VM is unrecoverable — the one legitimate DNS intervention, and not a maintenance tool.

---

## Fleet scale

- **3–10 regions:** Quadlet/systemd + digest-pinned pull + Git-tracked release manifests. Cost near-flat.
- **~30 regions:** you want one declarative fleet view and drift detection. Inflection point for **Flux or Rancher Fleet**; also the point at which ACME rate limits and certificate persistence must already be solved.
- **300 regions:** staged/wave rollouts, per-tenant observability isolation, automated promotion, rate-limit override in place. Worst-scaling option is **serial SSH push with no reconciliation**; second-worst is **N hand-managed K3s control planes**.

---

## Observability (requirement 3)

Platform-independent; not a differentiator. Run **Grafana Alloy** on the box. Piri pushes **OpenTelemetry** via OTLP (`[[telemetry.metrics]] endpoint = "http://collector:4317"`), so terminate OTLP in Alloy rather than scraping, then `remote_write`/OTLP to **Grafana Cloud** and logs to **Loki**. Alloy's WAL buffers through outages. **Per-region tenant IDs and scoped tokens** so no partner reads another's telemetry.

Alert on: **proof set in fault state**, **no proofs submitted in a proving period**, `piri_datadir_free_bytes`, and `chain_current_epoch` vs `next_challenge_window_start_epoch` drift. PDP failure means slashing, so proof-fault alerts are pageable.

Add for requirement 4: **blackbox_exporter** probing each region's 443 endpoint, alerting on `probe_ssl_earliest_cert_expiry - time()` (21 days for 90-day certs) and on endpoint availability. Also alert on **cutover failures** and on **reboot duration exceeding expected bounds** — the latter is your early warning that reboots are drifting outside client retry tolerance.

Certificate monitoring is load-bearing rather than hygienic under on-box issuance, because TLS-ALPN-01 couples renewal to serving health: a bad deploy that breaks the proxy also blocks renewal, and the failure is silent until expiry. Two additions:

- **Probe from more than one geographic location.** A single probe cannot detect the MPIC failure mode, where the endpoint is reachable from your vantage point but not from enough of the CA's. Multi-region probing is the only early warning that a partner has introduced geo-filtering.
- **Certificate Transparency monitoring** (crt.sh or certspotter) alerting on any `*.fil.one` certificate FilOne's pipeline did not originate. This is identical across challenge types and so is not a differentiator, but it is the detection path for a partner minting certificates for their own hostname — which the threat model concedes they can always do.

---

## Comparison matrix (1 = poor, 5 = excellent)

R4 here means _zero-downtime application upgrades_, which is the achievable part; OS reboots are ~30–90s for every option without exception.

| Option                                     | R1 Zero-touch onboarding       | R2 Timely fixes                   | R3 Observability                 | R4 Zero-downtime app upgrades                                                       | R5 PDP-gated upgrades                 | R6 Immutable IaC                  | R7 Bundle pinning                 | Maintenance burden                    |
| ------------------------------------------ | ------------------------------ | --------------------------------- | -------------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------- | --------------------------------- | --------------------------------- | ------------------------------------- |
| **Podman Quadlet + systemd on FCOS/bootc** | 5 (pre-baked image + Ignition) | 4 (registry pull; push available) | 5 (Alloy)                        | **3** (auto-update restarts in place; needs proxy + blue/green units + flip script) | 5 (ExecStartPre gate)                 | 5 (bootc OCI OS + Quadlet)        | 5 (digest-pinned OS + containers) | Low                                   |
| **Plain systemd + Go binaries**            | 5                              | 4                                 | 5                                | **3** (socket activation possible; custom work)                                     | 5                                     | 3 (needs config mgmt for drift)   | 4                                 | Very low                              |
| **Docker Compose + Kamal**                 | 4                              | 4 (SSH push viable)               | 5                                | **5** (kamal-proxy: health-gated cutover + connection draining, native)             | 4 (pre-deploy hooks)                  | 3 (Compose pinning; mutable host) | 4                                 | Low–medium                            |
| **Single-node K3s + Flux**                 | 4 (K3s baked into image)       | 5 (pull reconciliation)           | 5                                | **5** (rolling update, 2 replicas, PDB, preStop — native)                           | 4 (suspend/Kured + custom controller) | 5                                 | 5 (Helm/OCI pinning)              | High (control-plane churn)            |
| **Talos + K3s/K8s**                        | 4                              | 4                                 | 5                                | **5** (as K3s)                                                                      | 3 (no native app-aware gate)          | 5 (A/B atomic OS)                 | 5                                 | Medium–high                           |
| **Balena managed fleet**                   | 5 (boot image, phones home)    | 5 (managed pull, delta OTA)       | 4 (custom path to Grafana Cloud) | **3** (supervisor restarts containers in place)                                     | 5 (update locks)                      | 4                                 | 4 (release pinning)               | Low (vendor + added trust dependency) |

**The tension this exposes, stated plainly.** Kamal and K3s now score highest on requirement 4 because both ship health-gated cutover as a native, off-the-shelf feature. Quadlet does not: `podman auto-update` restarts the unit in place. Bringing Quadlet to parity means a proxy unit, a blue/green unit pair, and a flip script — modest, well-understood work, but it _is_ custom, in a project that explicitly prefers off-the-shelf.

Three ways to resolve this, depending on how hard requirement 4 really is:

1. **If zero-downtime app upgrades are aspirational** and "outage within client retry tolerance" is acceptable: keep Quadlet, accept in-place restarts (a few seconds), and skip the blue/green machinery entirely. This is the simplest system and probably the right call for the first regions.
2. **If they are a hard requirement:** either add the proxy + blue/green layer to Quadlet (~a day's work, then maintained forever), or **use Kamal**, which gives it natively at the cost of weaker immutability (R6) and a mutable host. A hybrid also works: Quadlet on an immutable OS for everything, with a Caddy or Traefik unit doing label-based backend discovery to automate the cutover.
3. **If VM-level failure tolerance is also required:** none of these suffice, and you need the two-VM + floating-IP conversation instead.

Recommendation stands on Quadlet, with option 1 for the first regions and a deliberate decision at the point where a customer SLA forces option 2.

---

## Recommended architecture

1. **OS:** FCOS or bootc, digest-pinned, shipped as a pre-baked qcow2/ISO with embedded Ignition. Updates staged but not auto-applied; reboot gated by `pdp-gate` and batched into scheduled windows. **No package layering, no kexec, no live kernel patching** (none available — see Host OS updates).
2. **Runtime:** Podman **Quadlet** units for Piri, Ingot, optional Postgres, Alloy, **and a separate long-lived reverse-proxy unit owning :443**. Images digest-pinned and cosign-signed.
3. **Networking:** partner supplies a public IP and opens inbound 443. FilOne creates `{region}.s3.fil.one`. Management path outbound-only, plus a Tailscale/Headscale overlay for break-glass.
4. **TLS:** per-region certs via **TLS-ALPN-01 on port 443**, issued on-box by the Caddy proxy unit. No DNS credentials, no acme-dns, no CNAMEs. CAA at `fil.one` pinning CA and `validationmethods=tls-alpn-01`; DNSSEC on; **certs and ACME account state persisted across rebuilds**; ARI renewals; 90-day certs (not the 6-day profile). Per-region DNS-01 fallback available for partners who cannot expose 443 globally.
5. **Upgrades:** `pdp-gate` → start new version alongside old → health-check → proxy cutover → drain old → stop. **No DNS involvement.** OS reboots batched into rare windows, also `pdp-gate`d.
6. **State:** SQLite where possible; if Ingot forces Postgres, one container with wal-g/pgBackRest → partner S3 and rehearsed PITR. **Backward-compatible migrations enforced in CI.**
7. **Secrets:** central Vault/OpenBao + Vault Agent into tmpfs; **funded wallet key off-box behind `piri-signing-service`**.
8. **Observability:** Alloy → Grafana Cloud/Loki/Prometheus, per-region tenancy, pageable proof-fault alerts, blackbox availability and cert-expiry probes.
9. **Release bundle:** signed OCI **release manifest** pinning OS digest plus every container digest, promoted staging→prod as one unit; Renovate proposes bumps into staging only.
10. **Partner hardware spec must include:** disks, a public IP, inbound 443, **and RAM/CPU headroom for ~2× the application during cutover**.

Example Quadlet unit (`/etc/containers/systemd/piri.container`):

```ini
[Unit]
Description=Piri storage node
After=network-online.target
Wants=network-online.target

[Container]
Image=registry.filone.io/piri@sha256:PINNED_DIGEST
ContainerName=piri
AutoUpdate=registry
PublishPort=127.0.0.1:3000:3000
Volume=/data/piri:/data/piri:Z
Notify=true
HealthCmd=/usr/local/bin/piri status
HealthInterval=30s

[Service]
Restart=always
RestartSec=5
ExecStartPre=/usr/local/bin/pdp-gate

[Install]
WantedBy=multi-user.target
```

Note the proxy is deliberately a **separate** unit binding :443 and forwarding to `127.0.0.1:3000`, so restarting Piri never closes the public listening socket.

The proxy unit also owns certificate lifecycle. Caddy does TLS-ALPN-01 by default, so no challenge configuration is needed — only a persisted data volume:

```ini
# /etc/containers/systemd/proxy.container
[Unit]
Description=Edge proxy (TLS termination, ACME, backend cutover)
After=network-online.target
Wants=network-online.target

[Container]
Image=docker.io/library/caddy@sha256:PINNED_DIGEST
ContainerName=edge-proxy
PublishPort=443:443
Volume=/etc/filone/Caddyfile:/etc/caddy/Caddyfile:ro,Z
# /data holds certs, the ACME account key, and ARI state.
# MUST survive A/B OS updates and VM rebuilds, or every rebuild
# burns a new certificate against the 50-per-registered-domain cap.
Volume=/var/lib/filone/caddy:/data:Z

[Service]
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```
# /etc/filone/Caddyfile
{
    email tls-ops@fil.one
    # TLS-ALPN-01 is the default and needs only :443.
    # Port 80 is deliberately not published, so HTTP-01 is unavailable.
}

us-east-1.s3.fil.one {
    reverse_proxy 127.0.0.1:8080 {
        health_uri /healthz
        lb_try_duration 5s
    }
}
```

Do not publish :80 unless you decide to accept HTTP-01; leaving it closed keeps the partner firewall requirement at a single port and removes the silent renewal-failure mode described above.

`pdp-gate` (sketch — exit code 2 fails closed; exhaustion escalates rather than proceeding):

```sh
#!/bin/sh
# Exit 0 => safe to proceed with restart/upgrade.
# Any nonzero => abort; caller retries on next timer tick.
DEADLINE=$(( $(date +%s) + 7200 ))   # escalate after ~2h

while [ "$(date +%s)" -lt "$DEADLINE" ]; do
  piri status upgrade-check
  case $? in
    0) exit 0 ;;                       # safe
    1) : ;;                            # not safe: wait
    2) logger -t pdp-gate "status indeterminate; failing closed" ;;
    *) logger -t pdp-gate "unexpected exit; failing closed" ;;
  esac
  sleep 300
done

# Do NOT proceed. Slashing risk is real; a human decides
# vulnerability-exposure vs missed-proof tradeoff.
logger -t pdp-gate "no PDP-safe window in 2h; escalating"
exit 1
```

---

## Staged recommendations

- **Regions 1–5:** plain systemd or Quadlet on a pre-baked FCOS/bootc image. Build `pdp-gate` first — highest-value, platform-independent component. Stand up central Vault/OpenBao and `piri-signing-service`. Per-region DNS, TLS-ALPN-01 via the Caddy proxy unit, CAA at the apex, cert persistence on a data volume. Alloy → Grafana Cloud with per-region tenancy. **Accept in-place restarts; skip blue/green.** No Kubernetes.
- **Regions 5–30:** Quadlet + `podman auto-update` + cosign verification + digest-pinned pull. Break-glass overlay. Signed OCI release manifest and staging→prod promotion. **Decide on the cutover question** (option 1 vs 2 above) based on whether any customer SLA requires it. File the ACME rate-limit override. Re-evaluate whether Ingot forces on-box Postgres.
- **30+ regions:** if fleet-wide declarative management becomes the bottleneck, migrate to single-node K3s + Flux, or Rancher Fleet, keeping the same containers and the same `pdp-gate`. Consider Talos as the K8s base. Note K3s also resolves the requirement-4 gap natively.
- **Whenever a partner or customer requires VM-failure tolerance:** open the two-VM + floating-IP product conversation. Do not try to solve it in the deployment layer.

---

## Bake-off plan

Three throwaway VMs with **public IPs and inbound 443**, comparing **(A) Quadlet + FCOS**, **(B) plain systemd + bootc**, **(C) single-node K3s + Flux**. Add **(D) Compose + Kamal** if requirement 4 is hard, since it is the off-the-shelf leader there. Criteria:

1. **Onboarding time** for a non-expert: minutes from "here's an image and an IP" to a proving node with valid TLS.
2. **Fix propagation:** push a signed digest bump; measure time to running on all boxes.
3. **PDP-gate correctness:** first **measure the actual proving cadence and unsafe duty cycle** for your proof sets. Then force an upgrade during a challenge window and verify deferral, then application. Verify exit code 2 fails closed.
4. **Cutover correctness:** upgrade under sustained S3 load including a large in-flight multipart upload; measure dropped connections and 5xx count. Target zero. Verify the proxy's listening socket never closes.
5. **Reboot impact:** measure actual reboot duration and whether a representative S3 client (with your customers' real retry configuration, not SDK defaults) transparently recovers.
6. **Migration compatibility:** deploy two adjacent Piri/Ingot versions concurrently against one database and confirm both function. **If this fails, zero-downtime is unachievable and the target must be renegotiated.**
7. **Failure injection:** ship a deliberately broken container image and a broken OS image; verify unattended rollback and that the old version keeps serving.
8. **Observability:** confirm proof-fault, availability, and cert-expiry alerts fire with per-tenant isolation.
9. **Secret and cert exposure audit:** confirm no durable wallet key, broad DNS token, wildcard TLS key, or ACME account key is readable on the VM.
10. **Maintenance burden:** log hours on control-plane and OS upkeep.

---

## Open questions and risks

**Load-bearing and unverified — resolve these first:**

- **Do Piri and Ingot support graceful shutdown?** Cutover depends on the old process finishing in-flight requests on SIGTERM rather than dropping them. If not, zero-downtime is unachievable regardless of platform.
- **Are Piri/Ingot schema migrations backward-compatible?** Two versions coexist during cutover. A breaking migration removes zero-downtime for that release. Needs to become a CI-enforced project invariant, not an aspiration.
- **What is the actual PDP cadence and unsafe duty cycle?** Determines whether "defer and retry" is cheap or routinely blocks security fixes.
- **Will partners provision VMs with 2× application headroom?** If not, cutover is impossible and the achievable target is a short in-place restart.

**Other:**

- **Slashing policy owner:** who decides, on what criteria, when a critical fix cannot find a PDP-safe window?
- **Will partners expose 443 globally without geo-filtering or DDoS scrubbing that strips ALPN?** This is now a partner-agreement obligation, and it is the precondition for TLS-ALPN-01. Test from multiple regions during onboarding rather than assuming.
- **Time-to-DNS-removal on partner offboarding.** This is the actual security control bounding a hostile partner's ability to mint certificates for their own hostname. Define the target, assign an owner, and rehearse it — it currently has none of the three.
- **Does Ingot terminate TLS itself anywhere?** TLS-ALPN-01 requires the terminator to serve an ALPN-selected validation certificate. If any path bypasses the Caddy unit, that path breaks issuance.
- **Wallet custody:** confirm `piri-signing-service` fully removes the funded key from the edge. If not, the risk to funds is unresolved and may argue for FilOne-owned hardware.
- **Does Ingot forge mode force on-box Postgres?** Quantify partner-readable data exposure.
- **Client retry tolerance:** measure real customer SDK retry configuration; it determines whether sub-minute reboots are actually invisible.
- **Measured reboot duration on your actual image and partner hardware.** Now load-bearing rather than informational, since it is the only reboot lever left.
- **Floating IP availability** across your target partner hosters — determines whether the two-VM option is even offerable.
- **Piri upgrade semantics:** does `piri update` replace the binary in place, and are SQLite migrations safe across restarts?
- **Inbound exposure hardening:** the appliance is internet-facing on untrusted hardware. Define the ingress baseline (rate limits, connection limits, abuse response), and consider what a partner MITM-ing their own VM's traffic means.
- **Confirm whether Piri's `deploy/full-node` HCL** is Storacha CI infra or a supported operator path.
- **Lotus dependency:** chain.love / partner Lotus availability sits in Piri's hot path; define failover.
- **cosign trust-root distribution** to bandwidth-constrained sites.

---

## Caveats

- **PDP proving cadence remains unverified.** The design deliberately avoids depending on a specific number; verify before sizing upgrade windows.
- Piri is early (v0.2.4) and Ingot's README warns it is "work in progress… Expect breaking changes." `upgrade-check` semantics and OTLP metric names may shift; pin and re-verify.
- Whether an official upstream Piri container image is published is unconfirmed; you may need to build and publish your own Piri/Ingot images.
- **"Zero-downtime" applies only to application upgrades, and only if graceful shutdown and backward-compatible migrations hold.** OS reboots (30–90s), Postgres major upgrades, and VM failure are all unavoidable downtime on a single VM. The defensible claim is: zero-downtime app upgrades, sub-minute reboots absorbed by client retries, regional outage on VM loss.
- **ACME rate limits and CA policy are moving targets.** The 50-per-registered-domain-per-7-days figure is current as of mid-2025 documentation; re-verify before a 300-region rollout and file the override early. Two dated changes affect the TLS design specifically: **MPIC quorum tightens through December 2026** (to 5 remote perspectives with ≥4 matching), progressively reducing tolerance for partner network misconfiguration under TLS-ALPN-01; and the **CA/B Forum is moving RFC 8657 processing from optional to mandatory** around September 2026. Also being discussed at the Forum is `http-validation-over-tls` — HTTP-01-style validation over port 443 with certificate verification — which if it lands would be a closer fit for this topology than either current option. Re-check before committing to a long-lived design.
- FOC/PDP is fast-moving (PDP on mainnet since May 2025, FOC mainnet January 2026). Re-check protocol claims before long-lived commitments.

---

# 2026-08-06 Claude review: the Piri upgrade path

Walking a Piri upgrade end to end through the Quadlet proposal turns up a set of concrete defects in
the sample units above. Under the constraint now stated in Requirements & Constraints, a Piri upgrade
is a stop-start of one container whose duration is set by Piri's own drain budget rather than by the
deployment platform. The units in "Recommended architecture" will not perform that stop-start
correctly as written.

## Verified against the Piri source

Checked at `storacha/piri@b94c811`:

- **`piri status upgrade-check` exists** in `cmd/cli/status/upgrade.go` with the documented exit codes
  0/1/2. It calls `client.GetNodeStatus(ctx)` against the running node and reads `UpgradeSafe`,
  derived from `IsProving`, `InChallengeWindow`, and `HasProven`. It is a point-in-time question with
  no duration argument. It is also smarter than epoch math would be: once the node has proven for the
  current window it reports safe even inside that window, which widens the usable time.
- **Piri does not implement sd_notify.** There is no `NOTIFY_SOCKET` handling and no systemd notify
  library anywhere in the tree.
- **Piri ships its own systemd installer** under `cmd/cli/setup/`. It writes units into a versioned
  directory and symlinks `/etc/systemd/system` entries at a `current` symlink. `piri update` downloads
  a new binary and explicitly does not restart the service. That answers the open question about
  `piri update` semantics, and it means the binary-plus-systemd path is upstream's supported operator
  path while the container path is ours to own and maintain.

## What one upgrade does

1. **Build and sign.** Piri main merges, CI builds `CGO_ENABLED=0`, pushes to the FilOne registry,
   cosign-signs the digest. The box enforces the signature through `/etc/containers/policy.json`, so a
   partner who swaps the image on disk gets a container that will not start.
2. **Staging.** The staging channel moves to the new digest and the staging box runs the same
   gate-and-restart path prod will run. This is the only place the upgrade mechanism itself gets
   exercised before it touches a partner.
3. **Promotion.** After soak, the pipeline moves the prod channel and publishes the signed release
   manifest. The soak duration and who may shorten it for a security fix are undefined; requirement 2's
   "hours" is soak plus gate wait, and soak is the part we control.
4. **The box learns.** See change 2 below. As written, it never will.
5. **The gate.** `pdp-gate` blocks until `upgrade-check` returns 0. At a 2880-epoch period with a
   60-epoch window the unsafe duty cycle is around 2%, so the gate usually returns immediately.
6. **The swap.** systemd stops the unit, podman sends SIGTERM, Piri drains, the new image starts,
   migrations run, the healthcheck goes green.
7. **Acceptance.** The unit being active is not acceptance. The region is upgraded when it has
   submitted one proof on the new version.

Downtime is drain plus start plus migrations plus health. With a bounded drain that is one to five
minutes. With Piri's default drain it can approach an hour. Both are longer than the 30–90s OS reboot
the research report treats as the unavoidable downtime, so for Piri the framing is inverted: the
application upgrade is the expensive event and the host reboot is the cheap one.

## Changes to make

1. **Move the gate off `piri.container`.** `ExecStartPre` runs in the start phase, which is after the
   stop. Under `podman auto-update` Piri is already stopped when the gate begins to block, so a
   two-hour wait becomes two hours of downtime inside the window the gate exists to protect. It also
   exceeds the default `TimeoutStartSec=90s`, so systemd kills the gate before it can ever return 0.
   The Requirement 5 section already says the right thing ("on the upgrade unit, driven by a
   `.timer`"); the code sample contradicts it. Mask `podman-auto-update.timer` so nothing runs ungated.

2. **`AutoUpdate=registry` and a digest-pinned `Image=` cancel each other out.** Auto-update resolves a
   tag against the registry; a digest reference has nothing to resolve. Pick one:
   - _Channel tag_ (`registry.filone.io/piri:prod`) moved deliberately by the promotion pipeline.
     Auto-update then works off the shelf. Requirement 7 still holds because the tag resolves to a
     digest at pull time and cosign binds it, but the unit file stops recording the running version, so
     report it from `podman inspect` through Piri's OTEL export.
   - _Digest in the unit_, no auto-update, plus an agent that fetches the release manifest, writes a
     Quadlet drop-in, reloads, and restarts. Purer IaC, more code we own.

   Channel tag for the first regions.

3. **Replace `Notify=true` with `Notify=healthy`.** Piri never sends `READY=1`, so the unit hangs until
   start timeout and then fails. `Notify=healthy` (podman 5.0+) ties systemd readiness to the container
   healthcheck, which is also what makes `podman auto-update --rollback` fire on a bad image. With
   `Notify=true` and a broken binary that nevertheless starts, rollback never triggers.

4. **Set both stop timeouts explicitly.** Podman defaults to 10s and systemd to 90s. Against the drain
   described in the Forrest notes, the default configuration SIGKILLs Piri mid-drain on every single
   upgrade.

5. **Bound the drain below the gap between challenge windows.** `upgrade-check` answers "safe now". A
   long drain begun shortly before the next window opens passes the gate and leaves Piri mid-shutdown
   when it should be proving. Cap the drain on our side, and ask Storacha for
   `piri status upgrade-check --duration 15m` so the gate can ask the question it actually needs
   answered.

6. **Define break-glass for exit code 2.** Exit 2 is what a wedged Piri returns, because the check
   queries the node itself. Fail-closed therefore means a broken node can never be upgraded to the
   version that fixes it. Give the override a named holder, most likely the same person as the
   slashing-policy owner in Open Questions.

7. **Do not label the proxy for auto-update.** `podman auto-update` has no per-container selector, so
   one run restarts everything labelled, Caddy included, which closes :443 and undoes the separate-proxy
   design. Upgrade the proxy as a rarer, separately gated event.

8. **Test rollback compatibility.** With in-place restarts two versions never run at
   once, so expand/contract is no longer the binding constraint. What binds is `--rollback`: it restores
   the previous image and starts the old binary against a database the new binary already migrated.
   Bake-off item 6 should test N-1 tolerance of version N's schema. Snapshot SQLite in the upgrade unit
   before the swap.

9. **Alert on the failed upgrade unit.** The `pdp-gate` sketch escalates by writing to the journal,
   which pages nobody. Add `OnFailure=` and an alert on the unit's failed state.

10. **Restate the sections that assume cutover.** Recommended architecture
    item 5 still reads "start new version alongside old → health-check → proxy cutover → drain old →
    stop", and the R4 column of the comparison matrix scores options on cutover. Both now conflict with
    Requirements & Constraints.

## Corrected units

```ini
# /etc/containers/systemd/piri.container
[Unit]
Description=Piri storage node
After=network-online.target
Wants=network-online.target

[Container]
Image=registry.filone.io/piri:prod
ContainerName=piri
AutoUpdate=registry
PublishPort=127.0.0.1:3000:3000
Volume=/data/piri:/data/piri:Z
# Piri does not implement sd_notify. Tie readiness to the healthcheck.
Notify=healthy
HealthCmd=/usr/local/bin/piri status
HealthInterval=30s
HealthStartPeriod=60s
# Bound the graceful drain so it cannot run into the next challenge window.
StopTimeout=300

[Service]
Restart=always
RestartSec=5
TimeoutStopSec=330

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/filone-upgrade.service
[Unit]
Description=PDP-gated container upgrade
OnFailure=filone-upgrade-alert.service

[Service]
Type=oneshot
TimeoutStartSec=8000
ExecStartPre=/usr/local/bin/pdp-gate
ExecStartPre=/usr/local/bin/snapshot-piri-db
ExecStart=/usr/bin/podman auto-update --rollback
```

```ini
# /etc/systemd/system/filone-upgrade.timer
[Timer]
OnBootSec=10min
OnUnitInactiveSec=15min
RandomizedDelaySec=5min

[Install]
WantedBy=timers.target
```

Requires `systemctl mask podman-auto-update.timer`, otherwise the stock daily timer runs the same
upgrade without the gate.

## Still open

- What is Piri's configuration key for the shutdown drain timeout, and what value fits inside the gap
  between challenge windows?
- What is the soak duration between staging and prod, and who may shorten it for a security fix?
- Does Piri run SQLite migrations at startup, and does version N-1 tolerate version N's schema?
- Do we publish our own Piri image, or does Storacha publish one we can consume? The upstream operator
  path is binary plus systemd, so this is a maintenance commitment either way.

---

# 2026-08-06 Running a Non-Archival, Live-Tip Filecoin RPC Node: Lotus vs Forest

## TL;DR

- **Both Lotus and Forest can do exactly what you want** — bootstrap from a snapshot, then follow the live tip and serve a Lotus-compatible JSON-RPC + Eth JSON-RPC API with only 1–2 days of chain retention — and for a _read-only, chain-following RPC node_, **Forest is the better default**: it needs far less RAM/CPU (ChainSafe's baseline is "16 GiB, 4 cores" vs Lotus's 32 GiB / 8 cores), imports/starts faster, auto-downloads snapshots, and per the 2026 conformance reports it passes Lotus-parity tests on essentially all read and `eth_*` methods you'd use.
- **Snapshot syncing does NOT leave you behind the tip.** A snapshot is only the _initial bootstrap_; snapshots are regenerated hourly and store the last 2000 tipsets, so a freshly loaded one is only ~1–2 hours old. After loading, the node validates forward and then tracks head, sitting effectively 0–1 tipsets behind in steady state. Your "must be within 1 epoch of the tipset" requirement is met in normal operation by both nodes.
- **Recommended build:** an 8-core modern x86-64 box, 32 GiB RAM, and a 512 GiB–1 TB NVMe SSD, running **Forest** as the primary node with **Lotus (or a hosted Glif endpoint) as fallback**, behind a health-checking reverse proxy that compares `ChainHead` height to wall-clock-expected height. Read the F3-finalized tipset (or 1–5 epochs back from head) for anything reorg-sensitive.

## Key Findings

- **Retention/pruning is a first-class feature in both.** Lotus uses SplitStore (default since v1.21.0) with a `discard` cold-store mode explicitly recommended for "small nodes that are simply watching the chain." Forest runs an automatic garbage collector and sizes disk as **"200 GiB + 5 GiB per day of retention"** (per ChainSafe's hardware docs; ~2 GiB/day and half the disk if GC is disabled). Its minimal-retention RPC profile fits in 256 GiB.
- **Forest is dramatically lighter.** ChainSafe's July 17 2026 head-to-head (same live RPC traffic, ~45 req/s) shows Forest at ~10–15% CPU and ~8–10% memory vs Lotus ~45–55% CPU and ~25–30% memory, with lower tail latency (P99 ~200–350 ms for Forest vs ~240–600 ms for Lotus). Forest's snapshot _generation_ benchmark ran ~28 min at ~11 GiB RAM.
- **No GPU is needed** for a chain-following/RPC node in either implementation. GPUs and Intel SHA extensions matter only for storage-provider sealing/proving (PreCommit2/Commit2/WindowPoSt), which neither your use case nor Forest does at all — Forest has no SP/sealing functionality.
- **RPC coverage is high but not 100% identical.** Forest supports `/rpc/v0` (legacy), `/rpc/v1` (production), and an experimental `/rpc/v2`; its Eth (`eth_*`) surface is essentially fully conformance-tested against Lotus. Some `Wallet*`, `Mpool*Push`, and `Net*` methods are functional but "not conformance-tested," and a handful are Forest-specific (`Forest.*`). The interoperability standard is **FRC-0104 (OpenRPC "Common Node API")**, co-driven by ChainSafe with a public conformance test suite.
- **Network-upgrade readiness is the real recurring operational risk, and both handle it well.** The current network is **nv28 "Fire Horse" (丙午)**, which went live on mainnet at **Epoch 6052800 on 2026-05-27T14:00:00Z** (Lotus v1.36.0). All implementations, including Forest, shipped nv28 releases for Calibration ahead of the upgrade, and Forest has shipped mandatory releases for each recent nv (nv24 Tuk Tuk, nv25 Teep, …). A node not upgraded before the upgrade epoch forks off the network — this is your #1 maintenance duty regardless of implementation.

## Details

### A. Hardware / system requirements

**Forest (recommended for this use case)** — from ChainSafe's official hardware page:

- **RPC node, low-traffic minimal retention:** 4-core (min) / 6-core (rec) CPU; **8 GiB (min) / 16 GiB (rec) RAM**; **256 GiB SSD, high-IOPS/NVMe recommended.** (Filecoin Docs summarize Forest's baseline as "low hardware requirements (16 GiB, 4 cores)… does not support storage, retrieval or mining.")
- **RPC node, 2 months retention, up to high traffic:** 6–8 core CPU; 16–32 GiB RAM; 500 GiB SSD.
- Disk-sizing rule (verbatim): **"an RPC node would require 200 GiB + 5 GiB per day of retention of disk space… if you disable GC, you can cut the disk space requirements in half… (it's growing by ~2 GiB per day)."** For your 12–36h retention target, the 256 GiB "minimal retention" profile is ample.
- "Network upgrades can require more memory" — migration spikes are the RAM high-water mark, so size RAM for the migration, not the steady state.

**Lotus** — from Lotus docs and operator reports:

- **Minimum:** 8-core CPU, **32 GiB RAM**, SSD. Models with Intel SHA extensions (AMD Zen+, Intel Ice Lake+) "significantly speed things up" but are not mandatory for a non-SP node.
- Historically RAM-hungry; operators commonly provision 32+ GiB and add swap. Per the Lotus v1.36.0 (NV28) release notes, the network-upgrade migration is light: **"The migration on the upgrade epoch is expected to take approximately 1 minute on a node with a NVMe drive and a newer CPU… RAM usage is expected to be under 20GiB RAM for both the pre-migration and migration."**
- **Disk:** raw chain grows ~38 GiB/day per Lotus docs (higher than Forest's ~2–5 GiB/day figure because of datastore amplification), so pruning/SplitStore is essential. A fully GC'd SplitStore hotstore is ~295 GiB; the working set is documented as 60–275 GiB. A `discard` cold store avoids a large cold store entirely — the appropriate choice for you.

**Both:** CPU is 64-bit x86-64 (or ARM64); **no GPU**; a public IP with the libp2p port open (default TCP 1347 Lotus / configurable Forest) improves peering but is not strictly required with good outbound connectivity. Bind the RPC/API port (Lotus 1234) to localhost or behind a protected proxy.

**OS / Docker:** Both are Linux-first (Ubuntu/Debian widely used) and build on macOS. Forest ships official multi-arch Docker images at `ghcr.io/chainsafe/forest:latest` and treats Docker as the primary path (Linux/macOS/Windows). Lotus provides Docker images but emphasizes released binaries / build-from-source.

### B. Provisioning and sync time

**Snapshot sizes (live figures from ChainSafe's Forest Archive, Aug 6 2026):**

- Lightweight/bootstrap snapshot (v2, per FRC-0108, ~last 2000 tipsets): **~79–80 GB compressed** `.forest.car.zst` (e.g. height 6,256,320 = 79.42 GB). The old "~100 GB lightweight / ~600 GB full" numbers are now stale — the compressed lightweight snapshot is ~80 GB even though the chain is now ~6.25M epochs.
- Uncompressed is roughly ~150–160 GB (inferred at ~2× zstd ratio; not an official 2026 figure).
- Hosted on **ChainSafe Forest Archive (`forest-archive.chainsafe.dev`), backed by Cloudflare R2 with no egress fees**, regenerated ~hourly; a legacy filops S3 endpoint (`snapshots.mainnet.filops.net`) also exists. Both Lotus and Forest can read `.zst`/`.forest.car.zst`.

**Provisioning wall-clock (modern 8-core NVMe box):**

- **Forest:** `forest` with default/`--auto-download-snapshot` downloads the snapshot automatically and can even serve blocks in-place from the indexed CAR without a separate DB import. End-to-end from `docker run` to serving RPC at head is typically well under an hour, dominated by the ~80 GB download. Import itself is "a few minutes."
- **Lotus:** you manually `aria2c -x5` the snapshot, then `lotus daemon --import-snapshot`. Import historically ran ~4 min at ~200 MiB/s when snapshots were far smaller; at ~80 GB expect tens of minutes, plus a one-time proof-parameters download (a few GB) and datastore expansion (the CAR data is duplicated into the DB).
- **Catch-up after import:** the snapshot is only ~1–2 hours behind head (hourly regeneration; stores the last 2000 tipsets ≈ up to ~16h). The node fetches headers to head, then validates forward. Lotus docs state ~4 seconds/tipset; catching up a few hundred-to-few-thousand tipsets is minutes on NVMe. Steady-state lag afterward is 0–1 tipsets.
- **Full sync from genesis** is "generally infeasible" — Lotus docs note it takes ~4s/tipset (≈1 month for 700k tipsets), and the chain is now ~6.25M epochs. Nobody does this for a non-archival node; snapshot bootstrap is the only practical path.

### C. Restart time and crash recovery

- **Clean process restart:** Both reopen their embedded DB (Lotus = Badger; Forest = ParityDB) and resume from the persisted head. Catch-up is roughly one tipset (~30s of chain) per 30s of downtime — a 1-minute restart is back at head in seconds; a 1-hour outage means ~120 tipsets (a few minutes at 4s/tipset, faster on NVMe).
- **DB open time:** ParityDB and Badger both open in seconds-to-tens-of-seconds. Lotus additionally "warms up" the SplitStore hotstore on first start after enabling it (loading current-head headers and state roots).
- **Unclean shutdown / corruption:** Lotus's Badger has a documented history of pain — snapshot-import crashes, "another process is using this Badger database," and content-integrity mismatches appear in Lotus GitHub issues (mostly 2020–2021 vintage; Lotus has hardened since). ParityDB is purpose-built for blockchain workloads and is generally robust. The universal remedy for either node when the store is corrupt or suspect: clear the datastore and re-import a fresh snapshot.
- **Re-import vs catch-up threshold:** because a fresh snapshot only covers ~2000 tipsets (~16h) and regenerates hourly, treat **> ~12–24h of downtime** (or any DB corruption) as "re-bootstrap from snapshot" — it's faster and safer than syncing a large gap.
- **HA / zero-downtime:** the recommended pattern is **N+1 redundancy** — two independent nodes behind a reverse proxy/load balancer that health-checks each backend by polling `Filecoin.ChainHead` and comparing tipset height to wall-clock-expected height, evicting any stale backend. Lotus additionally offers `lotus-gateway`, a hardened, rate-limited read/`MpoolPush` fronting layer (this is what Glif runs). There is no in-process hot-restart; achieve zero downtime via rolling restarts across the pool.

### D. Operational pros and cons for this use case

**Maturity / track record:** Lotus is the Go reference implementation by the Filecoin Core Devs (originally Protocol Labs); protocol features land there first, and it's run by most SPs and infra providers (Glif/Protofire's public RPC runs Lotus + lotus-gateway). Forest (ChainSafe, Rust) is production-grade for chain-following and RPC, hosts the network's canonical snapshot service, and is used as a lighter alternative — but it deliberately has **no storage-provider/sealing stack**, and its block-production path is untested.

**Resource efficiency:** Forest wins clearly (see Key Findings — ~3–5× less CPU/RAM, faster snapshot ops, lower tail latency in ChainSafe's July 2026 benchmark). These exact multipliers are ChainSafe-reported, but they align with the published hardware minimums (Forest 8–16 GiB vs Lotus 32 GiB RAM) and Forest's Rust/ParityDB design.

**RPC completeness / stability:** Both expose Lotus-compatible JSON-RPC v0/v1 and the `eth_*` FEVM API (Lotus needs `LOTUS_FEVM_ENABLEETHRPC=1`). Forest's Jan–Aug 2026 API parity reports show the full `Filecoin.Eth*` set and the vast majority of `Filecoin.State*`/`Chain*` methods as conformance-tested-and-passing; some `Wallet*`, `Mpool*Push`, and `Net*` methods are functional but untested, plus Forest-specific `Forest.*` methods. Risk: subtle behavioral differences on less-common methods — mitigate by testing your client's exact method set against Forest before cutover.

**Network-upgrade readiness:** critical. Both ship mandatory releases ahead of each nv upgrade epoch; missing it forks your node off. Historically both have been timely (Forest shipped nv24/nv25/…/nv28; Lotus is the reference and sets the epochs). Subscribe to `#fil-forest-announcements` / Lotus release notifications and the `filecoin-project/community` "Network Updates" discussion.

**Observability:** Lotus exposes Prometheus metrics at `/debug/metrics`, commonly paired with Grafana. Forest supports Prometheus metrics, structured logging, optional Grafana Loki telemetry (`--loki`), and built-in health-check endpoints; ChainSafe publishes a public RPC-performance Grafana dashboard. Both provide sync CLIs (`lotus sync wait`, `forest-cli sync status`).

**Licensing:** Both are permissively dual-licensed **Apache-2.0 / MIT**.

**Does Forest still need Lotus?** No — for chain sync, validation, snapshot generation, and RPC serving, Forest is standalone and does not depend on a Lotus node; it hosts its own snapshots and is "an operational consensus node." You only need Lotus for SP/sealing features.

### E. Alternatives to self-hosting

- **Glif public RPC** (`api.node.glif.io` with `/rpc/v0` and `/rpc/v1`, plus `wss://wss.node.glif.io`): load-balanced hosted Lotus behind lotus-gateway; read-only + `MpoolPush`/`EthSendRawTransaction`; all Filecoin + Eth methods. Operated by Protofire/Infinite Scroll — Filecoin is "a network now handling 4+ billion node requests a month," and Protofire reports "99.95%+ RPC uptime," with the public endpoint serving "10M+ requests a day." **The public endpoint "guarantees only 2000 of the latest blocks"** — the same lightweight-snapshot limitation you have, so it _does_ expose live-tip data. Dedicated (archival) nodes are available via request form.
- Other providers exist (Ankr, Chainstack, Tatum, etc.) with their own rate limits/tiers.
- **When to self-host:** guaranteed rate limits/latency, private methods, custom retention, or control over which epoch you read. **Best-practice hybrid:** run your own Forest/Lotus node AND wire a hosted endpoint (e.g., Glif) as automatic fallback in the client, so a restart/resync doesn't take your app down. Both serve live-tip data, so read failover is seamless.

### F. Gotchas for "closely following the tip"

- **Monitoring staleness:** poll `Filecoin.ChainHead`, take the tipset `Height`, and compare to expected epoch = `(now − genesis_timestamp) / 30`. Filecoin genesis is 2020-08-24 22:00 UTC; epoch duration is 30s. If head is more than a few epochs below expected, you're lagging. Forest exposes `Forest.SyncStatus`; Lotus has `lotus sync wait`.
- **Null tipsets / null rounds:** some epochs legitimately produce no block. Your node can look "1 behind" when it's at the true head, because the expected-height formula assumes a block every epoch. Don't alarm on a 1–2 epoch gap; alarm on a sustained/growing gap.
- **`ChainNotify` vs polling:** `ChainNotify` returns a subscription channel of head changes (first message is the current head, then apply/revert events) — the correct way for a client to _follow_ the tip and _see reorgs_ (revert then apply). Use `ChainNotify` for live following; use `ChainHead` for health checks.
- **Reorgs and finality:** Filecoin's classic Expected Consensus fork-choice tolerates reorgs up to the finality threshold (900 epochs). In practice tip reorgs are shallow (a few epochs), but **the immediate head is not final**. **F3 (Fast Finality, FIP-0086)** is now live on mainnet: per FIP-0086, "F3 is expected to finalize tipsets within tens of seconds during regular network operation, compared to the current 900-epoch finalization delay," and per FilOz "in the vast majority of cases, it finalises tipsets within 2 epochs" — a 450× speedup from the old ~7.5-hour (900-epoch) EC finality. For a client reading live data: reading the exact head is fine for "latest-ish" views but is reorg-exposed; for correctness read the **F3-finalized tipset** (`Filecoin.ChainGetFinalizedTipSet` / F3 certificate methods) or stay a few epochs back. A common safe compromise is 1–5 epochs behind head, or the finalized tipset for anything money-related.
- **Peering:** both auto-connect to bootstrap peers (ChainSafe publishes Forest mainnet bootstrap nodes; status at probelab.io). Insufficient peering is the most common cause of a node silently falling behind — target a healthy peer count (Forest's default target is 75 peers) and prefer a public IP / open libp2p port for inbound peering.

## Recommendations

**Primary recommendation: run Forest as the chain-following RPC node, with a fallback.** For a non-archival, live-tip, read-only RPC server it is the more resource-efficient and operationally simpler choice (auto-snapshot download, single binary + Docker, low RAM/CPU, GC on by default, passes Lotus Eth/read conformance). Keep a Lotus node OR a Glif endpoint as fallback, because Lotus is the reference and a few uncommon methods are only conformance-hardened there.

**Staged plan:**

1. **Validate method coverage first.** Enumerate the exact RPC methods your clients call and cross-check them against the current Forest API parity report. If every method is "✅ tested" (or you verify the "➖ untested-but-functional" ones behave correctly for you), proceed with Forest. If you depend on a method Forest lacks or that differs, make Lotus the primary.
2. **Provision one node.** 8-core x86-64, 32 GiB RAM, 512 GiB–1 TB NVMe, Ubuntu LTS. For Forest: `docker run ghcr.io/chainsafe/forest:latest --auto-download-snapshot`; enable Prometheus/Loki/health endpoints. For Lotus: enable SplitStore with `ColdStoreType="discard"`, set `LOTUS_FEVM_ENABLEETHRPC=1` for `eth_*`, `aria2c -x5` the snapshot, then `--import-snapshot`.
3. **Configure retention to your 12–36h target.** Forest: rely on default GC (256 GiB is plenty). Lotus: `discard` SplitStore + periodic `lotus chain prune hot` keeps the footprint minimal.
4. **Add HA before production.** Two nodes behind a reverse proxy that health-checks on `ChainHead` height vs wall-clock-expected height (allow a 1–2 epoch null-round tolerance); optionally front with `lotus-gateway`. Wire a hosted endpoint (Glif) as last-resort fallback in the client.
5. **Set client read semantics.** Use `ChainNotify` to follow the tip; read the F3-finalized tipset (or 1–5 epochs back) for anything that must not be reverted.
6. **Operate the upgrade calendar.** Track the `filecoin-project/community` network-update discussion and the implementation announcement channels; upgrade both nodes before each nv upgrade epoch. This is the single most important recurring task.

**Thresholds that change the recommendation:**

- Need any storage-provider / sealing / block-production feature → **Lotus** (Forest doesn't do these).
- Need deep historical state (archival) later → different sizing entirely (multi-TB, SplitStore `universal` or a dedicated archival node); not this build.
- Clients need a method Forest doesn't conformance-test and you can't verify it → **Lotus primary**.
- Ops burden not justified by your traffic/latency/privacy needs → use **Glif/hosted** and skip self-hosting (both expose live-tip data, limited to ~2000 recent blocks, which matches your retention need anyway).

## Caveats

- The headline efficiency figures (Forest ~3–5× lighter; the P50/P95/P99 latencies; ~28 min Forest vs longer Lotus snapshot generation) come from **ChainSafe's own benchmarks** (July 17 2026 RPC comparison; snapshot comparison). They are internally consistent and align with published hardware minimums, but they are vendor-reported; independent third-party benchmarks are scarce. Validate on your own hardware and traffic.
- Sources conflict on Lotus's peak RAM during snapshot _generation_: ChainSafe's snapshot-comparison page implies a very high peak, while **official Lotus docs state creating a CAR snapshot "will take over an hour… and use around 100GB of RAM."** Either way it is far heavier than Forest's ~11 GiB — but treat the exact Lotus peak as uncertain.
- Several Lotus numbers are **long-standing doc figures that may be stale**: "~4 s/tipset," "~38 GiB/day chain growth," "~295 GiB GC'd hotstore," and the "~4 min snapshot import" (from a 2022–23 PR, before snapshots reached ~80 GB). Treat them as order-of-magnitude, not freshly benchmarked.
- Uncompressed snapshot size (~150–160 GB) is **inferred** from the ~2× zstd ratio, not an official 2026 figure.
- The current network version is **nv28 "Fire Horse"** (mainnet Epoch 6052800, 2026-05-27); upgrades happen periodically and can shift memory requirements and add methods. Re-check hardware and RPC parity at each upgrade.
- Forest's `/rpc/v2` is explicitly **experimental**; use `/rpc/v1` in production.
- Badger-corruption anecdotes cited are from older Lotus versions (2020–2021); Lotus has hardened since, but the re-import-from-snapshot recovery playbook remains the reliable mitigation for both nodes.

---

# 2026-08-06 Claude review: provisioning and persistent storage

The persisted volume listed under Components & Dependencies interacts with the immutable-OS choice in
exactly one place, and the sample units above get that place wrong. This section covers where
operator-supplied storage mounts on an image-based OS, and how a fresh VM gets from "here is an image"
to a running appliance.

## What immutability covers

On Fedora CoreOS and bootc, only `/usr` is read-only, and it is a per-deployment ostree bind mount.
Two directories are writable and shared across every deployment, so they survive both an A/B update
and a rollback:

- `/etc` — writable, three-way merged on update
- `/var` — writable, persistent, not part of the OS image at all

Block-device mounts sit outside the image model. The OS image is a filesystem tree; a mount is a
kernel operation on a directory. Nothing about atomic updates constrains it, so the persisted volume
requirement costs nothing in immutability.

**`Volume=/data/piri:/data/piri:Z` will not survive an OS update.** It appears in both the original and
the corrected `piri.container` above. `/data` is a new top-level directory in the deployment root, so
ostree does not carry it into the next deployment, and on images with composefs the root is read-only
and creating it fails outright. Every persistent path must be under `/var`. FCOS documents `/var/mnt/`
as the location for additional filesystems. The Caddy unit is already correct at
`/var/lib/filone/caddy`.

## Where state lives

Mount the operator volume at `/var/mnt/filone` and point every stateful container at a subdirectory:

| Path                       | Contents                                              |
| -------------------------- | ----------------------------------------------------- |
| `/var/mnt/filone/piri`     | SQLite databases, `service.pem`                       |
| `/var/mnt/filone/postgres` | Ingot's data directory, if forge mode forces Postgres |
| `/var/mnt/filone/caddy`    | Certificates, ACME account key, ARI state             |
| `/var/mnt/filone/alloy`    | Telemetry WAL                                         |

The certificate-persistence control in Required controls ("persist certificates across rebuilds on a
data volume") and the operator volume requirement are the same requirement, stated in two places.

## First-boot disk setup

Ignition runs in the initramfs on first boot, before the real root is mounted, and handles
partitioning, filesystem creation, LUKS, and mount units. Butane source:

```yaml
variant: fcos
version: 1.5.0
storage:
  filesystems:
    - device: /dev/disk/by-id/virtio-filone-data
      format: xfs
      label: FILONE-DATA
      # Never reformat. A rebuilt VM must adopt the existing volume.
      wipe_filesystem: false
systemd:
  units:
    - name: var-mnt-filone.mount
      enabled: true
      contents: |
        [Unit]
        Description=FilOne control-plane data volume
        [Mount]
        # Format by device path once; mount by label forever.
        What=/dev/disk/by-label/FILONE-DATA
        Where=/var/mnt/filone
        Type=xfs
        Options=x-systemd.growfs,x-systemd.device-timeout=30s
```

Three details carry weight:

**`wipe_filesystem: false` is load-bearing.** A VM rebuilt from a fresh image re-runs Ignition against
a volume that already holds Postgres, the Caddy ACME account key, and Piri's SQLite. With this set,
Ignition adopts a matching existing filesystem and refuses to boot if it finds one it does not
recognise. The operator hardware spec therefore has to say **raw, unformatted volume**, because an
operator-preformatted ext4 volume fails provisioning rather than being silently adopted.

**Format by device, mount by label.** Operator volumes surface under names nobody can predict
(`/dev/vdb` on one hypervisor, `/dev/sdb` on another, and ordering shifts when a disk is added). The
label removes the problem after the volume is first prepared; see Disk identity below for how the
first preparation finds the device.

**The mount is deliberately not wanted by `local-fs.target`.** Each stateful unit's
`RequiresMountsFor=` creates Requires plus After on the mount unit and pulls it in on demand, which
keeps the whole thing out of early boot and avoids ordering cycles with the preparation step.

## The failure mode to guard against

If the mount is missing or late, Podman creates the host directory itself, Postgres runs `initdb` onto
the boot disk, and the next successful boot mounts the volume over the top. Two half-populated data
directories, no error anywhere. Guard it in every stateful unit; Quadlet passes `[Unit]` through
unchanged, so this goes directly into `piri.container`:

```ini
[Unit]
RequiresMountsFor=/var/mnt/filone
# RequiresMountsFor falls back to the nearest parent mount, so assert the real
# thing. Assert rather than Condition: a missing volume must fail loudly, not
# silently skip the unit.
AssertPathIsMountPoint=/var/mnt/filone
```

Mounting by a label we assigned ourselves is what makes the assert sufficient. No sentinel file is
needed, because nothing but our own volume can satisfy `What=/dev/disk/by-label/FILONE-DATA`.

## Layout: the volume carries only unrecoverable state

FCOS supports putting all of `/var` on a separate device, which would give persistence to everything
stateful for free, container image store included.

Mount only `/var/mnt/filone` instead. The reason is the recovery story: a box with `/var` on the
operator volume does not boot when that volume fails to attach, on hardware FilOne cannot touch. A box
missing only the data volume boots, fails its asserts, and pages with a diagnosable host. Container
images are re-pullable and belong on the boot disk; the volume carries the state that cannot be
regenerated.

## SELinux, growth, encryption at rest

**SELinux.** `:Z` relabels the bind-mounted host directory to a private container label, and labels
persist in xattrs on xfs and ext4. Use `:Z` only for single-consumer directories. Two containers
sharing one directory need `:z`, because `:Z` will relabel it out from under the other.

**Growth.** `x-systemd.growfs` in the mount options covers the operator enlarging the volume later.

**Encryption.** The volume sits on partner-controlled storage, and Ignition supports LUKS with Clevis
network binding. Binding to a FilOne-operated Tang server means the volume unlocks only when the box
can reach FilOne, so a disk pulled from the rack is inert. The limit is worth stating: it does nothing
against a hypervisor reading the unlocked volume or process memory, which is the threat Trust boundary
and secrets already names. It closes disk theft and decommissioning hygiene.

**Other image-based OSes.** bootc has no Ignition, so the mount unit ships inside the image and the
one-time preparation needs its own first-boot oneshot. Flatcar uses Ignition but its writable-path set
differs, so the mountpoint needs re-checking if it stays in the bake-off.

## Provisioning a fresh VM

### The operator never authors an Ignition config

Ignition is a build artifact produced by FilOne CI and opaque to the operator. One enrollment token
varies between regions. Disk identity is resolved on the box, so it never appears in the config.

| Artifact        | Platforms                         | How Ignition arrives                                             | Operator action                          |
| --------------- | --------------------------------- | ---------------------------------------------------------------- | ---------------------------------------- |
| **qcow2 / raw** | Proxmox, libvirt/KVM, OpenStack   | QEMU `fw_cfg` (`opt/com.coreos/config`), config drive, user data | Upload our disk image, attach our `.ign` |
| **OVA**         | VMware                            | `guestinfo.ignition.config.data` (base64)                        | Import the OVA, paste one property       |
| **Live ISO**    | Anything that can only boot media | Embedded in the ISO by us                                        | Boot it, paste nothing                   |

The disk-image artifacts are the better default. They have no install step, so the boot disk is the
image itself and the destructive "which device do we install to" question disappears. The ISO is for
operators who can only hand a VM a boot medium.

### What FilOne publishes

Per release, not per region:

```bash
butane --pretty --strict appliance.bu > appliance.ign

# ISO variant, for operators who can only boot media.
coreos-installer iso customize \
  --dest-device /dev/vda \
  --dest-ignition appliance.ign \
  --pre-install  ./guard-install-target.sh \
  --post-install ./prepare-data-volume.sh \
  -o filone-appliance-1.4.0-virtio.iso \
  fedora-coreos-live.x86_64.iso
```

`--dest-device` has to be a name we can predict, and `coreos-installer install` has no auto-detect. Cut
two ISOs: `virtio` for KVM, Proxmox, OpenStack and libvirt, and `sd` for VMware, Hyper-V and bare
metal. Both stay generic across regions.

### The per-region stub

Keep the embedded config a stub that fetches the substantive one. The stub carries the only per-region
byte:

```yaml
variant: fcos
version: 1.5.0
ignition:
  config:
    replace:
      source: https://provision.fil.one/v1/appliance.ign
      httpHeaders:
        - name: Authorization
          value: Bearer ONE_TIME_ENROLLMENT_TOKEN
```

Producing a per-region artifact is then a second of CI work (`coreos-installer iso customize
--dest-ignition stub-us-east-1.ign` over the generic ISO), and the substantive config lives in one
place that changes without recutting images.

The operator can read that token off the ISO, and the threat model already concedes they can read
anything on the box. So: one-time use, short TTL, scoped to enrollment only, and no registry
credentials or wallet material anywhere in the fetched config. That is the same constraint Trust
boundary and secrets already imposes.

### Operator runbook

1. Create a VM with the specified vCPU and RAM, including the cutover headroom, and one boot disk at
   or above the stated minimum.
2. Attach the data volume as a second disk, raw and unformatted.
3. Boot the artifact FilOne sent, with the supplied `.ign` attached by whatever mechanism the platform
   uses.
4. Wait for the install and reboot. Detach the installer medium.
5. Report the public IP. FilOne creates `{region}.s3.fil.one`.

Nothing in that list requires knowing what Ignition is, which is what requirement 1 asks for.

### Disk identity

Two problems with different answers.

**The install target** is the dangerous one, because writing FCOS to the data volume destroys it. It is
decided at build time by `--dest-device`, which is why there are two ISO variants. The guard against
the destructive case, reinstalling over a live region, is a refusal:

```bash
#!/bin/bash
# --pre-install
set -euo pipefail
if blkid -L FILONE-DATA | grep -q .; then
  echo "a FILONE-DATA volume is attached; detach it before reinstalling" >&2
  exit 1
fi
```

The disk-image artifacts do not have this problem at all.

**The data volume** is identified by exclusion in the live installer, where both disks are visible and
there is a shell. After `coreos-installer install` runs, the OS disk is the one carrying the `root`
filesystem label, so the exclusion is exact:

```bash
#!/bin/bash
# --post-install: runs in the live environment after the OS is written, before reboot.
set -euo pipefail

if blkid -L FILONE-DATA | grep -q .; then
  echo "existing FILONE-DATA volume, adopting"; exit 0
fi

os_disk=$(lsblk -no PKNAME "$(blkid -L root)")
mapfile -t candidates < <(
  lsblk -dn -o NAME,TYPE,RM | awk '$2=="disk" && $3=="0" {print $1}' | grep -v "^${os_disk}$"
)

if [ "${#candidates[@]}" -ne 1 ]; then
  echo "expected one data disk besides ${os_disk}, found ${#candidates[@]}: ${candidates[*]-none}" >&2
  exit 1
fi

dev=/dev/${candidates[0]}
if blkid -p "$dev" >/dev/null 2>&1; then
  echo "$dev carries a filesystem or partition table; refusing to format" >&2
  exit 1
fi

mkfs.xfs -L FILONE-DATA "$dev"
```

Discovery belongs in the installer because Ignition's config is static and cannot enumerate devices,
while the live environment can. The installed system then only ever mounts
`/dev/disk/by-label/FILONE-DATA`, with identical configuration in every region.

The disk-image path has no installer phase, so the same logic runs as a first-boot oneshot ordered
`Before=var-mnt-filone.mount`.

## Additions to the operator hardware spec

Beyond disks, a public IP, inbound 443, and the 2× application headroom already listed:

- One boot disk at or above a stated minimum size.
- Exactly one additional volume, raw and unformatted. Two extra disks fails provisioning by design
  rather than picking one.
- A preformatted volume also fails, loudly, during provisioning.
- Which disk bus the platform presents, which selects the ISO variant, or which disk-image format the
  platform can import.

## Still open

- Whether `blkid -L root` is the right discriminator on the FCOS release we pin. Verify during the
  bake-off; the partition labels are stable but worth confirming rather than trusting.
- Hosters that cannot attach a raw unformatted volume, or that expose it only through their own agent.
  Survey the target hosters before writing "raw and unformatted" into the spec.
- Whether the enrollment endpoint is worth building for the first five regions, or whether a fully
  baked per-region config is simpler until config churn starts to hurt.
- Volume loss is a regional data-loss event, and the volume is the operator's storage with unknown
  redundancy. Backup targets for Postgres and Piri's SQLite belong in the same conversation as
  wal-g/pgBackRest to partner S3.

---

# 2026-08-06 What "mutable but 100% code-driven (Terraform-preferred)" unlocks

_Review of the FilOne Appliance Deployment Strategy RFC, addendum to the immutable-infrastructure analysis. Scope: what NEW options become viable once immutability is dropped in favour of "mutable infrastructure, all changes code-driven, Terraform preferred", with per-layer pros/cons. The immutability decision itself is not re-litigated._

## TL;DR

- The relaxation unlocks a real menu at the OS layer (Ubuntu Pro + Livepatch, openSUSE/Btrfs+snapper, ZFS boot environments, Packer golden images, cloud-init on stock templates) and at the app layer (Kamal, Nomad, Compose+Watchtower, K3s on a normal distro). But two of the headline candidates — **Kamal and any inbound-SSH push tool — are disqualified by the trust boundary**, and Kamal's zero-downtime advantage **evaporates for singleton Piri/Ingot** because there is no second instance to cut over to.
- The single highest-regret decision is **giving up unattended A/B rollback**. FCOS/bootc gave you "bad update → automatic boot into the last-good tree" for free on hardware nobody can touch. On a mutable distro you must rebuild that with **openSUSE MicroOS/Leap + snapper boot-into-snapshot** or **ZFS boot environments + zfsbootmenu**, both more moving parts and neither as clean.
- Recommended shape: **Terraform owns only central resources** (DNS, Vault/OpenBao policies, Grafana Cloud, registry, inventory) — it has almost nothing legitimate to do _inside_ a partner VM. Pick **one** OS with native rollback (openSUSE Leap Micro/MicroOS or a ZFS-BE Ubuntu), keep the **app layer on Podman Quadlet** (it loses nothing off FCOS), add **Ubuntu Pro Livepatch or TuxCare KernelCare** only if you move off FCOS, and drive in-VM convergence with **ansible-pull (outbound-only)**. NixOS is the one option that satisfies "all changes driven by code" more completely than Terraform+Ansible ever will while keeping rollback — but only if the team pays the Nix tax.
- **A follow-up discussion on the same date shifted the balance back toward bootc; section F records it.** Quadlet runs natively on an immutable OS and needs nothing from a mutable host, and unit-file _placement_ (`/etc` vs baked into `/usr` vs bootc bound images) is itself a per-layer cadence lever — which answers the "different approach per layer" question without adding a single extra tool. Three outputs: **do not bake Caddy or Postgres into the OS image** (§F3), **OS rollback does not roll back container images and nothing currently detects the mismatch** (§F4), and **a list of RFC-level defects that hold regardless of OS choice** (§F5) — the most serious being that `pdp-gate` lives in partner-writable, unsigned, never-rolled-back state. On the strength of §F, "keep bootc and relax only the no-layering rule" moves from _arguably the sweet spot_ to **the recommended default**.

---

## Key Findings

1. **The trust boundary kills push-based tooling, and this is the dominant constraint.** The partner VM is semi-trusted and the management path must be outbound-only. That single requirement eliminates Kamal (mandatory inbound SSH from a controller), plain Ansible push, and any Terraform in-VM provisioner over SSH. It strongly favours pull/dial-out models: ansible-pull, K3s agent (reverse tunnel), Nomad client→server, or a GitOps agent. **This is unchanged by the immutability relaxation and it is the fact that should drive the decision.**

2. **Live kernel patching is now genuinely available but buys FilOne very little.** Because Piri and Ingot are static pure-Go binaries (CGO_ENABLED=0), host glibc/OpenSSL CVEs cannot reach them, and the only public listener is containerised Caddy. Live kernel patching (Canonical Livepatch, kpatch, Ksplice, SUSE, TuxCare KernelCare) addresses kernel CVEs only — it does _not_ patch the container runtime (podman/crun) or systemd, which are exactly the components that still force a reboot. So live patching lets you _defer_ the batched-tier reboot, not eliminate it. The two-tier CVE policy survives essentially intact.

3. **Kamal, the option the prior matrix marked down only for "mutable host", is still wrong here for two independent reasons.** (a) It needs inbound SSH into every partner box — trust-boundary violation. (b) Its zero-downtime cutover requires running a second container alongside the old one; Piri/Ingot cannot run two instances against the same state, so Kamal degrades to `kamal app stop` → `kamal deploy` (downtime) — exactly the in-place restart we already have with Quadlet. **The Kamal advantage evaporates precisely on FilOne's workload.** This overturns the implicit "Kamal becomes attractive once the host is mutable" reading of the prior matrix.

4. **Nomad is the most coherent "Terraform-preferred" app-layer option** — client-only, dialing out to central FilOne Nomad servers (outbound-only compatible), submitted via the Terraform nomad provider — but it does not solve the singleton constraint either, and its `nomad alloc exec`/`logs` operations route through the servers over the client's outbound RPC connection, so break-glass debugging still works without inbound access. It adds a genuinely new capability (declarative fleet view, canary health-gating) at real maintenance cost.

5. **The clearest loss is unattended A/B rollback**, and the credible replacements are openSUSE-style **btrfs + snapper boot-into-snapshot** (true subvolume-swap rollback, integrated bootloader entries) and **ZFS boot environments + zfsbootmenu** (boot-time snapshot browse/rollback, even over SSH). Both work, both are more complex than FCOS's built-in A/B, and both need the partner to pick a snapshot at the GRUB/ZBM menu if an update leaves the box unbootable and auto-rollback (boot-counting) is not perfectly wired.

6. **The reboot is not the cheap event, and this inverts a prior conclusion.** Because a reboot stops `piri.service` on the way down and honours `TimeoutStopSec=330`, an OS promotion costs gate-wait + SIGTERM + drain (up to 300s) + boot (30–90s) + start + migrations + `HealthStartPeriod` — call it 2–7 minutes, load-dependent, consuming a PDP-safe window. A Caddy container restart is ~1–3 seconds. **If the drain is honoured on shutdown, a reboot is a strict superset of a Piri restart**, so the Piri review's "the application upgrade is the expensive event and the host reboot is the cheap one" is backwards and should be restated. This is what disqualifies baking Caddy into the OS image (§F3).

7. **OS rollback and container images are independent lifecycles, and the gate script sits on the wrong side of the line.** bootc's own documentation states that for floating images — anything fetched by podman-systemd — host upgrades and rollbacks do not affect the set of images. Meanwhile `/var` is shared across ostree deployments while `/etc` is per-deployment, so _old units + new images_ is a reachable, undetected state. The more serious discovery: on ostree `/usr/local` is a symlink into `/var`, so `ExecStartPre=/usr/local/bin/pdp-gate` actually resolves to `/var/usrlocal/bin/` — **shared state that never rolls back, is writable by the partner, and is covered by no signature**, because cosign policy protects images and not shell scripts. The control that prevents slashing can be neutered with a one-line edit and nothing notices. Cheapest high-value fix in this entire review (§F4, §F5).

8. **Podman's BoltDB→SQLite transition is a live hazard for a Quadlet-at-boot topology, on either OS model.** SQLite became the default during 4.x/5.0, automatic migration landed in 5.8, and 6.0 removes BoltDB entirely — Fedora's change page warns that unmigrated hosts find container state unusable after the upgrade. Podman issue #28216 (March 2026) reports the 5.8 migration is not concurrency-safe and reproduces it with several Quadlet units starting together at boot: one unit migrated while others kept writing to BoltDB, leaving a container invisible to `podman ps` despite systemd reporting the service active. Pin the stream across that boundary and serialise unit startup.

## A. What the relaxation unlocks, layer by layer

### A1. Operating system layer — general-purpose distros become viable

Dropping "atomic/immutable OS" means you can now run a stock hoster template plus cloud-init instead of shipping a pre-baked qcow2/OVA/ISO with embedded Ignition. That is a **real** onboarding-friction reduction for requirement 1: almost every hoster and hypervisor ships ready-to-boot cloud images for Ubuntu/Debian/Rocky/openSUSE, and the partner does not have to know how to boot an ISO or import an OVA — they clone a template the hoster already has. Given Forrest's two concerns (2014-era Xeons lacking SHA extensions; "must know how to boot an ISO"), cloud-init on a stock template is the friction-minimising path.

**cloud-init vs Ignition datasource/hypervisor coverage.** This is where cloud-init wins decisively for a heterogeneous partner fleet:

- **cloud-init** ships datasources for essentially every platform in the RFC's list: OpenStack, VMware (vSphere GOSC via guestinfo, 7.0U3+), NoCloud (bare metal / any hypervisor via seed ISO), Proxmox (NoCloud/ConfigDrive — natively integrated in the Proxmox UI), libvirt/KVM (NoCloud/ConfigDrive), plus AWS/Azure/GCE/Oracle/Vultr and MAAS. Proxmox has _native_ cloud-init support in its VM template workflow.
- **Ignition** supports libvirt, bare metal, Proxmox (proxmoxve, via config drive), QEMU (fw_cfg), VMware (guestinfo), OpenStack, Nutanix, VirtualBox, Oracle Cloud, and others — but the _ergonomics_ are worse on the hypervisors FilOne partners actually use. On Proxmox, feeding Ignition requires setting the `args` field (blocked for API-token auth; only root@pam password auth works), and the community workflow is "run an HTTP server in an LXC to serve the Ignition file." On VMware you must set `guestinfo.ignition.config.data`. Ignition also only runs on _Ignition-enabled OS images_ (FCOS/Flatcar/RHCOS) — you cannot Ignition a stock Ubuntu template.

**Per-distro provisioning story:**

- **Ubuntu LTS + Ubuntu Pro.** Best-supported cloud images everywhere; cloud-init is Canonical's own tool. Ubuntu Pro is billed per workstation ($25/year) or per server ($500/year), and is free for up to 5 machines (50 for official Ubuntu Community members) — verified against Canonical's official pricing page. It unlocks Livepatch (see A2) and — critically for requirement 7 — the **snapshot.ubuntu.com** archive service (see A4). This is the most operationally mature general-purpose option. Verdict: **strong default if you leave FCOS.**
- **Debian stable.** Rock-solid, minimal, excellent cloud images, cloud-init native. No first-party livepatch (TuxCare covers it). No first-party archive-snapshot service (use snapshot.debian.org — see A4). Verdict: **viable, slightly more DIY than Ubuntu.**
- **RHEL / Rocky / AlmaLinux (package mode).** Good cloud images, dnf versionlock for pinning, kpatch available on RHEL (subscription) or TuxCare on Rocky/Alma. Rocky/Alma are free. Verdict: **viable; pick Alma/Rocky + TuxCare to avoid RHEL subscription cost.**
- **openSUSE Leap / Leap Micro / MicroOS.** The differentiator: **btrfs + snapper with boot-into-snapshot is default and deeply integrated** — every zypper transaction gets pre/post snapshots and the GRUB menu can boot a read-only snapshot, and `snapper rollback` does a true subvolume swap. Leap Micro / MicroOS give you a transactional-update model on a general-purpose base. Verdict: **the strongest OS-layer answer to "keep rollback without FCOS."**

**Honest caveat:** the onboarding-friction win assumes the hoster offers a stock template. Where FilOne controls the hypervisor (Proxmox/vSphere it owns), a Packer golden image (B3) is at least as easy. Where the partner brings bare metal, you are back to ISO/PXE regardless of OS — cloud-init via a NoCloud seed ISO doesn't remove that.

### A2. Live kernel patching — now available, but a small prize here

| Product                           | Distro scope                                      | Userspace patching                           | Outbound-only?                       | ~Price/machine/yr                            |
| --------------------------------- | ------------------------------------------------- | -------------------------------------------- | ------------------------------------ | -------------------------------------------- |
| **Canonical Livepatch**           | Ubuntu LTS only                                   | No (kernel only)                             | Yes (snap dials out)                 | Bundled in Ubuntu Pro ($500 server; free ≤5) |
| **Red Hat kpatch**                | RHEL (+ EUS for windows)                          | No                                           | Yes                                  | Rides RHEL sub (~$879 Standard)              |
| **Oracle Ksplice**                | Oracle Linux (forces Oracle kernel)               | Yes, but after a one-time restart            | Yes                                  | Rides OL Premier (~$1,399/CPU-pair)          |
| **SUSE Live Patching**            | SLES                                              | No                                           | Yes                                  | Rides SLES sub                               |
| **TuxCare KernelCare Enterprise** | RHEL/Rocky/Alma/Oracle/Ubuntu/Debian + 60 distros | **Yes, via LibCare (OpenSSL/glibc, x86-64)** | Yes (or on-prem ePortal for air-gap) | ~$50/server (LibCare add-on ~$34.50)         |

TuxCare is the cheapest by a wide margin and the only one that patches userspace libraries in memory and works across the mixed distro fleet a partner network implies. At 10/30/300 machines: Ubuntu Pro ≈ $5k/$15k/$150k; TuxCare KernelCare ≈ $500/$1.5k/$15k (list; volume discounts likely).

**The sanity check — how much does this actually buy FilOne?** Very little, and this is a real finding. Because Piri/Ingot are static Go binaries, LibCare's OpenSSL/glibc patching — TuxCare's main differentiator — is irrelevant to them; those libraries aren't linked into the app. The only public listener is containerised Caddy, which uses its own bundled Go TLS stack, not host OpenSSL. So the userspace-patching selling point is moot. Kernel live patching does help — it lets you defer the _batched-tier_ reboot for kernel CVEs — but it patches **only the kernel**, not the container runtime (podman/crun) or systemd, which are exactly the host components the prior research identified as still requiring a reboot. **Conclusion: live kernel patching changes the two-tier policy only at the margin. If you stay on Ubuntu it comes free with Pro (which you may want anyway for snapshot.ubuntu.com); buying TuxCare purely for live patching is not justified by this workload.**

### A3. Rollback and failure recovery without an A/B atomic OS — the clearest loss

This is where the relaxation costs the most. On FCOS/bootc, a bad update meant the box booted the previous deployment automatically; nobody had to touch it. Substitutes, assessed for "unattended recovery at 3am on a box only the partner can physically reach":

- **openSUSE btrfs + snapper (boot-into-snapshot).** The best substitute. Pre/post snapshots around every zypper transaction; GRUB can boot a read-only snapshot; `snapper rollback` swaps the default subvolume so kernel + packages + RPM DB all roll back together. **Limit:** if a kernel update leaves the box _unbootable_, recovery requires selecting the snapshot from the GRUB menu — that is a human at a console (or serial/IPMI), i.e. the partner. Auto-rollback on failed boot needs boot-counting wired up; it is not as bulletproof as FCOS's automatic greenboot-style rollback.
- **ZFS boot environments + zfsbootmenu.** Strong: timer-and-update snapshots, boot-time snapshot browser, and **remote rollback over SSH at the ZBM prompt** (zfsbootmenu explicitly supports unlocking/rolling back over SSH). Requires the OS on a single ZFS filesystem for clean rollback, and ZFS-on-root setup is non-trivial to bake reproducibly. Still needs someone (or an SSH break-glass) to pick the BE if auto-boot-assessment isn't configured.
- **LVM snapshots / Timeshift.** File-level restore, not a clean atomic OS swap; kernel/bootloader rollback is manual. **Not adequate** for unattended kernel-update recovery.
- **systemd-sysupdate + systemd-sysext/confext.** Genuinely interesting middle path: sysupdate is a systemd-native A/B image updater (files/dirs/partitions, A/B/C slots, signed SHA256SUMS manifest), and sysext mounts a squashfs over /usr to add software without rebuilding the base. This effectively _reintroduces_ image-based A/B onto a systemd distro — but it requires you to build and host the image artifacts and lay out A/B partitions yourself. If you're going to do that, you are 80% of the way back to the FCOS/bootc model you just relaxed. **Coherent only if you want A/B rollback but reject rpm-ostree specifically.**
- **Flatcar + sysext.** Flatcar is an Ignition/immutable OS like FCOS — choosing it contradicts the relaxation. Mentioned for completeness; not a "mutable distro" answer.

**Concrete 3am scenario.** A kernel package update panics on boot. On FCOS: greenboot marks the deployment failed and the box rolls back automatically — zero human action. On stock Ubuntu with no snapshot layer: the box is down until someone gets console access — this is the catastrophic case, and it's the same failure class the prior research flagged for kexec on partner hardware. On openSUSE/snapper or ZFS-BE with boot-counting wired: the box boots the previous snapshot/BE automatically. **Verdict: if you leave FCOS, openSUSE Leap Micro (or ZFS-BE) with automatic boot assessment is not optional — it is the price of keeping requirement-1-compatible unattended recovery.**

### A4. Reproducibility and version pinning under a mutable OS (requirement 7)

"Only the combination tested in staging gets deployed" is harder when apt/dnf repos move under you. Options:

- **Ubuntu snapshot service (snapshot.ubuntu.com).** The standout. Point-in-time view of the entire archive for any date after 1 March 2023; the command form is verbatim `apt install hello --update --snapshot 20240301T030400Z`, auto-detected on 24.04+ and opt-in on earlier releases (supported "in Ubuntu 23.10 onward and on updated installations of Ubuntu 20.04 LTS (starting from apt 2.0.10) and Ubuntu 22.04 LTS (starting from apt 2.4.11)" per Canonical's Server docs). This is purpose-built for "staging and prod pull byte-identical packages." **This alone is a strong reason to pick Ubuntu.** Canonical's stated retention: "We intend to ensure snapshots are available for dates up to at least 2 years in the past, which we may extend if there is demand."
- **snapshot.debian.org.** Equivalent for Debian, long-established; pin sources.list to a timestamped snapshot. Slightly less integrated than Ubuntu's `--snapshot` flag but fully usable.
- **apt pinning / `apt-mark hold`; dnf `versionlock` (python3-dnf-plugin-versionlock).** Pin individual packages. Fine for a handful of critical packages, brittle as a fleet-wide reproducibility strategy because transitive deps still float. Use as a supplement, not the primary mechanism.
- **Internal mirror / Pulp / Artifactory / reprepro / aptly.** The heavyweight answer: mirror the exact repo state you tested, point all regions at it. Gives total control and offline capability but is real infrastructure to run and secure. **At 30 regions this is arguably overkill if snapshot.ubuntu.com exists; at 300 regions a mirror (or a caching proxy in front of the snapshot service) becomes attractive for bandwidth and blast-radius control.**

**Operational cost read:** at 30 regions, `snapshot.ubuntu.com` + a pinned snapshot ID per release ≈ near-zero marginal cost and directly satisfies R7. At 300 regions, add a pull-through cache/mirror. Choosing Ubuntu makes R7 almost free; choosing a distro without an archive-snapshot service pushes you toward running Pulp/aptly, which is the more expensive path.

## B. Configuration management and the role of Terraform

### B1. Terraform's actual fit and limits

**Central question, answered bluntly: if the partner provisions the VM, Terraform has almost nothing legitimate to do _inside_ the box.** Terraform's model is "reconcile declared cloud/API resources against state." A partner-owned VM is not a resource FilOne's Terraform created and cannot authoritatively manage. What Terraform _should_ own here is **central resources only:**

- DNS records (Cloudflare/Route53 providers) for each region's stable domain.
- Vault/OpenBao policies, roles, auth backends (per-region identities).
- Grafana Cloud tenants, alert rules, dashboards, per-region tokens.
- Registry credentials / robot accounts.
- Per-region inventory as data (feeding the pull layer's Git repo).

This is Forrest's point restated and it is correct: **"the purpose of TF is to deploy infra, not apps."**

**Where FilOne _does_ control the hypervisor**, the hypervisor providers become legitimate and useful: `bpg/proxmox` (mature, actively maintained), `dmacvicar/libvirt`, `vsphere`, `openstack`, `xenorchestra` (XCP-ng), and MAAS/Tinkerbell for bare metal. There Terraform genuinely provisions the VM (clone template, attach cloud-init/Ignition, set resources). This applies to a minority of regions.

**In-VM options, assessed honestly:**

- **`kreuzwerker/docker` provider over SSH/TLS socket.** Works, but requires FilOne to hold an inbound path to the Docker socket — **trust-boundary violation**, and it makes Terraform state responsible for container lifecycle, which drifts the moment podman auto-update or the partner touches anything. **Reject.**
- **remote-exec / file provisioners.** HashiCorp explicitly discourages provisioners as a last resort; they are not idempotent, not tracked in state, and need inbound SSH. **Reject.**
- **`terraform-provider-ansible` and the new (HashiConf 2025) Terraform Actions + Ansible/AAP integration.** Terraform Actions (beta as of Sep 2025) can _trigger_ an Ansible/AAP playbook or Event-Driven Ansible from a `terraform apply` as a codified Day-2 operation. This is the sanctioned "Terraform for infra, Ansible for config" seam — but it still relies on Ansible's own connectivity model (which, for outbound-only, means ansible-pull, not AAP push). Useful as the _orchestration trigger_, not as a way to reach into the box.
- **Nomad / Kubernetes / Helm providers.** These are legitimate: point the Terraform `nomad` provider at central Nomad servers and submit jobs declaratively. This is the clean way to do "Terraform-preferred" app deployment _without_ Terraform touching the box directly — Terraform talks to the orchestrator's API, the orchestrator's agent (dialed out from the box) does the work. **This is the most defensible in-VM-adjacent use of Terraform.**

**State management at 30–300 VMs.** One monolithic state is a non-starter (blast radius, lock contention). Options: per-region **workspaces** (simple, but 300 workspaces is a lot of `plan` runs), **Terragrunt** (DRY wrapper, keeps per-region state files, battle-tested at this scale), or **Terraform Stacks** — which HashiCorp's official blog confirms went GA at HashiConf 2025 (Sep 25, 2025) "for all new HCP Terraform plans based on resources under management (RUM)," with the standalone `terraform-stacks-cli` now deprecated in favour of a `terraform stacks` subcommand in the main CLI. Stacks is purpose-built for "same components, many deployments (regions)" and is the natural fit _if_ you're on HCP Terraform and accept the RUM billing and cloud dependency. On-prem Terraform Enterprise support was still trailing GA at announcement (HashiCorp's Liese, per TechTarget: "when you actually get to the deployment stage, that's an HCP Terraform thing"). **Recommendation: Terragrunt for 5–30 regions today; evaluate Stacks at the ~30-region inflection if already on HCP Terraform.**

**Drift when nobody runs `plan`.** This is the honest weakness of Terraform-for-central-resources combined with a mutable box: Terraform only knows about drift in the resources _it_ manages (DNS, Vault, Grafana). It has **zero visibility** into the partner VM's on-disk state. Drift _inside_ the box is detected only by the pull-convergence layer (ansible-pull re-asserting state every 30 min) or the orchestrator (Nomad/K3s reconciling declared jobs). **So "drift detection" splits by layer: Terraform for central, the pull agent for in-VM. Do not expect Terraform to catch a partner editing a Caddyfile.**

### B2. Config management, push vs pull, under the trust boundary

The outbound-only constraint is the filter. Verified network directions:

- **Ansible push (ad-hoc over SSH).** Needs inbound SSH into 300 partner boxes. **Reject on trust boundary.**
- **ansible-pull from Git.** Each box clones a Git repo (outbound HTTPS/SSH to the Git host) and runs the playbook locally via cron/systemd-timer, with `--only-if-changed` and a random sleep. **Outbound-only, scales to thousands, and is the natural convergence loop for a mutable box.** Loss vs push: no central real-time orchestration, no single-terminal view of results, and you must ship logs out (Alloy already does this). **This is the recommended in-VM convergence mechanism.**
- **Puppet / Chef / Salt.** Puppet and Salt both have agent/minion pull models (Salt can also do `salt-ssh` push or masterless). Puppet agent dials out to a master — outbound-compatible — but is heavier than ansible-pull and adds a server to run. Salt masterless (salt-call from a Git checkout) is effectively ansible-pull with a different syntax. **No advantage over ansible-pull for this shape; more infrastructure.**
- **Chezmoi-style / plain Git + systemd.** Lightweight dotfile-style convergence. Fine for tiny config surface; underpowered for full host convergence.
- **GitOps agent (Flux/Argo on K3s).** Agent dials out to Git and reconciles — outbound-only compatible — but requires running K3s (see C3). Powerful declarative reconciliation and drift-healing; heavy for a single-node appliance.

**Orchestrator dial-out models (verified):**

- **Nomad client → server.** Client needs outbound to server ports 4646 (HTTP API) / 4647 (RPC); **the Serf WAN gossip port 4648 is server-to-server only and clients do not need it** (HashiCorp docs: "It isn't required that Nomad clients can reach this address"). Crucially, `nomad alloc logs` and `nomad alloc exec` are served through the client's RPC connection to the servers — the operator hits the _server_ API and the traffic is relayed — so **log retrieval and exec do not require inbound access to the client.** This is a strong trust-boundary fit.
- **K3s agent → server.** Verified verbatim from official K3s docs: "K3s uses reverse tunneling such that the nodes make outbound connections to the server and all kubelet traffic runs through that tunnel." The agent maintains a websocket tunnel to the server (port 6443), and in the default `agent` egress mode this "allow[s] agents to operate without exposing the kubelet and container runtime streaming ports to incoming connections" — so **`kubectl exec` and `kubectl logs` flow back through that tunnel with no inbound port on the agent.** Also strong trust-boundary fit. (The VXLAN port 8472 must be firewalled off from the world but is only relevant in multi-node clusters; a single-node appliance has no peer agents.)

**Verdict for B2:** the outbound-only requirement is satisfiable three ways — **ansible-pull (simplest, no orchestrator)**, **Nomad client (declarative, Terraform-submittable, exec/logs via server)**, or **K3s agent (GitOps-native, exec/logs via reverse tunnel)**. All three respect the trust boundary. Kamal and Ansible-push do not.

### B3. Packer + Terraform golden images as a middle path

Forrest is right that Packer is the HashiCorp tool for golden images, and its plugin coverage is complete for FilOne's targets: **`proxmox-iso`/`proxmox-clone`, `vsphere-iso`/`vsphere-clone`, QEMU (qcow2/raw output), VirtualBox (native OVF/OVA output)** are all official, current plugins (2025-2026, e.g. vSphere plugin v2.1.2); QEMU cannot emit OVA directly (per HashiCorp maintainer: "it can't do that directly. It only supports raw and qcow2 disks") but a shell-local post-processor bridges that. HashiConf 2025 added Packer **SBOM storage (GA)** and package-visibility (beta) via the `hcp-sbom` provisioner (Packer ≥1.12) — a supply-chain nicety.

**What a Packer golden image of a mutable distro gets you over the two alternatives:**

- _vs FCOS/bootc image:_ you get a general-purpose distro (Ubuntu/Rocky/openSUSE) with all its tooling and the live-patch/snapshot options above, baked to a known state — without rpm-ostree.
- _vs stock template + cloud-init:_ you get a _pinned, tested_ starting point (versions frozen at build time, cosign-able) instead of "whatever the hoster's template has today," which directly helps R7.

**The honest risk — "golden image + in-place mutation" can be the worst of both worlds.** If you bake an image _and then_ let ansible-pull mutate it continuously, you have **both** image sprawl (a new artifact per release, per output format, per hypervisor) **and** drift (the box diverges from the image the moment convergence runs). The coherent position is one of two disciplines: (a) **image is the unit of change** — rebuild and redeploy the image per release, minimal in-place mutation (this is basically re-inventing immutable infra on a mutable distro, and if you want that, systemd-sysupdate or just going back to bootc is cleaner); or (b) **image is a fast-start baseline** — bake rarely (quarterly), converge continuously with ansible-pull, and accept that the image is just to cut cloud-init time and pin the floor. **Pick (b) explicitly, or don't use Packer.** Build time is minutes-to-tens-of-minutes per image and CI cost is a per-release image build per output format — non-trivial at several hypervisor targets but not prohibitive.

## C. Newly / more viable application-layer options

### C1. Kamal 2.x — reject, for two independent reasons

The prior matrix scored Kamal 5 on R4 (zero-downtime) and marked it down only for requiring a mutable host. With the host now mutable, one might expect Kamal to rise. **It does not, and this is worth stating plainly:**

1. **Trust boundary.** Verified against official Kamal docs: Kamal runs from a controller/CI/laptop and "uses SSH to connect run commands on your hosts. By default it will attempt to connect to the root user on port 22." The target **must accept inbound SSH from the controller**. There is **no pull/agent mode** — inbound SSH is mandatory. Driving 300 partner boxes this way means holding inbound SSH into all of them. **Direct violation of the outbound-only management-path requirement.**
2. **The singleton constraint dissolves the zero-downtime advantage.** Verified against 37signals docs and maintainer guidance: kamal-proxy achieves zero downtime by starting "a new container... Tell kamal-proxy to route traffic to the new container once it is responding with 200 OK to GET /up... Stop the old container running the previous version" — i.e. new-alongside-old. Piri/Ingot/Postgres/Vault/Caddy **cannot run two instances against the same state**. For a single-instance service the documented workaround is `kamal app stop` followed by `kamal deploy` (or a pre-deploy hook doing the stop) — i.e. **stop-then-start with downtime**, which is exactly the in-place restart Quadlet already gives you. Kamal's `boot` config only controls rollout _across hosts_, not single-node replacement.

Kamal has nice ergonomics elsewhere (`.kamal/hooks/pre-deploy` aborts on non-zero exit — a clean home for the PDP gate; `.kamal/secrets` with 1Password/Bitwarden/LastPass/AWS/GCP/Doppler adapters — **but not a first-class Vault adapter**; git-SHA-tagged rollback). None of that overcomes the two disqualifiers. **Verdict: Kamal is the option the relaxation appears to unlock but does not. Do not adopt it for the partner fleet.** (It could be fine for FilOne's own centrally-controlled infra, which is a different trust domain.)

### C2. HashiCorp Nomad — the genuinely newly-attractive option

Given "Terraform preferred" and HashiCorp-stack coherence, Nomad is the option that most rewards the relaxation.

- **Topology:** run **client-only** on each partner box, dialing out to central FilOne Nomad servers (3–5 servers, federated across regions). Outbound-only: client needs 4646/4647 outbound to servers; 4648 Serf is server-only. **Exec/logs relay through the servers — no inbound to the client.** This is the trust-boundary sweet spot.
- **Footprint:** a single Go binary, modest RAM — comfortable on a partner VM, and lighter than a K3s control plane.
- **Update/canary + health + external gate:** the `update` stanza with `canary`, `health_check`, `min_healthy_time` and `auto_revert` gives health-gated cutover _for services that can canary_. For Piri/Ingot it **still cannot conjure a second instance** — you express `count = 1` with `max_parallel = 1` and an in-place restart. The PDP gate is expressed as a pre-restart check (a `prestart` task/hook or an external check that Nomad honours before draining). So Nomad **does not solve the singleton constraint** either — nothing does — but it makes the in-place restart declarative and fleet-visible.
- **Secrets:** native Vault integration + `consul-template`/`template` stanza renders short-lived creds into the alloc (tmpfs), which fits the "central Vault/OpenBao + agent rendering into tmpfs" recommendation cleanly — better than Quadlet's DIY secret handling.
- **Terraform:** the `nomad` provider submits jobs declaratively from central Terraform — this is the "Terraform-preferred" story realised without Terraform touching the box.
- **Honest maintenance burden:** running 3-5 Nomad servers (+ ideally Consul for service discovery, +Vault) is a real platform to operate and upgrade. Versus Quadlet (zero central infra) it is a large step up; versus K3s it is lighter and less churny (no kubelet/etcd/CNI/control-plane version treadmill). At 300 single-node clients Nomad is operationally proven for edge fleets.

**Verdict:** Nomad is the right answer _if_ FilOne wants a declarative, Terraform-submittable, outbound-only fleet control plane and is willing to run the servers. At 3-10 regions it is over-built; it earns its keep around the ~30-region inflection where a declarative fleet view and canary/health-gating matter.

### C3. Other app-layer options a mutable host opens up

- **Podman Quadlet on a mutable distro.** **Quadlet loses nothing off FCOS.** Read this as a statement about _portability_ — Quadlet is a Podman/systemd feature (Podman ≥4.4) available on Ubuntu/Debian/Rocky/openSUSE, so moving it off an immutable base costs nothing. It is emphatically **not** a claim that Quadlet wants or needs a mutable host: Quadlet is native to FCOS/bootc and that pairing is the intended one (see §F1). This is the smallest-delta option: keep the exact app-layer design from the prior recommendation, whichever base you choose. Note `AutoUpdate=registry` and a digest-pinned `Image=` cancel out — the RFC's Piri review already resolved this in favour of a promotion-moved channel tag. Verdict: **default app layer.**
- **Docker Compose + Watchtower / Renovate.** Storj does exactly this (single image + watchtower). Simple, familiar, but Watchtower auto-pulling floating tags contradicts requirement 7 (pin digests) and gives no health-gated cutover. Renovate proposing digest bumps into staging (already in the prior plan) is the disciplined version. Verdict: **viable but a downgrade from Quadlet; no reason to switch.**
- **docker-rollout.** Adds blue-green to Compose — but same singleton problem as Kamal; irrelevant for Piri/Ingot.
- **systemd units + plain binaries + socket activation.** The lightest possible; socket activation can even mask restart gaps for some services. But you lose container isolation/image distribution and take on config-management-for-drift (the prior matrix's R6=3 concern). Verdict: **not worth it given the app is already containerised.**
- **supervisord/s6 in a single "omnibus" container (GitLab-Omnibus style).** Attractive for onboarding simplicity (one container, one thing to run) but couples Piri/Ingot/Caddy/Postgres lifecycles and fights the "each service updated frequently and independently" requirement and the PDP-gated Piri restart. Verdict: **reject for FilOne's update cadence.**
- **Single-node K3s / k0s on a mutable distro.** Now viable without an immutable OS. Reverse-tunnel agent model is a genuine trust-boundary fit, and K3s+Flux scored 5/5 on R4/R6 in the prior matrix. But the prior "high maintenance from control-plane churn" objection **survives unchanged** — a single-node cluster still carries etcd/sqlite, kubelet, CNI, and the Kubernetes version treadmill for one pod's worth of work. Verdict: **only if FilOne independently wants Kubernetes; otherwise Nomad gives the same dial-out benefits with less churn.**
- **NixOS — deserves a fresh look, and here is the honest read.** NixOS is declarative and code-driven with atomic generations and bootloader rollback — it delivers reproducibility, atomic upgrade, and rollback _through functional package management_, not through a read-only root. Crucially, **NixOS satisfies "all changes driven by code" more completely than Terraform+Ansible ever will**: the entire machine (packages, services, users, kernel, bootloader) is one `configuration.nix`/flake with a `flake.lock` pinning exact revisions — same inputs, same system, every time, with `nixos-rebuild switch --rollback` and per-generation boot entries. That is _exactly_ the RFC's stated ideal, and it keeps rollback without FCOS. **What it uniquely solves:** it is the only option where R6 ("mutable but fully code-driven") and R7 (exact pinned combination) and unattended rollback are all satisfied _by construction_ rather than bolted on. **The cost is real:** the Nix language and model are a steep learning curve for a small Go team, FHS-expecting software needs workarounds, and the whole team must be able to operate it at 3am — you cannot have one Nix expert and a bus factor of one across 300 regions. The prior research rejected NixOS on team-adoption cost, not technical grounds, and **that judgement still stands** — but if the team is willing to invest, NixOS is the most complete answer to the literal requirement. Verdict: **the technically-best fit for the stated requirement; adopt only with genuine team-wide Nix buy-in, otherwise its rollback/reproducibility benefits are better bought via openSUSE/snapper at a fraction of the learning cost.**

## D. Mixing approaches per layer

The user explicitly asks whether layers can use different tools. General principle: **each additional tool is another mental model the 3am on-call engineer must hold, and every seam between layers is a place two tools can both think they own the same file/unit.** Judge each combo by (i) distinct mental models, (ii) what breaks at the seams, (iii) where drift-detection responsibility sits.

- **TF (central) + Packer image + ansible-pull (OS/infra) + Quadlet/Compose (app).** Mental models: 4 (TF, Packer, Ansible, Podman/systemd). Industry-standard combination; the seams are well-understood. Drift: TF (central), ansible-pull (in-VM OS/infra), Quadlet auto-update (app). **Seam risk:** ansible-pull and Quadlet must not both manage the same unit files — give ansible-pull ownership of _writing_ Quadlet files and let systemd/podman own runtime. Verdict: **coherent, recommended baseline.**
- **TF + cloud-init (stock distro) + ansible-pull + Kamal.** Same as above but Kamal for app — **reject** (C1: inbound SSH + singleton). Drop Kamal, use Quadlet.
- **TF + Nomad for everything above the OS.** Mental models: 2-3 (TF, Nomad, +Vault/Consul). Cleanest HashiCorp-stack story; Nomad owns app _and_ can run infra services as jobs. Seam risk low (one control plane). Drift: TF central, Nomad reconciles jobs. Verdict: **coherent; the "all-in on HashiCorp" path.** Best at ≥30 regions.
- **bootc/FCOS retained for OS, "no package layering" rule relaxed, + TF central.** The _minimal_ relaxation: keep A/B rollback (the thing you'd otherwise rebuild), allow a little package layering for e.g. a kernel module or a debug tool, Terraform for central only. Mental models: 2 (rpm-ostree/bootc, TF). **This is the lowest-regret way to honour the user's request without throwing away unattended rollback** — it relaxes immutability _just enough_ to add packages while keeping the safety net. Verdict: **strongly worth considering; arguably the sweet spot.**
- **NixOS (OS+infra) + containers (app).** Mental models: 2 (Nix, containers) but Nix is a heavy one. Everything-as-code, rollback native. Seam: Nix can _also_ declare the containers (oci-containers module) — tempting to unify, but then app cadence is coupled to nixos-rebuild. Better to let Nix own OS/infra and Quadlet/Podman own the fast-moving app. Verdict: **coherent and elegant iff the team pays the Nix tax.**
- **K3s + Flux on a mutable distro.** Mental models: 2-3 (K8s, Flux, distro) but K8s is heavy. Everything reconciled via GitOps; drift self-heals. Seam risk: the distro's own updates (kernel/podman) sit _below_ K8s and aren't managed by Flux — you still need a host-update story (unattended-upgrades or a snapshot layer). Verdict: **only if you want K8s for its own sake.**

**The cross-cutting failure mode to design against:** two tools both believing they own the same file or unit — e.g. cloud-init writing a Caddyfile _and_ ansible-pull templating it, or Quadlet auto-update bumping a digest _and_ Ansible pinning a different one. **Mitigation: assign each file/unit exactly one owner, document it, and make the other layers read-only consumers.** This is the single discipline that keeps a multi-tool stack sane.

## E. Updated decision material

### E1. Comparison matrix (1-5; R6 = "mutable but fully code-driven, TF-preferred")

| Option (app/OS layer)                                           | R1 onboarding | R2 fast fix (hrs) | R3 observability | R4 short downtime | R5 PDP timing | R6 code-driven/TF | R7 pinning | Maint. burden | Drift resist. | Unattended rollback | Trust-boundary fit  |
| --------------------------------------------------------------- | ------------- | ----------------- | ---------------- | ----------------- | ------------- | ----------------- | ---------- | ------------- | ------------- | ------------------- | ------------------- |
| **Quadlet + systemd on openSUSE Leap Micro (snapper)**          | 4             | 5                 | 5                | 3                 | 5             | 4                 | 4          | 4             | 3             | **4**               | 5                   |
| **Quadlet on Ubuntu Pro + Livepatch + snapshot.ubuntu.com**     | 4             | 5                 | 5                | 3                 | 5             | 4                 | **5**      | 4             | 3             | 2 (no A/B)          | 5                   |
| **Quadlet on FCOS/bootc (prior rec, package-layering relaxed)** | 4             | 5                 | 5                | 3                 | 5             | 4                 | 5          | 4             | **5**         | **5**               | 5                   |
| **Nomad client + Vault (any mutable distro)**                   | 3             | 5                 | 5                | 3                 | 5             | **5**             | 4          | 2             | 4             | 3 (OS-dependent)    | 5                   |
| **Kamal on any mutable distro**                                 | 4             | 5                 | 4                | **2***            | 4             | 3                 | 4          | 4             | 2             | 2                   | **1 (inbound SSH)** |
| **K3s + Flux on mutable distro**                                | 2             | 5                 | 5                | 5                 | 4             | 5                 | 4          | 2             | 5             | 3                   | 5 (reverse tunnel)  |
| **NixOS + Quadlet**                                             | 3             | 5                 | 5                | 3                 | 5             | **5**             | **5**      | 3 (Nix tax)   | **5**         | **5**               | 5                   |
| **Docker Compose + Watchtower (Storj-style)**                   | 4             | 4                 | 4                | 3                 | 3             | 3                 | 3          | 4             | 2             | 2                   | 5                   |

*Kamal R4=2 because for singleton Piri/Ingot it degrades to stop-then-start; the 5 it would score for canary-able services does not apply here.

### E2. What the relaxation costs, stated plainly

The prior FCOS/bootc recommendation got four things **for free** that must now be rebuilt or accepted as risk:

1. **Unattended A/B rollback.** Gone unless you re-buy it with openSUSE/snapper boot-into-snapshot, ZFS boot environments, systemd-sysupdate, or NixOS generations. On a plain mutable distro, a bad kernel update is a partner-console incident.
2. **Drift-freeness by construction.** A read-only /usr made most drift impossible. On a mutable box, drift is prevented only by an active convergence loop (ansible-pull / orchestrator reconcile) that you must run and monitor.
3. **Digest-pinned OS as a single signed artifact.** The signed OCI release manifest pinning OS + every container digest as one promotable unit is harder when the OS is a package set. You partially recover R7 via snapshot.ubuntu.com (Ubuntu) or a Packer image digest, but "one artifact, one digest" becomes "an image plus a pinned snapshot ID plus versionlock."
4. **The clean "no layering" discipline.** Package mode invites ad-hoc host packages; you must impose discipline (versionlock/hold + a minimal package policy) that the immutable base enforced automatically.

### E3. Prior conclusions: overturned / survive / re-verify

**Overturned / newly-nuanced:**

- _"Kamal becomes attractive once the host is mutable."_ **Overturned.** It is disqualified by inbound-SSH and by the singleton constraint dissolving its zero-downtime advantage.
- _Live kernel patching was rejected on FCOS as unavailable._ **Now available** on general-purpose distros — but the fresh finding is that it **buys FilOne almost nothing** because the app is static Go and the only listener is Caddy with its own TLS stack; it defers but does not remove the batched reboot (podman/crun/systemd still need one).
- _The cloud-init onboarding win._ **Newly true and material** — stock template + cloud-init genuinely reduces requirement-1 friction versus shipping an OVA/ISO, and cloud-init's hypervisor/datasource coverage is broader and more ergonomic than Ignition's on Proxmox/vSphere specifically.
- _"The application upgrade is the expensive event and the host reboot is the cheap one."_ **Overturned** (§F3). A reboot stops Piri and honours the drain, so it is a strict superset of a Piri restart: 2–7 minutes plus a consumed PDP window, versus ~1–3 seconds for a container swap.
- _"Bake the infra tier into the OS image to enforce a slow cadence."_ **Proposed and then withdrawn within the same discussion** (§F3). Correct for the kernel/systemd/podman tier; wrong for Caddy, which is the most exposed component and belongs in the fast tier; unnecessary for Postgres, where an unlabelled digest-pinned unit gives the same deliberateness without reboot coupling.
- _Implicit assumption that an OS rollback restores a known-good system._ **Overturned** (§F4). It restores `/usr` and `/etc` only; container images, `/var/usrlocal` scripts and app data do not roll back, and the upgrade timer re-converges forward within ten minutes.

**Survive unchanged:**

- Single VM = single point of failure; no tool changes this.
- Piri/Ingot/Postgres/Vault/Caddy cannot run two concurrent instances → upgrades are in-place restarts; **zero-downtime remains aspirational under every option** (Kamal, Nomad, K3s all included). This is a property of the _workload_, not the deployment tool.
- Trust boundary → keep the funded delegated wallet key off the box (piri-signing-service), no durable Vault in the partner VM, outbound-only + Tailscale/Headscale break-glass. **The relaxation does not touch this and it remains the dominant design constraint.**
- Piri's up-to-60-min graceful shutdown and the PDP gate on every restart/reboot.
- Rebase Piri/Ingot off floating tags onto digest-pinned distroless/static — still correct regardless of OS.

**Need re-verification:**

- Whether openSUSE Leap Micro's auto-rollback (boot-counting/`health-checker`) is as unattended as FCOS greenboot **on the specific hypervisors partners use** — verify on real Proxmox/KVM before committing.
- Terraform **Stacks** on-prem (Terraform Enterprise) availability if FilOne won't use HCP Terraform — GA was HCP-first.
- Exact TuxCare/Ubuntu Pro **volume pricing** at 300 (list prices used above; both discount heavily).
- **`readlink -f /usr/local` on the pinned FCOS release.** The `/var/usrlocal` symlink is the documented rpm-ostree default, but FCOS could set `opt-usrlocal` differently in its treefile. One command settles it, and finding A in §F4 depends on it.
- **Whether Caddy supports systemd socket activation** (`LISTEN_FDS` inheritance). Determines whether the fast-tier proxy swap can be made connection-preserving.
- **Which FCOS stream release crosses podman 5.8/6.0**, and whether issue #28216 is fixed by the time you cross it.

### E4. Staged recommendation and bake-off

**Regions 1-5 (now):** Keep it boring. **Quadlet + systemd** on a single chosen mutable distro with native rollback (**openSUSE Leap Micro** preferred, or Ubuntu 24.04 with a ZFS-BE or snapper layer). **Terraform for central resources only.** **ansible-pull** for in-VM convergence. Digest-pinned images + cosign + release manifest carried over unchanged. Minimal-delta path; keeps the option to fall back to bootc.

**Regions 5-30:** Introduce **Terragrunt** for per-region state. Harden the pull loop (Alloy already ships logs/metrics out). Decide the OS-rollback mechanism definitively based on bake-off results. If HashiCorp-stack momentum is real, **pilot Nomad client-only in 2-3 regions** in parallel with the Quadlet baseline. Add **snapshot.ubuntu.com pinning** (if Ubuntu) or a Packer image floor to nail R7.

**Regions 30-300:** This is the declarative-fleet inflection. Commit to **either Nomad (Terraform-submittable, outbound-only, exec/logs via server)** or stay on Quadlet+ansible-pull with a strong central inventory and drift dashboard. Add a **pull-through package mirror/cache** for bandwidth and blast-radius. Evaluate **Terraform Stacks** for the central-resource layer if on HCP Terraform.

**Bake-off, concrete pass/fail for the newly-viable candidates:**

| Candidate                     | PASS criteria                                                                                                                                                                   | FAIL trigger                                                                       |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| openSUSE Leap Micro + snapper | Corrupt a kernel update on a Proxmox VM → box auto-boots previous snapshot with **zero human action**; PDP window honoured                                                      | Requires console/GRUB selection to recover                                         |
| Ubuntu + ZFS-BE + zfsbootmenu | Same, and remote rollback over SSH works from the break-glass overlay                                                                                                           | ZFS-on-root setup not reproducible via cloud-init/Packer                           |
| Nomad client-only             | `count=1` in-place restart honours a PDP pre-check gate; `alloc exec`/`logs` work with **no inbound port** on the client; 3 federated servers survive one region's network loss | Any operation needs inbound to the client; server quorum fragile                   |
| Packer golden image           | One `packer build` emits pinned qcow2 + OVA + Proxmox template, cosign-signed, reproducible; cloud-init time cut materially                                                     | Per-format sprawl unmanageable, or image+ansible-pull double-manage the same files |
| snapshot.ubuntu.com pinning   | staging and prod install byte-identical package sets from a pinned snapshot ID across a simulated month of archive drift                                                        | Snapshot unavailable for a needed timestamp; retention gap                         |
| NixOS (stretch)               | Two boxes from the same flake.lock are bit-reproducible; `--rollback` recovers a bad generation unattended; a non-Nix engineer can operate it from a runbook                    | Team cannot operate it without the one Nix expert                                  |

### E5. New open questions and risks introduced by the relaxation

1. **Unattended-rollback parity.** Can any mutable-distro mechanism match FCOS greenboot's _fully automatic_ rollback on partner hardware, or do we accept "partner picks a snapshot at GRUB" as the recovery path for the unbootable case? This is the biggest new risk and must be answered in the bake-off.
2. **Drift ownership at the seams.** With Terraform (central) + ansible-pull (OS/infra) + Quadlet (app), which layer owns each file? An unowned or double-owned Caddyfile/Quadlet unit is the most likely source of a 3am surprise.
3. **Package-mirror operational burden at 300.** If snapshot.ubuntu.com is unavailable or you're on a distro without it, running Pulp/aptly/reprepro is new infrastructure with its own CVE surface.
4. **Tool-sprawl cognitive load.** Every per-layer tool is a bus-factor risk across 300 regions. The relaxation _invites_ sprawl; the discipline of "one OS, one convergence tool, one app runtime, one owner per file" must be enforced deliberately.
5. **Central control-plane blast radius.** Nomad/K3s servers or HCP Terraform become a new single-point-of-failure for the fleet's _manageability_ (not its data plane). A region keeps serving if the control plane is down, but you lose the ability to push fixes — which collides with requirement 2 (timely fixes).
6. **Live-patch false comfort.** Buying live patching may create a false sense that reboots are gone; they are not (podman/crun/systemd). Don't let it erode the batched-reboot discipline.

## F. The immutable path, re-examined

_Added after a follow-up discussion on 2026-08-06. The trigger was a narrow question — can Quadlet+systemd run on an immutable distro? — but working the per-layer update mechanics through surfaced findings that partly redirect this review's recommendation, and exposed defects in the RFC's sample units that hold regardless of which OS is chosen._

### F1. Quadlet is native to an immutable OS; placement is the cadence lever

Yes, and it is the intended pairing rather than a workaround. "Immutable" on FCOS/bootc constrains only `/usr`. Quadlet's rootful search paths are `/etc/containers/systemd/` and `/usr/share/containers/systemd/`, plus ephemeral `/run/containers/systemd/`, with `/run` shadowing `/etc` shadowing `/usr`; the documented convention is that the distribution ships packaged units under `/usr` and the administrator puts units under `/etc`. Podman is a core component of the FCOS image, so there is nothing to install.

That yields three placements, and **choosing between them is how each layer gets a different update cadence:**

| Placement                                                  | Mutable at runtime? | Rolls back with the OS?               | A change requires         |
| ---------------------------------------------------------- | ------------------- | ------------------------------------- | ------------------------- |
| `/etc/containers/systemd/`                                 | Yes                 | Yes (per-deployment `/etc`)           | `daemon-reload` + restart |
| `/usr/share/containers/systemd/` (baked via Containerfile) | No                  | Yes                                   | New OS image + reboot     |
| bootc **bound images**                                     | No                  | Yes, _including the app image itself_ | New OS image + reboot     |

**bootc bound images** are worth knowing because they are a native implementation of the RFC's "signed OCI release manifest pinning OS digest plus every container digest". Each image is declared in a `.image` or `.container` file and selected by symlinking it into `/usr/lib/bootc/bound-images.d`; during `bootc upgrade` or `bootc switch` the bound images are pulled into bootc image storage and consumed via `GlobalArgs=--storage-opt=additionalimagestore=/usr/lib/bootc/storage`. Limitations to note: only the `Image` field is parsed, rootless is unsupported, and the images must be present in `/var/lib/containers` when `bootc install` runs. This is bootc-specific — a mild argument for bootc over FCOS.

**Two ostree behaviours to know before committing units to `/etc`:**

1. _Merge timing._ OSTree does not perform a second `/etc` merge on reboot, so modifications made after a pending deployment is created would otherwise be lost. Staged deployments exist to counter exactly this, delaying the 3-way merge until reboot or shutdown via `ostree-finalize-staged.service`. FCOS stages by default, so this is covered — but verify it in the bake-off, because anything that un-stages the flow makes config changes between staging and reboot vanish silently.
2. _`/etc` is per-deployment, `/var` is shared._ This is the root of §F4.

**Conclusion for requirement 6's "different approach per layer" question:** you can give each layer a distinct update cadence _without adding a second tool_, purely by choosing where the unit file lives. That is strictly cheaper in mental models than any of the multi-tool combinations in section D, and it is available only on the image-based OS.

### F2. Update flow, layer by layer, on bootc/FCOS

**OS — kernel, libraries, container runtime.** There is no package-level update path and, importantly, **no separate runtime update path**: kernel, systemd, glibc, podman and crun all ship inside one image, so "update Podman" _is_ "update the OS". Flow: zincati or `bootc-fetch-apply-updates.timer` fetches the new release → rpm-ostree/bootc stages it into the inactive slot → **stage but do not auto-reboot** → `pdp-gate` clears → reboot in the scheduled window → greenboot health-checks on boot, and failure boots the previous deployment unattended. Cadence: days, batched — the batched CVE tier unchanged. Rollback: automatic, unattended, and the property the mutable-distro option obliges you to rebuild. Discipline: no layering, no kexec, no live patching.

**Infra services — Postgres, Vault, Caddy.** Mechanically a tag or digest change plus a unit restart, but all three want different handling and treating them as one group was the error corrected in §F3. Auto-update selection is **by label only**: Podman checks only containers carrying `io.containers.autoupdate`, ignores unlabelled ones, and a single run acts on every labelled container. **Label discipline is therefore the tier boundary.** Caddy owns `:443` and the ACME account state; Postgres carries the major-version data-directory trap; Vault should not be durable on the box at all.

**Forge services — Piri, Ingot.** The fast tier, as already corrected in the RFC's Piri review: CI builds `CGO_ENABLED=0`, cosign-signs, pushes; promotion moves the `:prod` channel tag; on-box `filone-upgrade.timer` fires every 15 minutes → `pdp-gate` → `snapshot-piri-db` → `podman auto-update --rollback`, with `podman-auto-update.timer` masked so nothing runs ungated. `Notify=healthy` (no sd_notify in the tree, and it is what makes `--rollback` fire on a bad image), `StopTimeout=300` / `TimeoutStopSec=330`, and acceptance defined as one proof submitted on the new version rather than the unit going active. Signature enforcement via `/etc/containers/policy.json` means a partner-swapped image will not start.

### F3. Do not bake Caddy or Postgres into the OS image

This reverses a suggestion made earlier in the same discussion, and the reversal is the substantive point.

**The cost is understated in the RFC.** See Key Finding 6: patching Caddy through an OS promotion costs 2–7 minutes of regional outage and one PDP-safe window, against ~1–3 seconds for a container restart. That is not a 2× difference.

**Caddy is the worst possible candidate for the slow path.** It is the only publicly listening process, internet-facing, on hardware FilOne does not control — the single highest-exposure component in the stack. Baking it demotes the most exposed component to the slowest patch path, and directly contradicts the RFC's own two-tier table, which lists Caddy in the fast tier. The CVE stream is not hypothetical: Caddy inherits the Go TLS/HTTP2 surface (the Rapid Reset and CONTINUATION-flood class hit Go servers broadly and required rebuilds against a patched toolchain), plus quic-go if HTTP/3 is enabled. Second-order effect: under TLS-ALPN-01 renewal is coupled to Caddy serving correctly and fails silently until expiry, so a Caddy defect that breaks renewal would need an OS promotion to fix — lengthening MTTR on a failure mode the design already flags as hard to see.

**The steelman for baking, and why it loses.** Baking (a) deletes the auto-update footgun by construction, since Caddy would structurally have no independent update path; (b) gives R6/R7 purity, the OS digest fully determining host config; (c) removes an entire unexpected-restart class, so `:443` has exactly one reason to close; (d) resists drift, a unit in `/usr` being read-only where `/etc` is partner-writable. But (a) is the only real motivation and it has a far cheaper fix — **do not label Caddy, and give it its own gated upgrade unit**, which is precisely what change 7 of the Piri review already prescribed. And (d) is weak here: the threat model concedes the partner can modify anything, and cosign policy rather than file permissions is what stops them running an image FilOne did not sign.

**Postgres is a different case and should not have been paired with Caddy.**

|                           | Caddy                                 | Postgres                                              |
| ------------------------- | ------------------------------------- | ----------------------------------------------------- |
| Exposure                  | Only internet-facing process          | localhost / container network only                    |
| CVE relevance             | Pre-auth remote; Go TLS/HTTP2 surface | Mostly authenticated or extension-specific            |
| Cost of a careless update | Restart, seconds                      | Major-version data-dir change; cluster will not start |
| Natural cadence           | Reactive, hours                       | Deliberate, rare                                      |

For Postgres, forcing changes through a staged, gated promotion _is_ a genuine safety property, because the failure mode is catastrophic and awkward to reverse and you want the restore rehearsal inside the same change window. But baking is a heavier mechanism than the goal needs: a digest-pinned unit in `/etc`, unlabelled for auto-update, with a hand-invoked upgrade unit achieves the same deliberateness without coupling a Postgres patch to a Piri drain.

**Verdict.** Keep both as containers with units in `/etc`, unlabelled for auto-update, each with its own gated upgrade unit — Caddy's firing reactively on a security bump, Postgres's invoked by hand inside a planned window with a rehearsed restore. Reserve image-baking for components that genuinely have no independent patch story: kernel, systemd, podman/crun — that is, the batched tier exactly as originally scoped. The narrower claim from §F1 stands: placement is a legitimate cadence lever. It was simply applied to the two services where the cadence it enforces is wrong.

**Two things worth chasing that would improve the fast path:**

- _Socket activation for the proxy._ If `:443` is owned by a systemd `.socket` unit rather than by the container, swapping the Caddy image does not close the socket — inbound connections queue in the kernel accept backlog across the swap, turning a Caddy patch from ~1–3s of refused connections into ~1–3s of added latency. That matters specifically for the pre-signed-URL-in-a-browser case, which has no retry. Podman/Quadlet supports socket activation; **whether Caddy consumes inherited descriptors via systemd's `LISTEN_FDS` protocol is unverified** and is the thing to establish before designing around it.
- _Pre-drain before reboot._ Since every reboot already pays Piri's drain, the OS upgrade unit should drain Piri deliberately _before_ invoking the reboot, inside the gated window, rather than discovering the drain during shutdown. This decouples reboot duration from load and makes the batched-tier window predictable, which the current design assumes but does not ensure.

### F4. Rollback coherence: an OS rollback does not roll back container images

Verified against bootc's documentation, which distinguishes "floating" images (fetched by podman-systemd) from logically bound ones and states the lifecycle difference explicitly. Floating images are managed by the user through `podman auto-update` and `podman image prune`, can be acted on at any time independent of host upgrades or rollbacks, and **host upgrades or rollbacks do not affect the set of images**. Bound images are managed exclusively by bootc during upgrades, and those corresponding to rollback deployments are retained — so bound images are the documented mechanism for coupling app and OS lifecycles.

The filesystem mechanics corroborate it. Upstream ostree documentation notes that `/var` is shared between independent versions (which is why package databases must live in the per-deployment `/usr`); rootful podman's graphroot is `/var/lib/containers/storage`; `/etc` is per-deployment.

| Path                             | Rolls back with the OS?      | Contents                                |
| -------------------------------- | ---------------------------- | --------------------------------------- |
| `/usr`                           | Yes                          | kernel, systemd, podman, baked units    |
| `/etc`                           | Yes (per-deployment merge)   | Quadlet units, Caddyfile, `policy.json` |
| `/var/lib/containers/storage`    | **No**                       | container images, libpod database       |
| `/var/usrlocal` (= `/usr/local`) | **No**                       | `pdp-gate`, `snapshot-piri-db`          |
| `/var/mnt/filone`                | **No** (correct and desired) | SQLite, Postgres, certs, ACME state     |

**Finding A — the gate scripts are in the wrong place, and this is the most serious item in this review.** rpm-ostree's treefile reference documents `opt-usrlocal` defaulting to `var`, meaning `/opt` and `/usr/local` are symlinks into `/var` and are purely machine-local state; the maintainers describe that territory as never rolled forward or backward. The corrected units use `ExecStartPre=/usr/local/bin/pdp-gate` and `/usr/local/bin/snapshot-piri-db`, which are **host** paths and therefore resolve to `/var/usrlocal/bin/`. Three consequences in ascending seriousness: (1) they do not roll back, so a broken gate survives an OS rollback; (2) **they are covered by nothing** — cosign policy protects container images, not a shell script in `/var`, so the control that prevents slashing is a plain file in partner-writable shared state, and a partner wanting to avoid upgrade downtime can `exit 0` the gate undetected; (3) R7 has a hole, since the gate logic's version is pinned neither by the OS digest nor by the release manifest. **Fix:** ship both inside the OS image at `/usr/libexec/filone/` via the Containerfile — read-only, versioned with the OS, rolled back with it, and covered by the OS image signature. (`HealthCmd=/usr/local/bin/piri status` is fine as written; that path is inside the container.)

**Finding B — the upgrade timer actively diverges after a rollback.** `filone-upgrade.timer` carries `OnBootSec=10min`, so roughly ten minutes after a greenboot rollback, `podman auto-update` resolves `:prod` and pulls whatever the channel tag now points at. The box converges to **old OS + newest app images**, unattended, in the middle of the incident the rollback was meant to escape. If the rollback was triggered by an OS-N/app-M interaction, it manufactures an untested combination rather than restoring a tested one; nothing in the design ever restores a known-good _bundle_. This is a new argument for the option change 2 of the Piri review set aside: if the release-manifest ID lives in `/etc` (per-deployment), an OS rollback reverts the pointer and the agent re-converges _backward_ to the app versions tested against that OS. The channel tag is version-agnostic by design, which is exactly why it cannot participate in rollback. Keep the channel tag for regions 1–5, but record this as the property being traded away rather than leaving it implicit.

**Finding C — podman version skew across a rollback is a live hazard.** See Key Finding 8. Note this is a podman property rather than an OS-model property, so it applies on the mutable path too; the only thing the image-based OS changes is that the _deployment_ determines which podman touches the shared `/var` state.

**Finding D — two uncoordinated rollback domains.** greenboot rolls back the OS on failed boot; `podman auto-update --rollback` rolls back an image on failed health check. Neither knows the other exists — the bootc documentation says as much for floating images. Reachable states include OS-rolled-back-only, containers-rolled-back-only, and both at different times, and nothing currently records which combination is running or whether it was ever tested together. **Compensating control:** emit the running triple — booted deployment checksum from `rpm-ostree status`, each container's resolved image digest from `podman inspect`, and the release-manifest ID — and alert when the combination does not match a manifest that passed staging. The Piri review already noted that channel tags stop the unit file recording the running version; this is what turns silent into paged.

**One balance note:** this asymmetry is not a bootc weakness. It is inherent to "roll back the OS but keep the data", and it reappears on openSUSE, where the default layout puts `/var` on a subvolume excluded from root snapshots. Choosing a mutable distro does not avoid §F4 — it just means you rebuild the A/B mechanism _and_ still own the problem.

### F5. Defects to fix regardless of OS choice

These are RFC-level bugs the discussion surfaced. None of them depend on the mutable/immutable decision:

1. **`pdp-gate` and `snapshot-piri-db` must move** out of `/usr/local/bin` (= `/var/usrlocal`: partner-writable, unsigned, never rolled back) and into the OS image at `/usr/libexec/filone/`.
2. **`Volume=/data/piri` in the corrected `piri.container` is still wrong** — the provisioning review flagged that `/data` is a new top-level directory ostree will not carry into the next deployment. It must be `/var/mnt/filone/piri`.
3. **"The application upgrade is the expensive event and the host reboot is the cheap one" is inverted.** Restate it, and add pre-drain-before-reboot so the batched window is predictable.
4. **Add running-combination telemetry** (deployment checksum + image digests + manifest ID) and alert on any mismatch against a staged manifest.
5. **Serialise Quadlet unit startup** across the podman 5.8 migration boundary, or run a oneshot `podman system migrate --migrate-db` ahead of any container unit.

### F6. Additional bake-off items

| Item                       | PASS criteria                                                                                                                                  | FAIL trigger                                                  |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Rollback coherence         | Pull a new Piri image, force an OS rollback, wait 15 min: the running combination is one the release manifest recognises, and a mismatch pages | Box silently settles on an untested OS/app pair               |
| Gate-script integrity      | `pdp-gate` is read-only, versioned with the OS, restored by an OS rollback; tampering fails closed and pages                                   | Gate editable from the partner's shell without detection      |
| podman migration boundary  | Cross podman 5.8 in staging with all Quadlet units starting concurrently; `podman ps` sees every unit systemd reports active                   | Any container invisible to podman while its service is active |
| Caddy socket activation    | Swapping the Caddy image adds latency but refuses no connections; a pre-signed URL loaded mid-swap succeeds                                    | Caddy cannot consume inherited `LISTEN_FDS`                   |
| `/etc` merge under staging | A unit-file edit made between staging and reboot survives the reboot                                                                           | Edit silently lost — staging not in effect                    |

## Recommendations

**Per layer, given "mutable but code-driven, Terraform-preferred":**

- **OS — take the minimal relaxation first.** Section F makes this the default rather than a footnote: **keep bootc/FCOS and relax only the no-layering rule.** You retain unattended A/B rollback, and you keep the placement lever (§F1) that lets each layer have its own cadence with no additional tooling. If you do leave, pick **one** distro with native near-unattended rollback — first choice **openSUSE Leap Micro/MicroOS (btrfs+snapper boot-into-snapshot)**, strong alternative **Ubuntu 24.04 + snapshot.ubuntu.com** (best R7 story) _with_ a ZFS-BE or snapper layer added; do not run Ubuntu without a rollback layer. **Do not adopt live kernel patching as a headline feature** — take Livepatch only because it rides along with Ubuntu Pro, and skip paid TuxCare for this workload.
- **Infra services (Postgres/secrets/Caddy):** Keep on the same runtime as the app (Quadlet units in `/etc`), Caddy as the long-lived :443 owner via TLS-ALPN-01, secrets rendered short-lived into tmpfs from central Vault/OpenBao. Do **not** run durable Vault in the partner VM. **Do not bake Caddy or Postgres into the OS image** (§F3): leave both unlabelled for auto-update and give each its own gated upgrade unit — Caddy's reactive on a security bump, Postgres's hand-invoked inside a planned window with a rehearsed restore. Chase socket activation for the proxy if Caddy supports `LISTEN_FDS`.
- **App (Piri/Ingot):** **Podman Quadlet + `podman auto-update`** as the default, driven by the gated `filone-upgrade.timer` with `podman-auto-update.timer` masked. Migrate to **Nomad client-only** at the ~30-region inflection _if_ FilOne wants a declarative, Terraform-submittable fleet control plane and will run the servers. **Reject Kamal.**
- **Alloy — the one thing worth binding.** bootc's own stated use cases for bound images are log forwarders, monitoring, config-management agents and security agents, which describes Alloy exactly, and binding guarantees it is present at boot without working networking — the property that matters most during the outages you want telemetry for. Bind Alloy; keep Caddy floating. These are consistent positions, not a contradiction: bind what should be slow and must always be present, float what must be patched in hours.
- **Terraform:** central resources only (DNS, Vault/OpenBao, Grafana Cloud, registry, inventory); Terragrunt for per-region state now, evaluate Stacks at ~30 regions on HCP. Terraform should **not** reach into the partner box; use the Nomad provider (talk to the orchestrator API) if you want Terraform-driven app deploys.
- **In-VM convergence:** **ansible-pull (outbound-only)**, one owner per file, logs shipped via Alloy. On bootc this shrinks further — much of what ansible-pull would converge is simply part of the image.
- **Do these five things regardless of the OS decision** (§F5): move the gate scripts into the image, fix the `/data/piri` volume path, restate the reboot-cost inversion and add pre-drain, add running-combination telemetry with alerting, and serialise Quadlet startup across the podman migration boundary.

**The single highest-regret decision in this space:** **abandoning unattended A/B rollback.** Everything else the relaxation unlocks is convenience; the rollback safety net is the one property that, once lost, turns a bad 3am update into a partner-console incident across regions nobody at FilOne can physically reach. Whatever OS you pick, **prove automatic rollback works on the partners' actual hypervisors before you ship region 1** — or keep bootc's A/B and relax only the no-layering rule.

**The cheapest high-value fix in this document** is unrelated to the OS question: `pdp-gate` currently sits in partner-writable, unsigned, never-rolled-back state (§F4, finding A). Slashing protection should not be a file the adversary can edit. Move it into the image before region 1.

## Caveats

- Pricing figures (Ubuntu Pro $500/server/yr, $25/workstation/yr, free ≤5 machines; TuxCare KernelCare ~$50/server/yr + LibCare ~$34.50; RHEL ~$879 Standard; Oracle Premier ~$1,399/CPU-pair) are **list prices** from vendor and vendor-comparison pages; all discount at volume — treat the 300-machine extrapolations as order-of-magnitude.
- TuxCare comparison figures come substantially from **TuxCare's own** competitive pages and should be read as vendor-favourable; the direction (TuxCare much cheaper, broader distro coverage) is corroborated but exact multipliers are theirs.
- Kamal's inbound-SSH requirement, no-pull-mode, and single-instance downtime degradation are verified against 37signals' official docs and maintainer discussion; the "no pull mode" is an inference from complete absence in the official docs.
- Terraform Stacks GA (HashiConf 2025, Sep 25 2025) is HCP-first; on-prem Terraform Enterprise support was still trailing at announcement — verify before assuming Stacks for a self-hosted setup.
- openSUSE auto-rollback (boot-counting) parity with FCOS greenboot on Proxmox/KVM/vSphere is **asserted from documentation, not tested** — this is the top item to validate in the bake-off.
- NixOS reproducibility/rollback claims are well-established; the team-adoption-cost judgement is qualitative and depends on FilOne's staffing.
- **Section F additions.** The floating-vs-bound image lifecycle, the Quadlet search paths, the `/etc` merge and staged-deployment behaviour, the `opt-usrlocal` default, and the podman BoltDB→SQLite timeline are all **verified against upstream documentation** (bootc, podman, ostree, rpm-ostree, Fedora). Three things are **not** verified and are flagged inline: whether Caddy consumes inherited `LISTEN_FDS` for socket activation; whether the pinned FCOS release keeps the `opt-usrlocal: var` default (`readlink -f /usr/local` settles it); and whether podman issue #28216 is still open by the time your stream crosses 5.8.
- The 2–7 minute reboot figure in §F3 is **derived from the configured timeouts** (`TimeoutStopSec=330`, `HealthStartPeriod=60s`) plus a 30–90s boot, not measured. Piri's real drain duration under load is already a load-bearing open question in the RFC; measure it before treating the range as authoritative.
- The Go HTTP/2 CVE examples (Rapid Reset, CONTINUATION flood) are cited to illustrate the _class_ of exposure Caddy inherits, not as an exhaustive or current advisory list. Check Caddy's own security advisories when setting the patch SLO.
- Nomad client↔server port directions and exec/logs-via-server behaviour are from HashiCorp docs; validate the exact firewall rules against your chosen Nomad version before committing.

---

# 2026-08-06 Running a Few Services on One Box in 2026: What to Use Besides Kamal

_Includes a continuous-deployment design for the two recommended runtimes (Docker Compose and Podman + Quadlet), added after the original survey._

## TL;DR

- **For the simplest, most durable single-host setup, use plain Docker Compose (or Podman + Quadlet if you want systemd-native, rootless, daemonless containers) fronted by Caddy or Traefik.** These have essentially zero platform overhead, no control plane, and the largest communities. Kamal is an excellent _deploy_ layer on top of Compose-style workloads but is Rails-optimized and registry-dependent.
- **If you want a web UI, pick Dokploy or Coolify** (full self-hosted PaaS, Docker + Traefik) or **Komodo** (GPL-3.0, GitOps-friendly, agent-based) or **Portainer CE/Dockge** (lightweight management UIs). Avoid CapRover for new builds — it is tied to Docker Swarm, which is frozen, and its own release cadence has slowed.
- **Avoid dead ends:** Docker Swarm is stable but in maintenance mode with no active feature development; Watchtower was archived December 17, 2025 (use Diun for notifications or the nickfedor/watchtower fork); several PaaS entrants (Ptah.sh) are archived. Nomad is fine single-node but is BUSL-licensed, now an IBM product, and has no meaningful open-source fork — it's usually more machinery than a handful of services needs.
- **For continuous deployment, prefer a pull loop over a CI push.** CI builds the image, pushes it, and commits the resulting digest into a central git repo; each node runs a systemd timer (optionally poked by a webhook) that reconciles its own directory. No inbound ports, no long-lived credential in CI that can reach the box, and git becomes the audit log of exactly what ran when.
- **The highest-value ~20 lines of the whole setup are failure handling:** health-gated apply, automatic revert to the last-good commit, and _quarantining_ the bad SHA so the loop can't flap. Add a staleness ("deadman") alert, because a dead timer looks exactly like a timer with nothing to do.

## Key Findings

**The 2026 consensus for "a few services on one box" is: don't reach for an orchestrator.** Across Hacker News, r/selfhosted, and practitioner blogs, the dominant advice is plain Docker Compose or Podman Quadlet, optionally with a thin deploy tool (Kamal, Dokku) or a management UI (Portainer, Dockge, Komodo). Kubernetes/K3s is repeatedly described as "the Kubernetes tax without the Kubernetes dividend" for teams that fit on one or two servers.

**Container-native, no control plane (best fit for the query):**

- **Docker Compose** — The default. Single YAML, ~36K GitHub stars, universal community support. Single-host oriented. No built-in zero-downtime deploy or secrets vault, but the most documented option on earth. Pair with a reverse proxy for TLS.
- **Podman + Quadlet** — The rising OS-native alternative. Quadlet (merged into Podman 4.4, Feb 2023) lets you declare containers as systemd units; rootless, daemonless, and integrated with journald/systemctl. Podman 5.4.2 ships in Debian 13 "trixie" (released August 9, 2025 — "After 2 years, 1 month, and 30 days of development, the Debian project is proud to present its new stable version 13"). `podman auto-update` + `podman-auto-update.timer` handles image refreshes. `podman-compose` exists but is a thin, less-maintained translation layer — Quadlet is the recommended path. `podman kube play` can run Kubernetes YAML locally if you want a K8s-compatible manifest without a cluster.
- **Kamal (2.x)** — 37signals' SSH-based, imperative, zero-server-overhead deploy tool (roughly 11,000+ GitHub stars — meaningful but smaller than Coolify and Dokku). Kamal 2 replaced Traefik with `kamal-proxy`, added multi-app-per-host, automatic Let's Encrypt HTTPS, canary deploys, maintenance mode, and improved secrets (password-manager integration). Battle-tested running HEY. Limitations: requires a Docker registry, installs via Ruby (or a container), no built-in monitoring/log aggregation/DB management, and does not reschedule workloads across nodes or provide cluster self-healing. Best for web apps; awkward for non-web/stateful-only workloads.

**Self-hosted PaaS (web UI, more batteries):**

- **Coolify** — Most popular (over 59,300 GitHub stars as of July 2026, the most-starred self-hosted PaaS on GitHub), Apache-2.0, PHP + Traefik. v4.0.0 went stable on April 27, 2026 after nearly two years of beta; v4.1.0 (May 2026) added Railpack builds, audit logging, MCP; v5 with multi-server is being built. 280+ one-click templates. Costs ~500–700 MB RAM and 5–6% idle CPU before you deploy anything. Multi-server scaling uses Docker Swarm. An independent audit reportedly flagged seven CVEs (see caveat).
- **Dokploy** — Younger (~24K stars), TypeScript + Traefik. Cleaner UI, stronger built-in monitoring, first-party MCP server, first-class Compose/Swarm support. Smaller template library and community than Coolify.
- **Komodo** — GPL-3.0, Rust + TypeScript, ~10K stars. Core + Periphery agent architecture; multi-host fleet management, GitOps via TOML resource sync, no node caps or paid tier. v2.0 (early 2026) switched to PKI (Ed25519) auth. You supply your own reverse proxy/TLS. More moving parts (Core + database + per-host agent).
- **Dokku** — The veteran "mini-Heroku" (since 2013), Go, git-push workflow, Heroku buildpacks or Dockerfiles, excellent docs, minimal resource use. Single-server only (no native multi-host). Still actively maintained and beloved for solo/CLI-first single-box use.
- **CapRover** — Docker Swarm-based PaaS with UI and one-click apps. GitHub activity has slowed considerably; last release was over a year old at one point; architecturally locked to frozen Swarm; the Captain instance is a single point of failure. Not recommended for new production builds.

**Management UIs (manage Compose/containers, not full PaaS):**

- **Portainer CE** — The most stable, broad web UI (Docker/Swarm/K8s/Podman). Free CE exists, but the free _Business Edition_ is capped at 3 nodes (a hard limit) and gates RBAC/SSO/audit-log behind paid tiers. CE can also back a Compose stack with a git repo (poll or webhook), which makes it a zero-scripting GitOps option — see the CD section.
- **Dockge** — MIT, single Node service, ~15K stars, tidy Compose-stack UI for one or two hosts, single-maintainer.
- **Swarmpit / Portainer for Swarm** — Swarm-oriented UIs; only relevant if you commit to Swarm (not advised for new builds).

**Ingress / TLS layer:**

- **Caddy** — Simplest; automatic HTTPS with Let's Encrypt + ZeroSSL failover, two-line config. Strongest ACME implementation of the three. Best for simple single-VPS setups.
- **Traefik** — Docker-label-driven dynamic discovery; the de facto choice for container-heavy/multi-service setups; steeper learning curve.
- **Nginx Proxy Manager (NPM)** — GUI over nginx; easiest for beginners, but bundles its own stack and has lagged upstream security fixes (e.g., CVE-2025-50579). Keep it updated and never expose the admin UI.
- **kamal-proxy** — Kamal's own lightweight proxy; not a Traefik replacement for general use (no middleware/rate-limiting/compression).

**Image updates:**

- **Watchtower is archived (Dec 17, 2025)** — the original containrrr/watchtower repo (24.7K stars at freeze) is read-only, no more security patches; also incompatible with Docker Engine ≥28/29. Migrate to **Diun** (notify-only, safest), or the actively maintained **nickfedor/watchtower** fork (github.com/nicholas-fedor/watchtower — "a drop-in compatible fork with continued active development, and the one we've seen the most adoption of") for drop-in auto-updates. Podman users get `podman auto-update` natively.
- **Renovate** is the better fit if you pin digests in git: it opens digest-bump PRs for third-party images (Postgres, Caddy, Redis) so you get version discovery with a human gate and no auto-puller on the box. Native support for `docker-compose.yml`; Quadlet files need a `customManagers` regex rule.

**Continuous deployment (added):** the design space has two independent axes — **who initiates the connection** (node pulls vs. CI pushes) and **what is the source of truth for the running version** (git, the registry, or a CI job's ad-hoc decision). Most durable setups are a hybrid: CI writes to git or the registry, the node reconciles from there, and CI never touches the node. See "Continuous deployment for Compose and Quadlet" below.

**Non-container / OS-native options:**

- **systemd units + Podman Quadlet** — see above; the cleanest containerized-but-systemd-native option.
- **systemd-nspawn / machinectl** — lightweight OS-container/VM-lite via systemd; niche but robust for full-OS-tree isolation.
- **systemd portable services / systemd-sysext** — Portable services attach a service + its dependencies from a disk image with sandboxing; sysext overlays additional binaries onto /usr atomically. Real production users exist (e.g., a Go+SQLite side project used by the DOE Science Bowl runs on a single Debian 13 VPS via portable services with Litestream S3 replication and Let's Encrypt). Good for non-container, single-artifact, version-controlled deploys.
- **NixOS + Colmena / deploy-rs / morph** — Fully declarative host config with atomic rollbacks (boot into the previous generation). Colmena (Rust, nix-community) is a stateless, parallel, SSH-push deployer; deploy-rs and morph are peers; NixOps is the older option. Steep learning curve but the strongest rollback and reproducibility story. Colmena deploys only to hosts already running NixOS.
- **Ansible** — Push-based over SSH, no agent, imperative-but-idempotent. Great for provisioning a single host and laying down Compose files / systemd units; commonly used to sync config across manually-run Dokku or Compose hosts. In **`ansible-pull`** mode it becomes a pull-based CD loop (see CD section).
- **LXC / Incus** — System containers (VM-like) rather than app containers. **Incus** is the community fork of LXD created after Canonical took LXD in-house in 2023; it's Apache-2.0, CLA-free, maintained by the original LXD team under Linux Containers, and now packaged in Debian stable, Fedora, openSUSE, and NixOS. LXD continues under Canonical but ships primarily via Snap and is best on Ubuntu. For a non-Ubuntu host in 2026, Incus is the recommended choice.

**Nomad (single-node):** Technically easy — one Go binary in combined server+client mode — and has a devoted small-scale following as a lighter-than-K8s orchestrator. But HashiCorp itself flags single-server as non-production; Community Edition is BUSL/BSL-licensed (relicensed Aug 2023); IBM completed its acquisition of HashiCorp on February 27, 2025 (per IBM's FY2025 Form 10-Q, shareholders "received $35 per share in cash, representing a total equity value of approximately $7.2 billion"; the announced enterprise value was ~$6.4B). Nomad is being repackaged as an IBM product, and starting April 2026 it adopts IBM's V.M.F versioning with a Nomad 2.0 coming (current line is 1.10.x LTS, with 1.11.x released). Crucially, **there is no significant open-source fork** analogous to OpenTofu (Terraform) or OpenBao (Vault) — only tiny, low-activity attempts (e.g., OpenNood). For a literal handful of services on one box, Nomad is usually more than you need.

**Newer entrants (2025–2026):** **Haloy** (Go, CLI + lightweight daemon, registry-optional layer-caching deploys), **Uncloud** (Go, WireGuard-mesh multi-host Compose, explicitly not production-ready yet, ~4.8K stars), **Disco** (self-hosted Heroku-style), **Canine** (self-hosted Kubernetes PaaS — skip given the K8s exclusion), **Sliplane** (managed, not self-hosted). **Ptah.sh is archived/discontinued** — its maintainer abandoned it after finding Docker Swarm "very unstable and unreliable" and announcing a pivot to Kubernetes/a hand-crafted orchestrator.

## Details

### Why not Kubernetes/K3s (confirming the user's instinct)

The user has ruled out K8s/K3s, and the 2026 field agrees for this scale. Practitioner writeups like "Kubernetes Was Overkill. We Moved to Docker Compose" describe eight-engineer teams "spending 60 hours a week managing Kubernetes instead of shipping features." The high-profile Gitpod "We're Leaving Kubernetes" piece drew a useful counterpoint on Lobsters: much of the complexity is in your application and "you can make and not make that mess regardless of using or not using Kubernetes." The threshold rule that recurs: if you don't have a distributed-systems problem — if your services fit on one or two boxes — an orchestrator is overhead. The one legitimate exception people cite is _learning_ K8s, or crossing ~25+ containers across 3+ machines where you genuinely need cross-node scheduling (at which point K3s, not full K8s, is the lighter step).

### The two "boring and correct" choices

**Docker Compose** remains the pragmatic default. It's single-host-oriented, universally documented, and everything else in this space is built on it or interoperates with it. Its gaps — no native zero-downtime rollout, no secrets vault, no rollback beyond re-deploying a previous image tag — are real but easily patched with a reverse proxy and disciplined image tagging. A representative homelab view: on a resource-constrained single node, "there is NO BETTER SOLUTION… little, to no overhead."

**Podman + Quadlet** is the choice if you value OS-native integration. Because each container is a real systemd unit, you get lifecycle management, dependency ordering, journald logging, `systemctl status`, and auto-restart for free — with rootless, daemonless operation. Practitioners report the logging and status story is _easier_ than Compose. Two caveats appear repeatedly: (1) rootless services need `loginctl enable-linger` or they won't start until the user logs in; (2) there is no clean automated migration path from Compose to Quadlet — you rewrite units by hand, and Quadlet "is absolutely not a solution for local-dev." `podman generate systemd` is deprecated in favor of Quadlet.

### Kamal's real limits for the user's case

Kamal 2 is genuinely excellent for a _fresh_ web app on a fresh server. But it is a deployment tool, not a platform: it won't self-heal across nodes, requires a Docker registry (an extra moving part if you don't already push images), installs through Ruby, and leaves monitoring, logging, and database operations entirely to you. For non-Rails, non-web, or purely stateful workloads, it fits awkwardly. Its virtue — nothing runs on the server except your app and a small proxy — is also why it does less than a PaaS.

### Zero-downtime, rollback, secrets, and multi-host at a glance

- **Zero-downtime deploys:** Built-in with Kamal (kamal-proxy), Coolify, Dokploy, Haloy, and Uncloud. With plain Compose you script it (start new, health-check, swap proxy, stop old) or accept a brief blip. NixOS does atomic activation but a service restart may blip.
- **Rollback:** Strongest with **NixOS** (boot previous generation) and image-tag-based tools (Kamal keeps prior containers; Coolify/Dokploy track deployments). Compose rollback = redeploy previous tag. Quadlet + `podman auto-update` supports rollback on failed health check.
- **Secrets:** Kamal 2 integrates password managers; Coolify/Dokploy have per-app secret stores; Compose relies on `.env`/Docker secrets; Colmena uploads secrets outside the Nix store. For a heavier need, **OpenBao** (the MPL-2.0, Linux-Foundation fork of Vault) is the open-source vault of choice in 2026 since Vault itself went BUSL.
- **Ingress/TLS:** Caddy (simplest auto-HTTPS), Traefik (dynamic, container-native), NPM (GUI). Kamal, Coolify, Dokploy, Haloy, and Uncloud bundle Let's Encrypt automation.
- **Persistent volumes/databases:** All Docker-based tools use named volumes/bind mounts. Watch for the classic footgun — never let an auto-updater silently jump a database major version (this is exactly why Diun's notify-only model is safer than Watchtower's auto-pull).
- **Multi-host growth path:** Compose → Uncloud (WireGuard mesh) or Komodo (agents) or Docker Swarm (if you must, but it's frozen); Kamal deploys to multiple servers over SSH natively; Coolify/Dokploy scale via Swarm. NixOS/Colmena and Ansible scale to fleets naturally over SSH.

---

## Continuous deployment for Compose and Quadlet

Neither Compose nor Quadlet ships a CD mechanism, so this is the part you assemble yourself. Two axes define the space:

1. **Who initiates the connection** — the node pulls, or CI pushes. Mostly a security and network-topology question.
2. **What is the source of truth for "which version runs"** — git (a digest committed to a file), the registry (a floating tag something watches), or a CI job's ad-hoc decision. Mostly a reproducibility question, and _orthogonal to the first axis_.

### Pull-based family

**Plain systemd timer + git + reconcile script.** The DIY baseline: a timer fires every 1–5 minutes, `git fetch`, and if `origin/main` moved and touched this node's directory, apply and restart. Needs only a read-only deploy key and outbound 443.

An important asymmetry between the two runtimes:

- **Compose mostly doesn't need change detection.** `docker compose up -d --remove-orphans --wait` _is_ a reconciler — it recreates only containers whose image or config hash actually differs, and `--wait` blocks on health checks and exits non-zero on failure. The loop is "pull, then `up -d --wait` per stack." Skip the diffing logic.
- **Quadlet does need it.** Writing unit files plus `systemctl daemon-reload` restarts nothing, so you need `git diff --name-only HEAD@{1} HEAD` mapped to units (`foo.container` → `foo.service`), and a changed `.volume` or `.network` means restarting dependents too.

_Pros:_ no inbound ports; no credential held by a third party; works behind NAT; keeps working if GitHub is down; trivially auditable; ~50 lines of bash.
_Cons:_ polling latency; no feedback path (the pusher doesn't learn whether it worked); you own failure handling; **silent stalls are the classic failure mode** — a timer that has been erroring for three weeks looks exactly like a timer with nothing to do.

**Webhook poke instead of polling.** A tiny socket-activated, HMAC-verified receiver (`adnanh/webhook`, or ~30 lines of Go) that only triggers the same reconcile unit. Seconds instead of minutes, while the node still pulls the content — so no CI-side credential can touch the node. _Cost:_ one inbound port, however small. Keep the timer as a fallback for missed hooks.

**`ansible-pull`.** Same shape, but the reconcile logic is a playbook run from the repo on a timer. You get idempotent modules for the rest of the host (users, firewall, packages, unit files, compose files), and **handlers give you "restart only what changed" for free** — precisely the hard part of the Quadlet case. _Cons:_ Python + Ansible on the node, slower runs, weak reporting in pull mode unless you wire up callback plugins.

**Portainer CE GitOps.** Back a Compose stack with a git repo; poll or webhook. Free in CE. _Pros:_ web UI, per-stack, zero scripting. _Cons:_ Compose only (no Quadlet), another daemon, and it invites out-of-band clicking that drifts from git.

**Komodo.** Purpose-built: TOML resource definitions in git, sync via webhook, Core + per-host Periphery agent. _Pros:_ the closest thing to Flux-for-Compose, multi-host ready, GPL-3.0, no node caps. _Cons:_ control plane + database + agent is a lot of machinery for one box, and you still supply the reverse proxy and TLS.

**Harbormaster** is the minimalist niche pick — reads a YAML list of git repos and runs their Compose files. Good if each service lives in its own repo.

### Push-based family

The auth problem is the whole game here. Roughly worst to best:

**Long-lived SSH key in a CI secret.** Common, and the thing to move away from. If you do it, at minimum: a dedicated `deploy` user with no shell, a forced command in `authorized_keys` (`command="/usr/local/bin/deploy",restrict ssh-ed25519 …`) so the key can only invoke one script, and a narrowly scoped sudoers entry for the specific `systemctl` calls. You still have a forever-valid credential held by a third party, guarding an inbound port.

**Same, but over a private network.** The official `tailscale/github-action` joins the runner to your tailnet using an OAuth client and an **ephemeral, tagged** node, so the runner leaves the tailnet when the job ends; your box then needs no public sshd at all, and ACLs restrict that tag to one host and one port. With Tailscale SSH you can drop SSH keys entirely and let tailnet identity plus ACLs be the authorization. Cloudflare Tunnel with a service token is the same shape. _Pros:_ removes the inbound port — worth a lot on bare metal, where you are the one patching sshd. _Cons:_ a dependency on a third-party coordination plane (self-hostable via Headscale, at a cost).

**SSH certificate authority with short-lived certs.** CI authenticates to OpenBao (or step-ca) via OIDC — **no static secret in CI at all** — and receives a 5–15 minute SSH certificate with a fixed principal and forced command. The node's sshd only needs `TrustedUserCAKeys`; it stores no per-client keys, and revocation happens by expiry. _Pros:_ no long-lived credentials anywhere, real identity, automatic expiry, scales cleanly to more nodes and workflows. _Cons:_ the most moving parts, and you now operate a CA.

**Invert it: self-hosted runner on the node.** The runner polls CI outbound over 443 — no inbound port, no credential for CI to hold, and you get a local build cache. _Cons:_ a runner is an arbitrary-code-execution surface by design. Run it as an unprivileged user with narrow sudo, and **never on a public repository** — fork PRs can execute code on self-hosted runners, which GitHub explicitly warns about.

### Registry-as-trigger family

**`podman auto-update` + `podman-auto-update.timer`** is the Quadlet-native answer and genuinely good: label a unit `AutoUpdate=registry`, the timer checks the registry, pulls on digest change, restarts the unit, and — uniquely, for free — **rolls back automatically if the new container fails its health check**.

The catch is structural: it only works on a _floating_ tag, so the registry, not your repo, decides what runs. That is fine if CI owns a tag like `prod` and moving that tag _is_ the deploy action — but then a `versions.conf` in git is decorative. Pick one source of truth deliberately.

**Watchtower's original repo is archived (Dec 2025)** and incompatible with Docker Engine ≥28 — use `nickfedor/watchtower` for the drop-in, or **Diun** for notify-only. Never point an auto-updater at a database container; a silent major-version jump is the canonical way to lose an afternoon or a dataset.

### The recommended hybrid: CI writes git, the node pulls

Combine both families so **CI never touches the node**:

1. CI builds and pushes `ghcr.io/org/app@sha256:…`.
2. CI commits that digest into the node's file in git (directly, or as an auto-merged PR).
3. The node's pull loop sees the change and reconciles.

You get CI's build power, git as the auditable record of exactly what ran when, no inbound ports, and no CI credential that can reach the box. The only loss is a tight feedback loop, bought back by having the reconcile script report status (a `deploy-status` file scraped by node_exporter's textfile collector, or a `systemd OnFailure=` unit that pings you). Add **Renovate** for third-party image bumps so you get discovery with a human gate.

**One note on repo layout.** A single `{node}/versions.conf` is nice for auditability but works against path-based change detection: every bump touches one file, so the diff can't tell you which service moved. Two ways out — put the digest inline in each component's own compose/quadlet file so path diffing just works, or keep `versions.conf` as the human-facing source and have CI render final files into a separate `deploy` branch that the node pulls. The second is the Argo/Flux pattern and earns its indirection once you have more than a couple of nodes.

### CD approach comparison

| Approach                   | Inbound port | Latency      | Feedback to pusher | Complexity  | Quadlet support        |
| -------------------------- | ------------ | ------------ | ------------------ | ----------- | ---------------------- |
| Timer + git pull           | none         | 1–5 min      | you build it       | very low    | yes (needs diff logic) |
| Webhook + git pull         | one, HMAC    | seconds      | weak               | low         | yes                    |
| `ansible-pull`             | none         | 1–5 min      | weak               | low-medium  | yes (handlers help)    |
| Portainer GitOps           | UI port      | poll or hook | in UI              | low         | no                     |
| Komodo                     | agent + Core | seconds      | good UI            | medium-high | no                     |
| CI → SSH (static key)      | 22           | seconds      | full CI logs       | low         | yes                    |
| CI → SSH over Tailscale    | none         | seconds      | full CI logs       | medium      | yes                    |
| CI → SSH cert via OIDC     | 22           | seconds      | full CI logs       | high        | yes                    |
| Self-hosted runner on node | none         | seconds      | full CI logs       | medium      | yes                    |
| `podman auto-update`       | none         | daily timer  | none               | very low    | native                 |

### Runtime mechanics that differ between Compose and Quadlet

- **Zero-downtime.** Compose has no native story; `up -d` stops the old container before starting the new one for the same service name. Options: the `docker rollout` CLI plugin (start new → health check → swap proxy → stop old), or two replicas behind Traefik/Caddy. Quadlet needs manual blue/green with templated units and a proxy swap. Honest read for a handful of services: a 1–3 second blip behind a retrying proxy is cheaper than the machinery. Reach for `docker rollout` when you find yourself caring.
- **Rollback.** Both reduce to `git revert` plus re-apply, which is only meaningful if you pinned digests. Two things make it real: don't prune images aggressively, so a revert doesn't depend on the registry still having the old digest; and script the auto-revert (below). Quadlet gets health-gated rollback for free, but only via the `podman auto-update` path.
- **Secrets.** Never plaintext in git. For Compose, **SOPS + age** is the sweet spot: encrypted files live in the repo, the age key sits in `/etc/sops/age/keys.txt` mode 600, and the reconcile script decrypts into tmpfs or a `docker secret`. For Quadlet, `podman secret` via `Secret=` works, but **`systemd-creds` with `LoadCredentialEncrypted=`** fits better — the blob is host-bound (optionally TPM-sealed), so it is safe to commit and only decryptable on that machine.
- **Rootless Quadlet.** Everything becomes `systemctl --user`, units live in `~/.config/containers/systemd/`, and `loginctl enable-linger` is mandatory or nothing starts until the user logs in.

## Handling a deployment that fails to start

Failure handling is where the pull model earns or loses its keep, because no human is watching a CI log. Five things to get right.

### 1. Fail before you touch anything running

Order the work so cheap checks run first and nothing stops until everything is ready:

```bash
docker compose config -q                 # syntax/interpolation
docker compose pull                      # missing digest, registry down, auth → abort here
# only now:
docker compose up -d --wait
```

The Quadlet equivalent is `/usr/lib/systemd/system-generators/podman-system-generator --dryrun` to validate unit syntax, plus an explicit `podman pull` of every digest before any `systemctl restart`. This alone converts a large class of outages into "the deploy didn't happen," which is a much better failure.

### 2. Detect properly — health checks are the linchpin

Without health checks, "started" means "the process hasn't exited yet," which catches almost nothing.

**Compose:** `up -d --wait --wait-timeout 120` exits non-zero if a container is unhealthy or exits non-zero — but only if a `healthcheck:` is actually defined. Set `--wait-timeout` above your slowest `start_period`, or slow starts will trigger rollbacks.

**Quadlet:** use `Notify=healthy` (Podman 5.0+, so fine on Debian 13's 5.4.2):

```ini
[Container]
Image=ghcr.io/org/app@sha256:...
HealthCmd=/app/healthcheck
HealthStartPeriod=10s
Notify=healthy

[Service]
TimeoutStartSec=120
```

The unit is not considered started until the container reports healthy, so `systemctl restart foo` blocks and exits non-zero on failure — the same semantics as Compose's `--wait`. Also add `Restart=on-failure` with `StartLimitBurst=3` so a crash-looping container lands in `failed` state instead of restarting forever unnoticed.

### 3. Revert — and quarantine the bad commit

This is the part that bites people. If the node reverts locally but `origin/main` still points at the bad commit, the next timer tick sees a mismatch and re-applies it: **a flapping deploy loop every 60 seconds**, which — because it keeps "recovering" — looks intermittent rather than broken.

So the node needs two pieces of persistent state: what is currently applied, and what is known bad.

```bash
#!/usr/bin/env bash
set -euo pipefail
exec 9>/var/lock/deploy.lock; flock -n 9 || exit 0   # no overlapping runs

REPO=/var/lib/deploy/repo; STATE=/var/lib/deploy/state; NODE=$(hostname -s)
cd "$REPO"; git fetch -q origin main
TARGET=$(git rev-parse origin/main)
APPLIED=$(cat "$STATE/applied" 2>/dev/null || true)

[[ "$TARGET" == "$APPLIED" ]] && { touch "$STATE/heartbeat"; exit 0; }
grep -qxF "$TARGET" "$STATE/quarantine" 2>/dev/null && exit 0

git checkout -q "$TARGET"
docker compose -f "$NODE/compose.yaml" config -q
docker compose -f "$NODE/compose.yaml" pull -q

if docker compose -f "$NODE/compose.yaml" up -d --wait --wait-timeout 120 --remove-orphans; then
  echo "$TARGET" > "$STATE/applied"; touch "$STATE/heartbeat"
else
  echo "$TARGET" >> "$STATE/quarantine"
  if [[ -n "$APPLIED" ]]; then
    git checkout -q "$APPLIED"
    docker compose -f "$NODE/compose.yaml" up -d --wait --wait-timeout 120 \
      || { echo "ROLLBACK FAILED" > "$STATE/fatal"; exit 2; }
  fi
  exit 1
fi
```

Three details worth noting:

- **Quarantine clears itself implicitly.** A _new_ commit has a different SHA, so pushing a fix resumes deploys with no manual intervention.
- **The working tree is left matching what is actually running**, so a reboot resurrects the good version rather than the bad one.
- **Exit code 2 (rollback itself failed) is a distinct, page-a-human state.** Do not retry it.

Rollback also depends on the old image still existing. Never run `docker image prune -a` from cron; keep the last few tags, and make sure your registry's retention policy won't delete digests you might revert to.

### 4. The cases revert doesn't fix

- **Database migrations.** If the new version migrated the schema, reverting the container leaves old code against a new schema. No deployment tool solves this — it is a discipline: every migration must be backward-compatible with the previous release (expand/contract, two-phase). At this scale you can also snapshot before migrating (`pg_dump` in a pre-deploy step), which works fine for small datasets and stops working somewhere in the tens of GB.
- **Partial failure.** Service A healthy, B unhealthy. Reverting the whole commit is the right default — it is the only combination you actually tested — even though it rolls back A's good change too. Per-service revert produces combinations that have never existed anywhere.
- **Healthy but wrong.** Passes the health check, serves errors. Automation cannot catch this; the mitigation is health checks that verify real dependencies (can it reach Postgres, can it read its config) plus a fast manual revert path. Make sure `git revert && push` is a complete, documented recovery action, because that is what someone will reach for at 3am.

### 5. Alerting — and specifically the deadman

With no CI log to watch, **absence of signal is the most dangerous state**. Alert on staleness, not just on errors.

```ini
# deploy.service
[Unit]
OnFailure=deploy-alert@%n.service
```

Plus a metric via node_exporter's textfile collector:

```
deploy_last_success_timestamp_seconds 1754...
deploy_quarantined_total 0
deploy_applied_info{sha="a1b2c3d"} 1
```

Then two alerts: `deploy_quarantined_total > 0` (a deploy failed and was rolled back), and `time() - deploy_last_success_timestamp_seconds > 900` (the loop itself is broken). The second one is the one that saves you.

If you want the pusher to learn the outcome without opening Grafana, the least-bad option is a fine-grained token scoped to _commit statuses: write_ on the single repo, used by the node to post a status on the deployed SHA. It reintroduces a credential, but a nearly powerless one; store it outside the repo (e.g. `systemd-creds` or mode-600 under `/etc`) and never in a committed file. Routing Alertmanager to Slack is simpler and usually enough.

**The cheapest improvement of all** is a staging target that applies the same commit first, with CI only writing the prod digest once staging goes green. That turns most of the above into a safety net you rarely exercise — which is what you want, since untested rollback paths tend not to work.

---

### Maintenance status scorecard (2026)

- **Actively developed:** Docker Compose, Podman/Quadlet, Kamal, Coolify, Dokploy, Komodo, Dokku, Incus, Caddy, Traefik, Diun, NixOS/Colmena, Renovate.
- **Stable but frozen / maintenance-only:** Docker Swarm (no active feature development; "peaked around 2019," running today "in most meaningful ways the same product it was in 2022"), podman-compose (thin, slow), NPM (spurty releases, security lag).
- **Slowed / at-risk:** CapRover (activity slowed, Swarm-bound).
- **Dead / archived:** Watchtower (archived Dec 17, 2025), Ptah.sh (archived 2024), NixOps (largely superseded by Colmena/deploy-rs).
- **Fine but heavyweight for this scale:** Nomad (BUSL, IBM-owned, no OSS fork), LXD (Canonical/Snap-centric).

## Recommendations

**Staged decision guide, mapped to what you optimize for:**

1. **Simplest possible thing that will still be here in five years → Docker Compose + Caddy.** Add Diun for update notifications. This is the lowest-overhead, best-documented, most future-proof baseline. Choose this unless you have a specific reason not to.

2. **You want OS-native, rootless, no daemon, tight systemd integration → Podman + Quadlet + Caddy/Traefik.** Use `podman auto-update` + timer. Best if you're on Fedora/RHEL/Debian 13+ and already comfortable with systemd. Enable linger for rootless services.

3. **You want a web UI / dashboard → Dokploy (cleaner UI, Docker-native) or Coolify (most features/templates, biggest community).** For pure container _management_ rather than a PaaS, use **Portainer CE** or **Dockge**. If you'll manage more than one host and want GitOps, use **Komodo**.

4. **You want declarative / GitOps-style config with the best rollback → NixOS + Colmena** (whole-host declarative, atomic rollback), or **Komodo** (TOML resource sync + Git webhooks) if you prefer containers with a GitOps layer. Ansible is the pragmatic middle ground if you want push-based config management without adopting Nix.

5. **You may grow to a few hosts → Kamal (SSH multi-server), Komodo (agent fleet), or Uncloud (mesh) — in that order of maturity.** Coolify/Dokploy can also scale via Swarm, with the caveat that Swarm is frozen. Do not architect on Swarm for a _new_ build expecting long-term investment.

6. **Maximum stability, minimal moving parts, non-web or stateful services → plain systemd units, systemd portable services, or Podman Quadlet.** No platform to maintain, no control plane, no daemon churn. This is also the right answer if "container-based" turns out to be optional for you.

7. **Git-push Heroku nostalgia on one box → Dokku.** Still the cleanest single-server git-push experience.

8. **Continuous deployment on one box → timer-based git pull, with a webhook poke for speed.** Concretely: digests pinned in git; CI builds, pushes, and commits its own digest; Renovate PRs for third-party images; SOPS+age (Compose) or `systemd-creds` (Quadlet) for secrets; **auto-revert on health-check failure with the bad SHA quarantined**; and an `OnFailure=` alert plus a staleness alert so a stalled loop can't hide. Add Tailscale later if you want a push path for interactive operations. Choose `podman auto-update` instead only if you accept the registry — not git — as the source of truth.

**Benchmarks that should change your choice:**

- If you cross **~25 containers and/or add a genuine second+ node needing cross-node scheduling**, revisit — that's the point where Komodo/Uncloud (or, reluctantly, K3s for learning) start earning their complexity.
- If you find yourself **hand-scripting zero-downtime swaps repeatedly**, adopt Kamal or a PaaS.
- If **update discipline is slipping**, move from manual pulls to Diun notifications (never blind auto-pull on databases).
- If **secrets sprawl into `.env` files**, stand up OpenBao.
- If the **reconcile script grows past ~150 lines** or you add a third node, move to `ansible-pull` (handlers, whole-host coverage) or Komodo rather than continuing to extend bash.
- If you find yourself **holding a static SSH key in CI**, replace it with an ephemeral Tailscale runner, an OIDC-issued short-lived SSH cert, or a self-hosted runner on the node.

## Caveats

- **Fast-moving space; verify at install time.** GitHub star counts, version numbers (Coolify v4/v5, Komodo v2, Nomad 2.0, Podman releases), and licensing tiers change quickly. Confirm current status before committing.
- **Several sources are vendor/advocacy blogs.** Much "moved off Kubernetes" writing and several comparison posts come from tools' own marketing sites (Haloy, Sliplane, deploy-tool vendors) or opinion pieces; directional sentiment is consistent with Hacker News/Lobsters, but treat specific claims as the authors' own rather than independent fact. The Coolify "seven CVEs" figure comes from a comparison blog citing an unnamed audit and is not independently confirmed here.
- **Docker Swarm's status is genuinely contested.** Mirantis markets continued maintenance and some bloggers insist it "still works," but the weight of evidence (including Portainer's own writeup and the official Docker roadmap barely mentioning it) is that it is frozen with no active feature investment. Treat it as legacy for _new_ builds.
- **Nomad licensing nuance:** BUSL source files auto-convert to an open license four years after publication, and internal single-host use is unaffected by BUSL's competitive-service restriction — so BUSL is more a governance/philosophy concern than a practical blocker for a homelab. The real reason to skip it here is fit, not license.
- **"Not production-ready" labels are the authors' own** — Uncloud explicitly warns of breaking changes; heed those before relying on newer entrants.
- **IBM/HashiCorp deal value figures differ by definition:** the ~$6.4B figure is the announced enterprise value; IBM's own 10-Q reports a total equity value of approximately $7.2 billion. Both refer to the same transaction, closed February 27, 2025.
- **The reconcile script above is an illustrative skeleton, not tested production code.** Treat it as a starting point; in particular, deliberately exercise the rollback path (deploy a knowingly broken image) before you rely on it, because untested rollback paths tend not to work.
- **`podman auto-update` and digest-pinning-in-git are mutually exclusive by design.** Auto-update requires a floating tag, so the registry becomes the source of truth. Pick one model per service and document which.
- **Self-hosted CI runners are an arbitrary-code-execution surface.** GitHub explicitly recommends against them on public repositories because fork pull requests can run code on them. Unprivileged user, narrow sudo, private repos only.
- **Migration reversibility is a discipline, not a feature.** Nothing in this document makes a schema change safe to roll back; expand/contract migrations are the actual mechanism, and pre-migration snapshots stop scaling in the tens of GB.
- **Health-check quality bounds everything.** Auto-revert can only catch what the health check detects; a container that starts and serves errors will be treated as a successful deploy under every option here.

---

# Container Runtime & Continuous-Deployment Decision Report

**Revision 2 — 2026-08-07.** Supersedes the initial report and its OpenBao/config-changes addendum; both are folded in here.

> **What changed since rev 1**
>
> 1. **OpenBao joins the bundle as a fifth service** and is the most availability-critical thing on the box — it sits on the hot read path (a transit call per part, per GET) _and_ gates boot. It gets its own cadence tier alongside Postgres, not Caddy's. Rev 1's secrets section is superseded (§2.6, §8.2).
> 2. **Config-file changes were a genuine gap.** Neither runtime notices that the _contents_ of a mounted file changed, so rev 1's reconcile logic would have silently ignored config-only commits. Fixed with a config-hash-in-unit pattern (§6).
> 3. **Piri and Ingot have no SIGHUP reload** (confirmed). Config edits therefore cost exactly what version bumps cost, which makes change batching a requirement and **inverts rev 1's webhook recommendation** (§5.3, §7.2).
> 4. **Rev 1's deadman alert threshold was wrong** — 15 minutes fires on every Piri deploy, given a drain of up to 60 minutes (§7.2).
> 5. The ranked top 3 combos are **unchanged**; the OpenBao findings strengthen the Quadlet case rather than altering it.

---

## TL;DR

- **Runtime: Podman + Quadlet.** Per-service systemd-unit granularity fits the mixed-cadence requirement (Piri+Ingot lock-step / Caddy independent / Postgres and OpenBao manual) better than Compose's project-level reconcile. It is native on bootc/FCOS where Docker requires rpm-ostree layering, rootless-capable on a partner-controlled box, and — newly decisive — it makes "never casually restart OpenBao" a structural property rather than a discipline. The OpenBao host-hardening checklist also turns out to be four systemd directives.
- **Top 3 combos, ranked:** **(1) Quadlet + git-pull systemd-timer reconciler, fed by a CI "commit-the-digest" hybrid**; **(2) Quadlet + `ansible-pull` on a timer**; **(3) Compose + Portainer CE GitOps polling** (only if you standardise on a mutable OS and want a UI). Avoid `podman auto-update` as the CD engine — it requires a floating tag and so contradicts digest-pinning-in-git.
- **One deploy mechanism covers images _and_ config.** Render a content hash of each service's config subtree into its unit file, so a config edit becomes a unit-file change and the existing digest-diff detection covers both. No second mechanism, no file watchers.
- **Reload where it exists, restart where it doesn't.** Caddy, Postgres and OpenBao can take config changes without a restart. Piri and Ingot cannot — for them a one-line config edit costs a `pdp-gate` wait plus a full restart, identical to a version bump.
- **Security shape:** CI commits digest bumps to an unprotected `deploy` branch using a short-lived **GitHub App installation token**; nodes pull **outbound-only** with a per-node read-only credential, or — better, given the trust boundary — ship config as an **OCI artifact** so the box needs only the registry credential it already has and no git client at all. **Critically, the deploy loop's own credentials must live outside OpenBao** or a sealed OpenBao becomes an unrecoverable brick.

---

## 1. What runs on the box, and how each service updates

Five services, four distinct update policies. Getting this table right is most of the design.

| Service      | Role                          | Pin                              | Discovery → merge                     | Image change         | Config change                                      |
| ------------ | ----------------------------- | -------------------------------- | ------------------------------------- | -------------------- | -------------------------------------------------- |
| **Piri**     | PDP proving, Filecoin storage | digest, **lock-step with Ingot** | CI commits both digests in one commit | `pdp-gate` → restart | `pdp-gate` → restart (no reload)                   |
| **Ingot**    | customer-facing S3 gateway    | digest, **lock-step with Piri**  | as above                              | restart              | restart (no reload)                                |
| **Caddy**    | TLS termination, owns :443    | digest via Renovate              | Renovate PR, **auto-merge**           | restart              | **`systemctl reload`**                             |
| **Postgres** | Ingot object/part metadata    | digest, human-gated              | Renovate PR, `automerge: false`       | **manual**           | reload + `pending_restart` alert                   |
| **OpenBao**  | regional secrets + Region KEK | digest, human-gated              | Renovate PR, `automerge: false`       | **manual**           | SIGHUP-reload where supported; else manual restart |

### 1.1 Why OpenBao is in the manual tier despite being "infra"

Per the regional key-management RFC, the local OpenBao runs on a unix socket with raft storage on the appliance disk, is sealed by a transit key held at a central OpenBao, and is the single home for the Region KEK, provider wallet key, TLS leaf keys, Postgres credentials, service identity PEMs and S3 credentials. Two properties make it unlike Caddy:

1. **It is on the hot read path.** The region wrap is a transit `encrypt`/`decrypt` call (`aes256-gcm96`, `derived=true`, context-bound to space + blob digest) per part, per GET. Ingot cannot serve a single read without it. OpenBao sealed or down is a **total regional read outage**, not degraded service. (Throughput is not the concern — the RFC measures ~0.1 ms per op and ~19,000 ops/s even constrained to two cores.)
2. **It gates boot.** The RFC records as verified behaviour that with central unreachable or the seal credential revoked, OpenBao 2.6.1 **refuses to start at all**. That is intentional (the Principle 1 "startup kill" lever), but it means central availability gates regional recovery.

Consequences that ripple through the rest of this report:

- **Restarting OpenBao is a read outage, not a blip.** The barrier key is in memory; a restart seals it until the unseal round-trip to central completes. Same treatment as a Postgres major upgrade: rare, announced, human-gated.
- **OS reboot windows acquire an external dependency.** The reboot-batching plan needs a second gate beyond `pdp-gate`: never batch reboots against a central-OpenBao maintenance window, and alert on _unseal failure_ distinctly from "OpenBao unhealthy."
- **Regional recovery time is longer than the 30–90 s FCOS reboot figure**, because the real sequence is boot → network → unseal round trip → OpenBao healthy → Ingot/Piri start.

### 1.2 Lock-step, concretely

A single `versions.conf` (or a rendered unit pair) bumped in **one commit** is what makes Piri+Ingot lock-step atomic by construction. Whatever the repo layout, the reconcile step must apply both from the same commit and never partially. Renovate can group the two into a single PR via `packageRules`, and `docker:pinDigests` handles digest pinning; a Postgres rule with `automerge: false` plus `dependencyDashboardApproval: true` gives the explicit human tick, while a Caddy rule sets `automerge: true`. This implements the whole table above declaratively.

---

## 2. Runtime decision: Docker Compose vs Podman + Quadlet

### 2.1 Operational simplicity

Quadlet has fewer long-running parts: no daemon, containers are ordinary systemd services driven by `systemctl`/`journalctl`. Compose's "one YAML, `docker compose up -d`" is arguably friendlier to newcomers, but Quadlet composes better with everything else systemd already does on this box — ordering, health-gating, `OnFailure=` alerting, and cgroup resource control, all of which this design now needs.

### 2.2 Robust upgrades: per-service isolation

Quadlet gives each service its own unit, so you restart exactly one and leave the rest untouched — directly serving "Postgres and OpenBao must never be auto-touched." Compose reconciles at _project_ level: per Docker's docs, `up` stops and recreates containers whose configuration or image changed, and while you can scope it (`docker compose up -d piri ingot`), the safe-by-default posture is weaker. There are documented cases (docker/compose #9357, #10259) of Compose recreating a container — including Postgres — that the operator did not intend to touch.

With OpenBao in the bundle this stops being a preference. The single most important operational rule on the box is now "never casually restart OpenBao," and under Quadlet that rule is enforceable by simply keeping the unit out of the auto-restart set.

### 2.3 Dependency ordering and health-gating

`Requires=`/`After=` plus `Notify=healthy` expresses "Ingot does not start until OpenBao reports unsealed and healthy," enforced by systemd, with a failed unseal surfacing as a **failed unit** that `OnFailure=` can alert on. Compose's `depends_on: condition: service_healthy` handles start ordering but ties it to project-level reconcile and gives no equivalent of a unit landing in `failed` state for the alerting path.

- **Quadlet:** `Notify=healthy` arrived in Podman **5.0.0** — the release notes describe it as sdnotify-ing that a container has started only once its health check begins passing. `.container` units default to `Type=notify`, so `systemctl restart` blocks and exits non-zero on failure. **Known bug (containers/podman #27290, open as of late 2025):** systemd sees the service as _starting_ until the startup health check succeeds, and if that exceeds systemd's own start timeout it stops the service and marks it failed. The fix per the issue is to set `TimeoutStartSec=` — to `infinity`, or to `HealthStartPeriod` plus margin. **Given Piri's up-to-60-minute graceful shutdown, set `TimeoutStartSec=` and `TimeoutStopSec=` explicitly.**
- **Compose:** `up -d --wait --wait-timeout <n>` blocks on health and returns non-zero — a good equivalent, but one project-wide gate rather than per-unit.

**Health-check quality now has a specific failure mode to defend against.** Ingot can start cleanly and 500 every GET because it cannot reach or authenticate to OpenBao. Ingot's health check must therefore exercise a real transit unwrap against a known context, not a TCP or `/healthz` liveness ping. Without that, health-gated deploys give false confidence on the failure most likely to occur.

### 2.4 Change detection and restart semantics (the mechanic that shapes the CD design)

**Editing a `.container` file and running `systemctl daemon-reload` regenerates the unit but does NOT restart the running container.** Per podman-systemd.unit(5), Quadlet files are read at boot and on `daemon-reload` and generate corresponding service units; systemd does not restart already-running services on reload. You must explicitly `systemctl restart <name>.service`.

For this design that is a _feature_: the reconcile script decides what restarts, so Postgres and OpenBao are excluded trivially. (Beware the common claim that Quadlets are "automatically updated on daemon-reload" — that refers to regenerating unit files, not restarting containers.)

Validation: `/usr/libexec/podman/quadlet -dryrun` (rootless: add `-user`), or `systemd-analyze --generators=true verify <unit>`. Podman **5.6.0** also added a `podman quadlet` command suite (`install`, `list`, `print`, `rm`) for lifecycle management — not a dry-run replacement, and not available with the remote client.

### 2.5 Host hardening maps onto unit directives

The key-management RFC's host checklist — following OpenBao's own post-mlock guidance after it removed the inherited mlock implementation as too subtle to get right (openbao #354, #363) — is not documentation; it is systemd directives:

| RFC requirement                                          | Directive                       |
| -------------------------------------------------------- | ------------------------------- |
| swap disabled (`memory.swap.max=0` in the unit's cgroup) | `MemorySwapMax=0`               |
| core dumps off                                           | `LimitCORE=0`                   |
| seeded entropy at first boot                             | Ignition/cloud-init, host image |
| no snapshot/clone reuse of a running appliance           | operational policy              |

Apply the first two to **both OpenBao and Ingot** — the RFC notes one checklist protects both processes. Per-service cgroup and rlimit control in the same reviewable file as the image pin is a real, if modest, Quadlet advantage. Compose's `mem_swappiness`/`ulimits` cover some of this but map less directly onto the RFC's cgroup-level language.

### 2.6 Secrets

**OpenBao owns application secrets.** This supersedes rev 1's SOPS-vs-`systemd-creds` comparison and simplifies the design in one important way: because configs in git now hold _references_ to OpenBao paths rather than values, **the config files are safe to commit in plaintext and SOPS + age largely disappears.**

What remains host-bound is a small, deliberate set — see §8.2 for why this set must not live in OpenBao:

- git read credential for the `deploy` branch (or the registry credential, on the OCI-artifact path)
- registry pull credential
- the OpenBao unseal/boot credential itself

`systemd-creds` / `LoadCredentialEncrypted=` is the right mechanism for exactly these, and the RFC's open item to "TPM-seal the boot credential" when the host-image work lands applies to precisely this set.

**Wiring the bootstrap credential into a container has a real rough edge.** Quadlet passes `[Service]` through untouched so `LoadCredentialEncrypted=` works at unit level, but `$CREDENTIALS_DIRECTORY` is a host path that Quadlet does not project into the container. The stock-pieces solution is a separate oneshot that materialises it as a `podman secret`:

```ini
# filone-bao-bootstrap.service  (plain systemd unit, not a Quadlet file)
[Unit]
Description=Materialise OpenBao seal credential as a podman secret
Before=openbao.service
[Service]
Type=oneshot
RemainAfterExit=yes
LoadCredentialEncrypted=bao-seal:/etc/filone/bao-seal.cred
ExecStart=/usr/local/bin/filone-bao-bootstrap   # reads $CREDENTIALS_DIRECTORY/bao-seal → podman secret
LimitCORE=0
```

The credential never enters git, a log, or unencrypted disk. **Verify the exact transit-seal token config key / env var against the OpenBao transit seal docs** — a placeholder is used below rather than a guess.

### 2.7 Immutable vs mutable OS — often the deciding factor

- **On bootc / FCOS (the current lean): Quadlet is native, Docker is not.** These images ship Podman and no Docker Engine; Docker requires rpm-ostree layering, a reboot to apply, and ongoing maintenance against a read-only `/usr`. This materially penalises Compose.
- **Unit files** live in `/etc/containers/systemd/` (writable) or baked into `/usr/share/containers/systemd/`. **All persistent state must be under `/var`** — mount the operator volume at `/var/mnt/filone` and point Postgres, OpenBao raft storage, Piri's SQLite and Caddy's cert store at subdirectories.
- **bootc bound images** (symlink a `.image`/`.container` into `/usr/lib/bootc/bound-images.d`, images referenced by digest or immutable tag, accessed via an additional image store) pre-pull app images at OS-upgrade time. Quadlet-only, no Compose equivalent — but it ties app cadence to OS cadence, so it suits a pinned baseline rather than the independent-Caddy / lock-step-Piri policy.
- **Debian 13 "trixie" ships Podman 5.4.2** — new enough for `Notify=healthy` (needs ≥5.0), too old for `ReloadCmd=` and `podman quadlet` (5.6.0). Both gaps have clean workarounds (§6.3).
- **Ubuntu LTS is the problem case: 24.04 ships Podman 4.9.3**, 22.04 the 3.4.x series — too old for `Notify=healthy`. On Ubuntu you need a backport or the OpenSUSE build repo. Compose is trivially available there via Docker's apt repo.

### 2.8 Runtime verdict

**Podman + Quadlet.** It wins on per-service isolation, dependency ordering and failure surfacing, host-hardening expressiveness, rootless operation on partner hardware, and the immutable-OS path. Compose's edges are newcomer familiarity and the `up -d --wait` one-liner — real but outweighed. The one budget item: if the OS decision lands on **Ubuntu LTS specifically**, plan for getting a modern Podman onto the box.

---

## 3. CD mechanism catalogue

### Pull-based

**A. systemd timer + `git fetch` + reconcile script.** _(Both runtimes.)_ A `.timer` fires a oneshot that fetches the deploy ref and, if it moved, reconciles. Compose: `up -d --wait` is itself the reconciler. Quadlet: copy changed units, `daemon-reload`, then selectively restart (§2.4). Most transparent option, slots `pdp-gate` in naturally, latency = poll interval. Git-as-truth: perfect. No conflict with git-revert.

**B. `ansible-pull` on a timer.** _(Both.)_ Handlers give idempotent "restart only what changed"; excellent fit for the manual tier (simply no restart handler). Costs Ansible on every box — awkward on immutable `/usr`, so install layered or run from a container. **Watch:** `ansible.builtin.systemd_service` daemon-reload is not idempotent (issue #86495 — a reload should always be assumed to have changed something), so gate restarts on file-change facts, never on the reload task.

**C. `podman auto-update` + timer.** _(Quadlet-native.)_ **Confirmed still incompatible with digest-pinning-in-git.** The podman-auto-update man page is explicit: `AutoUpdate=registry` compares the local digest against the remote image _for a tag_, and a digest-pinned `Image=...@sha256:...` will never trigger an update. So the registry, not git, becomes the source of truth and the git-pinned digest is decorative. `AutoUpdate=local` only helps when another process builds/pulls on the host. Its automatic rollback (re-tag to previous on restart failure) also actively fights the git-revert model. **Not the CD engine.** Cosign verification can be layered independently via `policy.json` `sigstoreSigned` + `use-sigstore-attachments`.

**D. Flux / Argo CD.** _(N/A.)_ Kubernetes controllers with no supported non-K8s mode. Steal the rendered-manifests/deploy-branch _pattern_, not the tools.

**E. Komodo.** _(Compose-only.)_ v2.0.0 (March 2026) moved to auto-generated key-pair PKI auth with rotation and — importantly for this trust model — added outbound Periphery, so the agent can initiate the connection to Core (default inbound port 8120 otherwise). Deploys Compose stacks from git with auto-deploy on push; uses Docker as the engine and **does not manage Quadlet.** Good dashboard.

**F. Portainer CE GitOps.** _(Compose-only.)_ Backs a stack with a git repo, polling (outbound-only) or webhook, comparing deployed commit SHA against the configured ref. **Verify CE vs BE:** "force redeployment / git-wins" is documented as Business-Edition-only in at least the 2.27 LTS docs.

**G. Harbormaster.** _(Compose-only.)_ Minimal git→Compose syncer. Niche; verify maintenance status.

**H. bootc bound images.** _(Quadlet-only.)_ See §2.7 — baseline pinning, not per-service cadence.

**I. Diun.** _(Both, notify-only.)_ Useful complement for watching Caddy/Postgres/OpenBao upstreams.

**J. Renovate.** _(Both.)_ The discovery layer. Native `docker-compose` manager and a **native `quadlet` manager** for `.container`/`.image`/`.volume` files (updates where an `Image` option is present), so no custom regex is needed for the basic case; a `customManagers` regex remains available for nonstandard layouts. Handles the grouping/auto-merge policy in §1.2.

### Push-based

**K. Actions → SSH, long-lived key in a CI secret.** Requires an inbound port and gives GitHub a forever-credential that reaches every box. Harden with a forced command, no-shell deploy user, narrow sudoers — but **not recommended** here.

**L. Actions → SSH over an ephemeral Tailscale node.** `tailscale/github-action` with an **OAuth client** (Tailscale recommends this over auth keys for automatic cleanup and better isolation), tagged ephemeral node, ACLs restricting `tag:ci` → the appliance tag, Tailscale SSH instead of static keys; federated-identity variant needs `id-token: write`. **No inbound port.** The cleanest push option for this trust model. Residual: the OAuth client secret lives in GitHub and the box joins a central tailnet. Headscale and Cloudflare Tunnel are alternatives.

**M. Actions → short-lived SSH cert via OIDC** from OpenBao/step-ca. No static secret in CI at all; most secure, most moving parts (you run a CA). Combine with L for outbound-only.

**N. Self-hosted runner on the node.** Outbound-only and no inbound port, but an arbitrary-code-execution surface and a trust inversion — your CI identity executing on the partner's box. GitHub's own docs warn that self-hosted runners should almost never be used with public repositories because fork PRs can compromise the runner environment including secrets and `GITHUB_TOKEN`, and that even on private repos anyone who can fork and open a PR is a risk. Private repos, `--ephemeral`, outside-collaborator approval, runner groups — or skip.

**O. Webhook receiver on the node (HMAC-verified) → triggers the pull.** Push the notification, pull the content. **Rev 2 caveat: debounce it.** See §5.3 — firing per push is now a liability, not a speed win. Needs an inbound port (tunnel it).

**P. Ansible push over SSH from Actions.** Same inbound concerns as K/L; prefer `ansible-pull`.

### The hybrid spine: CI commits the digest, the node pulls

CI builds, resolves digests, and commits the bump into git; nodes pull. Repo-layout options:

- **`versions.conf` per fleet/node** — simplest lock-step story, trivial path-based change detection, Postgres/OpenBao lines edited only by human PR. **Recommended.**
- **Digests inline per unit file** — idiomatic for Renovate's native managers; lock-step then spans two files (Renovate grouping handles the PR, the reconciler must still apply atomically).
- **Rendered `deploy` branch** (the Argo/Flux pattern) — `main` keeps readable sources under full branch protection; CI renders digests **and config hashes** into an unprotected `deploy` branch that nodes read. Slightly more machinery, and §6.2 gives it a second independent justification, which promotes it from optional to preferred.

---

## 4. Comparison matrices

### Runtime × mechanism applicability

| Mechanism                       | Compose | Quadlet                   |
| ------------------------------- | ------- | ------------------------- |
| A. Timer + git pull + reconcile | ✅      | ✅                        |
| B. ansible-pull                 | ✅      | ✅                        |
| C. podman auto-update           | ❌      | ✅ but breaks git-pinning |
| D. Flux/Argo                    | ❌      | ❌                        |
| E. Komodo                       | ✅      | ❌                        |
| F. Portainer GitOps             | ✅      | ❌                        |
| G. Harbormaster                 | ✅      | ❌                        |
| I. Diun / J. Renovate           | ✅      | ✅                        |
| K–P. push family                | ✅      | ✅                        |

### Scoring the realistic candidates

| Mechanism (runtime)                     | Git = truth    | Mixed cadence     | Config changes         | Merge→running                 | GH privilege          | Inbound ports        | Visibility                                      | Parts    | Fights git-revert? |
| --------------------------------------- | -------------- | ----------------- | ---------------------- | ----------------------------- | --------------------- | -------------------- | ----------------------------------------------- | -------- | ------------------ |
| **A. Timer pull + CI digest (Quadlet)** | ✅ full        | ✅ excellent      | ✅ via §6.2 hash       | poll interval, **coalescing** | commit-only App token | none                 | `OnFailure=` + textfile metrics + journald→Loki | low      | No                 |
| **B. ansible-pull (Quadlet)**           | ✅ full        | ✅ excellent      | ✅ handlers native     | poll interval                 | same                  | none                 | Ansible recap + `OnFailure=`                    | medium   | No                 |
| **F. Portainer GitOps (Compose)**       | ✅ SHA compare | ⚠️ project-level  | ⚠️ needs hash in label | poll interval                 | node read token       | none (poll)          | UI; weak alerting                               | medium   | No                 |
| **E. Komodo v2 (Compose)**              | ✅             | ⚠️ project-level  | ⚠️ as above            | webhook, per-push             | webhook               | none (outbound mode) | good UI                                         | med-high | No                 |
| **L. Tailscale SSH push**               | ⚠️ push        | ✅ script decides | ✅                     | seconds, **no coalescing**    | OAuth client          | none                 | Actions UI                                      | medium   | Neutral            |
| **C. podman auto-update**               | ❌ registry    | ⚠️ tag-driven     | ❌ ignores config      | timer                         | none                  | none                 | journald only                                   | low      | **Yes**            |
| **N. Self-hosted runner**               | ⚠️             | ✅                | ✅                     | seconds                       | **high (ACE)**        | none                 | Actions UI                                      | medium   | Neutral            |

---

## 5. Applying changes: images, config, and the cost of a restart

### 5.1 The gap: mounted config is invisible to both runtimes

**Neither runtime notices that the contents of a mounted config file changed.** Symmetrically:

- **Quadlet:** the `.container` file is unchanged, so `daemon-reload` regenerates an identical unit and nothing restarts. The new config sits unread until an unrelated restart — the worst case, because it then applies at a random later time nobody connects to the commit.
- **Compose:** `up -d` compares the _service definition_, not the contents of bind-mounted files. An edited Caddyfile with an unchanged `compose.yaml` yields "up-to-date."

### 5.2 The fix: hash the config into the unit file

Adapt the Kubernetes config-checksum-annotation convention. Render a content hash of each service's config subtree into that service's unit, so **a config change becomes a unit-file change** and one detection mechanism covers images and config:

```ini
# units/caddy.container  (rendered by CI)
[Container]
Image=docker.io/library/caddy@sha256:PINNED_DIGEST
Annotation=filone.config-hash=3f9a1c7e2b8d4056
Volume=/var/mnt/filone/config/caddy:/etc/caddy:ro,z
```

Use `Annotation=` rather than `Environment=` so you are not injecting values into the application's environment where they could change behaviour. The hash covers only that service's subtree, so a Caddyfile edit does not restart Ingot.

**Have CI render the hash on merge into the `deploy` branch** — humans edit readable config on protected `main`, CI renders digests and hashes into `deploy`, nodes read only `deploy`. This is the second independent reason to adopt the rendered-branch layout. A local-hash alternative works (the reconciler computes hashes and keeps per-service state) but puts more state and logic on 30+ boxes.

Pair it with an **apply-policy manifest in git**, keeping the reconciler generic and the policy reviewable:

```yaml
# services.yaml
services:
  openbao: { apply: reload, restart: manual, validate: bao-validate }
  postgres: { apply: reload, restart: manual }
  caddy: { apply: reload, validate: caddy-validate }
  ingot: { apply: restart }
  piri: { apply: restart, pre: pdp-gate }
```

### 5.3 Apply verbs, and why Piri/Ingot dominate the cost

**Confirmed 2026-08: Piri and Ingot do not support SIGHUP config reload.** The reload path therefore covers exactly the three services whose config rarely changes, while the two that change most often are restart-only.

| Service      | Config change                                                                                                           | Image change   |
| ------------ | ----------------------------------------------------------------------------------------------------------------------- | -------------- |
| **Caddy**    | `systemctl reload` → `caddy reload`, **zero-downtime**, config validated as part of the load                            | restart        |
| **Postgres** | `pg_ctl reload` (SIGHUP) — always safe; params needing restart simply don't take effect                                 | **manual**     |
| **OpenBao**  | SIGHUP reloads listener TLS, log level, declarative audit devices. **Seal and storage stanzas need a restart** → manual | **manual**     |
| **Piri**     | **restart, `pdp-gate`d, up to 60 min drain**                                                                            | identical cost |
| **Ingot**    | **restart, drops in-flight requests** (no graceful shutdown)                                                            | identical cost |

**For Piri and Ingot, a one-line config edit costs exactly what a version bump costs.** That simplifies their apply logic (`apply: restart`, unconditional) but makes change batching an operational requirement:

**Coalesce; do not deploy per-commit.** Deploying a config edit and an image bump separately pays the restart cost twice. The reconciler in §9.1 diffs `APPLIED..TARGET`, so N commits landing between two ticks collapse into **one** restart at the latest state — preserve that property in any refactor.

**This inverts rev 1's webhook recommendation.** Rev 1 treated the webhook (mechanism O) as a straight upgrade for cutting latency to seconds. For Piri/Ingot that is a liability: three commits merged in ten minutes become three `pdp-gate` waits and three drains. **Polling's coalescing window is a feature.** Keep the poll loop; if you add a webhook, debounce it rather than firing per push.

**The Postgres nuance turns a judgement call into an automatable one.** `pg_ctl reload` is unconditionally safe — parameters whose `context` is `postmaster` simply don't take effect, and Postgres sets `pending_restart`. So always reload, then alert on the leftovers:

```sql
SELECT count(*) FROM pg_settings WHERE pending_restart;
```

No allowlist of reloadable parameters to maintain, and nothing silently un-applied.

### 5.4 Quadlet's native reload support

`ReloadCmd=` / `ReloadSignal=` generate `ExecReload=` in the service unit, with Podman auto-prepending `podman exec <container>`:

```ini
# units/caddy.container
[Container]
ReloadCmd=caddy reload --config /etc/caddy/Caddyfile
```

`systemctl reload caddy` then performs a zero-downtime Caddy config reload. **Both keys arrived in Podman 5.6.0 and are mutually exclusive** (setting both is a generation error). Debian 13's 5.4.2 predates them, but the hand-written form works because Quadlet passes `[Service]` through untouched:

```ini
[Service]
ExecReload=/usr/bin/podman exec caddy caddy reload --config /etc/caddy/Caddyfile
```

### 5.5 The bind-mount inode footgun

**Never bind-mount a single config file from a git working tree.** A file bind-mount resolves to an inode; `git checkout` replaces the file, so the container reads the _old_ file forever — and `systemctl reload` dutifully reloads stale config while reporting success. Silent, confusing, and it survives reloads.

```ini
Volume=/var/mnt/filone/config/caddy/Caddyfile:/etc/caddy/Caddyfile:ro   # WRONG
Volume=/var/mnt/filone/config/caddy:/etc/caddy:ro,z                     # correct — mount the directory
```

Use `:z` (shared relabel) rather than `:Z` (private) for any config directory read by more than one container. Corollary: don't mount the git worktree; sync rendered config into a stable directory the containers mount, keeping the checkout as staging.

### 5.6 Validate before touching anything running

Order the work so cheap checks run first and nothing stops until everything is ready. Run these against _staged_ config, before syncing into the live mount path:

```bash
/usr/libexec/podman/quadlet -dryrun >/dev/null            # unit syntax
podman exec caddy caddy validate --config /etc/caddy/Caddyfile
podman exec openbao bao operator diagnose                 # verify availability/flags
# then pre-pull every digest; only then apply
```

This converts a large class of outages into "the deploy didn't happen," which is a much better failure than a half-applied one.

### 5.7 Secret-bearing config: hash the template, never the render

If any config is rendered from OpenBao (via `openbao-template` or an agent), keep two things strictly separate:

- **the template** — in git, hashed into the unit, changes only when a human changes it
- **the rendered output** — never in git, **never hashed**, on tmpfs

Hashing rendered output creates a restart loop on every credential rotation: new lease → render changes → hash changes → restart → repeat on the rotation period. With Postgres credentials rotating daily that is a daily unexplained Ingot restart and `pdp-gate`d Piri restarts fighting the proving schedule. Let the renderer own restart-on-secret-change (`openbao-template` has an exec/supervisor mode with a configurable `reload_signal`, default SIGHUP) and the git loop own restart-on-config-change. Disjoint triggers, no overlap.

### 5.8 Why not systemd path units

`.path` units with `PathChanged=` look obvious and are the wrong tool, for three documented reasons:

1. **They start, not restart.** `PathChanged=` activates the configured unit, so an already-running service needs a separate restarter helper.
2. **They don't see atomic replacement.** Neither `PathChanged=` nor `PathModified=` detects atomic symlink replacement (systemd #17727) — exactly the shape of a git-driven atomic deploy. (Hidden files are ignored too.)
3. **No gating.** A path unit fires whenever the file closes — a Piri restart at an arbitrary moment, outside `pdp-gate`, possibly mid-proof. The reconcile loop must be the only thing that restarts Piri.

---

## 6. The Caddy TLS-key gap — flag it, don't build it

The key-management RFC lists TLS leaf keys among the secrets moving into OpenBao. **Caddy does not natively store certificates in Vault/OpenBao** — CertMagic defaults to filesystem storage, and the Vault path needs a third-party module (`caddy.storage.vault`) plus a custom `xcaddy` build, using KV-v2 and AppRole. That module is lightly maintained.

**Don't spend a custom Caddy build on it.** Better:

1. **LUKS + Clevis/Tang volume binding**, which the RFC itself lists as complementary — it extends dead-disk coverage to Caddy's cert store, Postgres's data directory and the spool in one mechanism, no custom builds, no plugin risk.
2. **Accept certs on a bound volume** and state it in the threat model.

The value case is weak on the RFC's own reasoning: it establishes that a hostile partner can obtain a valid certificate for their own hostname by intercepting traffic to their own IP, without touching the VM at all. A leaf key in OpenBao does not close that route. The Region KEK warrants real machinery; the leaf key does not.

---

## 7. Observability & alerting

- **Deploy lifecycle:** wrap the reconciler in a systemd oneshot with `OnFailure=deploy-alert@%n.service`; write node_exporter **textfile collector** metrics on each run (`filone_deploy_last_success_timestamp`, `filone_deploy_last_status`, `filone_deploy_active_sha`).
- **The deadman — and the rev 1 correction.** Rev 1 recommended alerting on `time() - last_success > 900`. **With Piri's drain running up to 60 minutes, that fires on every Piri deploy** — the fastest way to train the team to ignore an alert. Options, best first:
  - **Heartbeat during the apply, not just after it.** Touch the heartbeat before each restart and keep a background `while sleep 60; do touch $STATE/heartbeat; done` for its duration, so "the loop is alive" and "the loop last succeeded" are separate signals. Alert tightly on the heartbeat; track last-success without paging.
  - **Emit an explicit in-progress state** (`filone_deploy_in_progress{service="piri"} 1`) and suppress the staleness alert while set, with a second alert for in-progress held beyond worst-case drain plus margin — which catches a genuinely wedged restart.
  - Raise the threshold above worst-case drain (simplest, blinds you for an hour).

  Also set `TimeoutStartSec=` on the reconcile unit **above** Piri's `TimeoutStopSec`, or systemd kills the reconciler mid-drain. Keep the `flock` so overlapping ticks exit cleanly.

- **Drift on the manual tier.** A manual-tier change must be visible, or "Postgres is manual" degrades into "Postgres is forgotten." Emit `filone_deploy_pending_manual{service="postgres"} 1` and alert if held more than N days.
- **Unseal failure as its own alert**, distinct from "OpenBao unhealthy" (§1.1).
- **Service health:** node_exporter `--collector.systemd` exposes `node_systemd_unit_state`; alert on any of the five landing in `failed`.
- **Logs:** journald → Loki via Alloy. Reconciler stdout and podman events land in the journal automatically — a real Quadlet advantage, no separate plumbing.
- **Pusher feedback:** to reflect status into the GitHub commit/PR UI, post to the commit status API with a token scoped to `statuses: write` (a GitHub App with the Commit statuses permission is least-privilege). Push mechanisms give this free in the Actions run UI.

---

## 8. Security: least-privilege specifics

### 8.1 CI committing a digest bump

Ranked by blast radius, best first:

1. **GitHub App installation token** — short-lived (1 h), repo-scoped, `contents: write` only, decoupled from any human, revocable. The App private key is the crown jewel; store it well and never on the box. **Recommended.**
2. **Fine-grained PAT** — repo-scoped, expiring, tied to a user account.
3. **Deploy key (write)** — per-repo, no expiry.
4. **`GITHUB_TOKEN` with `contents: write`** — simplest, but branch protection and required reviews block a bot pushing to `main`.

**Branch protection interaction:** rather than granting a bypass, have CI **push to an unprotected `deploy` branch** that is the only thing nodes read, keeping `main` fully reviewed. This is the same layout §5.2 wants for config hashes — one decision serving two purposes. (A PR with auto-merge is the alternative.)

### 8.2 The circular dependency you must design out

**Do not put the deploy loop's own credentials in OpenBao.** If the git read credential or registry pull secret lives there, then a sealed or broken OpenBao means you cannot pull the image that fixes OpenBao — a region that can brick itself with one bad commit and no remote recovery path. Keep the §2.6 bootstrap set host-bound in `systemd-creds`.

### 8.3 Node-side read credential across 30+ semi-trusted boxes

Best first:

- **OCI artifact (ORAS / bootc bound images) instead of git-on-the-node.** The box already needs a registry credential for images; shipping compose/quadlet config as an OCI artifact removes the git client and git credential entirely. **Lowest-privilege and the best fit for a partner-operated box** — worth prototyping. Scope with a per-node registry robot account.
- **Per-node read-only deploy keys or fine-grained tokens** — revocable per box.
- **A single shared read-only deploy key across 30 partner boxes — avoid.** One compromised partner leaks read access for the whole fleet, and rotation means touching every box.
- **A central artifact mirror** the nodes pull from — reduces GitHub exposure, adds an audit choke point.

### 8.4 Push transport

Ephemeral Tailscale node or short-lived OIDC SSH certs beat a long-lived SSH key in a CI secret; both keep the box free of inbound ports. Self-hosted runners on partner boxes invert trust and are an ACE surface (§3, mechanism N).

---

## 9. The 3 recommended combos, ranked

### 🥇 #1 — Quadlet + git-pull systemd-timer reconciler, fed by a CI "commit-the-digest" hybrid

**Concretely.** A private repo holds unit files and config on `main`. On merge, GitHub Actions builds/resolves Piri+Ingot digests and renders units — digests **and config hashes** — into an unprotected `deploy` branch using a GitHub App installation token. Each node runs a timer that fetches `deploy` outbound (read-only deploy key, or the OCI-artifact path), and on change: validates, pre-pulls, syncs config to a stable path, `daemon-reload`s, then applies per-service verbs from `services.yaml` — reload for Caddy/Postgres/OpenBao, `pdp-gate`+restart for Piri/Ingot, and _nothing_ for the manual tier beyond a drift metric.

**Why #1.** Maximum operational simplicity (git + systemd + one script; no daemon, no controller, no UI to run); best mixed-cadence story since per-unit restart is native; outbound-only; git as single source of truth with no revert conflict; strongest failure detection via systemd; and it handles config changes through the same path as image bumps. Best fit for the bootc lean.

**Cost.** A ~120-line reconcile script and the observability wiring. No dashboard — visibility is journald/Grafana.

**Failure modes to watch.** (1) `Notify=healthy` + `TimeoutStartSec` (#27290) — set timeouts explicitly given Piri's 60-minute drain. (2) The deadman threshold vs that same drain (§7). (3) The bind-mount inode trap (§5.5). (4) Deploy-key sprawl across 30 partner boxes — per-node keys or OCI artifacts. (5) A sealed OpenBao blocking its own fix (§8.2).

#### Reconcile script

```bash
#!/usr/bin/env bash
set -euo pipefail
exec 9>/var/lock/filone-reconcile.lock; flock -n 9 || exit 0

REPO=/var/lib/filone/config; STATE=/var/lib/filone/state
cd "$REPO"
APPLIED=$(cat "$STATE/applied" 2>/dev/null || echo "")
git fetch --quiet origin deploy
TARGET=$(git rev-parse origin/deploy)
[ "$APPLIED" = "$TARGET" ] && { touch "$STATE/heartbeat"; exit 0; }

git checkout --quiet --force "$TARGET"           # staging only; nothing live yet

# 1. Validate, then pre-pull. Failure here = "the deploy didn't happen".
/usr/local/bin/filone-validate
awk -F= '/^Image=/{print $2}' units/*.container | xargs -rn1 podman pull --quiet

# 2. Which services changed? Unit files encode config hashes, so this single
#    diff covers image bumps AND config edits. Multiple commits coalesce here.
CHANGED=$(git diff --name-only "${APPLIED:-$TARGET}" "$TARGET" -- units/ \
          | xargs -rn1 basename | sed 's/\.container$//' | sort -u)

# 3. Apply: config to a stable path (never mount the worktree), then units.
rsync -a --delete config/ /var/mnt/filone/config/
install -m0644 units/*.container /etc/containers/systemd/
systemctl daemon-reload

for svc in $CHANGED; do
  policy=$(yq -r ".services.\"$svc\".restart // \"auto\"" services.yaml)
  verb=$(yq   -r ".services.\"$svc\".apply   // \"restart\"" services.yaml)
  pre=$(yq    -r ".services.\"$svc\".pre     // \"\""       services.yaml)

  if [ "$policy" = "manual" ]; then
    echo "$svc" >> "$STATE/pending-manual"       # git moved; the box deliberately did not
    logger -t filone-deploy -p warning "$svc changed in git; manual apply required"
    continue
  fi
  [ -n "$pre" ] && "/usr/local/bin/$pre" wait    # pdp-gate for piri
  systemctl "$verb" "$svc.service"               # reload | restart
done

echo "$TARGET" > "$STATE/applied"; touch "$STATE/heartbeat"
/usr/local/bin/filone-write-metrics
```

#### Representative units

```ini
# /etc/containers/systemd/openbao.container
[Unit]
Description=OpenBao (regional secrets, Region KEK)
Requires=filone-bao-bootstrap.service var-mnt-filone.mount
After=filone-bao-bootstrap.service var-mnt-filone.mount network-online.target
[Container]
Image=ghcr.io/openbao/openbao@sha256:PINNED_DIGEST
Annotation=filone.config-hash=CONFIG_HASH_PLACEHOLDER
Secret=bao-seal,type=env,target=SEAL_TRANSIT_TOKEN_PLACEHOLDER
Volume=/var/mnt/filone/openbao:/openbao/data:Z
Volume=/var/mnt/filone/config/openbao:/openbao/config:ro,z
Volume=/run/filone:/run/filone:z          # unix socket, shared with ingot/piri
Notify=healthy
HealthCmd=bao status -address unix:///run/filone/bao.sock
HealthStartPeriod=30s
[Service]
MemorySwapMax=0        # RFC host hardening: no swap for this cgroup
LimitCORE=0            # RFC host hardening: no core dumps
TimeoutStartSec=180    # unseal round-trip to central
Restart=on-failure
StartLimitBurst=3
```

```ini
# /etc/containers/systemd/piri.container
[Unit]
Description=Piri PDP node
Requires=openbao.service
After=openbao.service network-online.target
[Container]
Image=ghcr.io/filoz/piri@sha256:PINNED_DIGEST
Annotation=filone.config-hash=CONFIG_HASH_PLACEHOLDER
Volume=/var/mnt/filone/config/piri:/etc/piri:ro,z
Volume=/var/mnt/filone/piri:/var/lib/piri:Z
Volume=/run/filone:/run/filone:z
Notify=healthy
HealthCmd=piri status
HealthInterval=30s
HealthStartPeriod=60s
[Service]
TimeoutStartSec=infinity   # required per #27290
TimeoutStopSec=3900        # graceful shutdown up to 60 min
Restart=on-failure
StartLimitBurst=3
ExecStartPre=/usr/local/bin/pdp-gate wait
[Install]
WantedBy=multi-user.target
```

Caddy stays a **separate** unit owning :443, so restarting Piri or Ingot never closes the public listening socket.

#### CI workflow shape

```yaml
permissions:
  contents: write
  id-token: write
steps:
  - uses: actions/create-github-app-token@v2
    id: app-token
    with: { app-id: ${{ vars.APP_ID }}, private-key: ${{ secrets.APP_KEY }} }
  - run: |
      # resolve digests, compute per-service config hashes, render units
      # commit to the unprotected deploy branch
      git push https://x-access-token:${{ steps.app-token.outputs.token }}@github.com/org/filone-deploy deploy
```

### 🥈 #2 — Quadlet + `ansible-pull` on a timer

Same repo/CI/digest spine; the node's reconcile is a playbook run locally on a timer. Handlers give idempotent restart-only-what-changed, plus templating and health-check tasks — less bespoke shell than #1, and the manual tier is expressed by simply having no restart handler.

**Costs:** Ansible on every box (awkward on immutable `/usr` — layer it or run from a container), version drift across a 30-box fleet, and the non-idempotent `daemon-reload` trap (gate restarts on file-change facts, not the reload task). Ensure the playbook is re-entrant so a half-succeeded run doesn't double-restart.

### 🥉 #3 — Docker Compose + Portainer CE GitOps (polling)

Choose **only if** you standardise on a mutable OS _and_ want a management UI for a small operator team. Good visibility with little custom code; Renovate still drives Caddy/Postgres/OpenBao PRs and CI still commits lock-step digests.

**Costs and failure modes:** Docker daemon (root) plus Portainer as persistent parts; a real penalty on bootc (layered Docker). Mixed cadence is project-level, so protecting Postgres _and now OpenBao_ from an unintended recreate takes care — pin digests, avoid `--remove-orphans`, name services explicitly, never a bare project-wide `up -d`. Config changes need the hash in a **label** rather than an annotation. You lose the `OnFailure=` alerting path and per-unit cgroup directives from §2.5, and must rebuild both in the script and in compose-level `ulimits`/`mem_swappiness`. "Force redeployment / git-wins" may be BE-only — verify on CE.

---

## 10. What would change the ranking

- **OS lands on Ubuntu LTS specifically:** Quadlet needs a Podman the archive doesn't ship (24.04 = 4.9.3, no `Notify=healthy`). If you won't add a backport, Compose (#3) rises.
- **Operators want a GUI:** Portainer or Komodo v2 rises, accepting Compose lock-in, the root daemon, and the loss of §2.5.
- **Piri's drain gets shortened** (see §11, first item): the coalescing argument weakens, and a debounced webhook or a Tailscale push becomes attractive again for faster feedback.
- **You want zero git credential on partner boxes:** the OCI-artifact config path becomes the default node-read mechanism, further favouring #1.
- **Central control with agent-initiated connections becomes acceptable:** Komodo v2 (PKI auth, outbound Periphery) gets competitive for fleet visibility — but stays Compose-only.
- **A first-class non-K8s GitOps agent that manages Quadlet natively appears:** would leapfrog the bespoke reconciler. Nothing qualifies today.
- **OpenBao moves off-box** (if a design change removed it from the read path): Compose's project-level reconcile becomes materially less risky and the gap narrows.

---

## 11. Open questions to verify at implementation time

1. **Can Piri's graceful-shutdown window be shortened for deploys?** It is configurable and defaults to up to 60 minutes. That figure currently sets your deploy cadence _and_ forces the alerting complexity in §7. If 5–10 minutes is safe — with `pdp-gate` handling the proof-window concern independently, which is its whole job — deploys get an order of magnitude cheaper at no cost to correctness. **Highest-value question here.** Needs the Piri team's view on what the long drain protects.
2. **Which Piri/Ingot settings genuinely need to be boot-time config?** Restart-only config hurts in proportion to churn. Log levels, rate limits, feature flags and tunables should not live in a file read at boot; moving them to a runtime-queryable store (Postgres or OpenBao KV, both already present) turns a restart into a no-op. An application change, but the actual fix.
3. **Is Ingot's graceful shutdown worth prioritising?** It is the customer-facing S3 endpoint with no graceful shutdown and no reload, so every deploy drops in-flight requests. SDK retries absorb most of it; pre-signed URLs in browsers do not retry at all. This buys more availability than any deployment machinery, and nothing in the deployment layer substitutes for it. Blue/green is foreclosed — two concurrent Ingot instances are documented as undefined behaviour.
4. **Exact transit-seal token config key / env var** for OpenBao, and whether it can be supplied as a `podman secret` env without appearing in `podman inspect`.
5. **Central OpenBao availability target and operator** — now the region's boot-time dependency, and the bound on regional recovery time. The key-management RFC leaves this open.
6. **Does OpenBao's SIGHUP path interact badly with the transit seal?** openbao #2915 reports SIGHUP with a declarative file audit stanza wedging the _gcpckms_ seal client until a full restart. Different seal type, same reload-vs-seal-client interaction class — and the RFC leaves the audit device an open question. If you enable file audit, test SIGHUP against the transit seal explicitly before relying on reload.
7. **Ingot health check must exercise a real transit unwrap** (§2.3). Confirm Ingot exposes something suitable, or request it.
8. **Measure the real boot sequence** — boot → network → unseal → OpenBao healthy → Ingot/Piri start — on partner hardware.
9. **Exact Podman version on the chosen OS**, and thus whether `Notify=healthy` (≥5.0), `ReloadCmd=` (≥5.6), `podman quadlet` (≥5.6) and bootc bound-image GC are all available.
10. **Portainer CE vs BE** for git-wins force redeployment, if #3 is in play.
11. **Renovate `packageRules`** for the Piri+Ingot group and the Postgres/OpenBao no-auto-merge policy — validate with a dry run.
12. **Prototype the OCI-artifact config path** (`oras push`/`pull` with a per-node robot account) and confirm the reconciler can diff and apply from it as cleanly as from git.
13. **Deadman thresholds** tuned to the real poll interval and worst-case drain; confirm textfile metrics survive reboots under immutable-`/var` semantics.

---

## 12. Caveats

- **Both RFCs are Status: Design**, and the appliance strategy RFC is an unmerged draft (PR #19). The region-wrap amendment is recent (2026-08) and the algorithm already moved once, from A256KW to transit AES-GCM. Re-check before building.
- **The "OpenBao refuses to start when central is unreachable" behaviour pins to 2.6.1.** Re-verify on your deployed version — a future change from hard-fail to degraded-start would silently remove the startup-kill property this design depends on.
- **Fast-moving Podman surface.** Podman reached 5.x in 2024 and the 6.0 series in mid-2026 (billed with a major Quadlet overhaul); the last 5-series is 5.8.x. Verify features against the version on your image, and prefer podman-systemd.unit(5) and release notes over any blog post.
- **The config-hash-in-unit pattern is an adaptation** of the Kubernetes checksum-annotation convention, not a documented Podman or Compose feature. It relies only on documented behaviour (unit files are text; `Annotation=` is a supported key) but exercise it end to end.
- **The `Notify=healthy` timeout bug (#27290)** was open as of late 2025. Check whether it's fixed on your version, but set `TimeoutStartSec` explicitly regardless.
- **`podman auto-update` vs digest-pinning are mutually exclusive by design.** Verified against current docs, but re-check the man page in case a digest-aware mode appears.
- **Ubuntu Podman versions** come from archive packages and can be bypassed with backports; confirm for your exact image.
- **`caddy.storage.vault` maturity** is a read of a third-party module's activity, not a maintenance commitment — check it yourself if you reject the LUKS/Clevis alternative in §6.
- **Vendor sources:** Portainer's git-as-source-of-truth framing and the CE/BE split are vendor docs; Komodo capability claims are partly vendor and community; Tailscale's zero-inbound framing is accurate but is their marketing.
- **Rollback:** the `git revert` model is sound for every recommended pull-based combo, since the node converges to origin. It breaks with `podman auto-update`'s auto-rollback and with any mechanism where local state can diverge from the deploy ref — which is why those are downranked. Nothing here makes a **schema change** safe to roll back; expand/contract migrations are the actual mechanism, and pre-migration snapshots stop scaling in the tens of GB.
- **Health-check quality bounds everything.** Auto-detection catches only what the health check detects; a container that starts and serves errors reads as a successful deploy under every option here — see §2.3 for the specific instance that now matters.
- **The reconcile script is an illustrative skeleton.** Deliberately exercise the config-only path, the manual-tier path, and a knowingly-broken image before relying on any of them.

---

# Adopting Komodo v2 for the FilOne Appliance: Concrete Mechanics and an Honest Comparison to the Quadlet + git-timer Baseline

## TL;DR

- **Keep the Podman Quadlet + git-pull systemd-timer + commit-the-digest baseline as the primary CD mechanism for the partner fleet.** Komodo v2's outbound Periphery genuinely satisfies the "no inbound ports" constraint, but adopting it means putting Docker on the box (fighting bootc/FCOS, which ship Podman), running a central Core+MongoDB that can execute arbitrary commands on all 300 nodes, and giving up systemd's ordering, `Notify=healthy`, `ExecStartPre` fail-closed semantics, and journald — for a dashboard and webhook-speed deploys you can approximate more cheaply.
- **If you want Komodo's benefits, adopt "Alternative 1" (Komodo manages only Piri+Ingot) but understand it is a genuine two-control-plane split**, not a free win: two runtimes (Docker+Podman) on one box with documented iptables/netavark conflicts, two config-delivery paths, and no shared transaction for cross-tier changes. "Alternative 2" (Komodo manages everything) is the cleaner single-runtime story but forces Postgres/OpenBao into a Compose model that loses systemd's guarantees and risks bouncing the database.
- **Komodo is a fast-moving, effectively single-maintainer project (Maxwell Becker / `mbecker20`).** Several of the exact behaviors you depend on — `pre_deploy` non-zero-exit abort semantics (your `pdp-gate`), config-file-only redeploy via TOML sync (issue #1381), and any automatic rollback — are either unverified in source or known-buggy/absent. Re-verify against the running version at implementation time.

---

## Key Findings

1. **Outbound Periphery is real and production-usable, but Core must be internet-reachable.** In v2 the Periphery agent dials Core over a WebSocket (`wss://<core>/ws/periphery?server=<name>`), authenticated with a Noise-protocol handshake over auto-generated Ed25519 keypairs. All operations — deploys, log streaming, terminal/exec, file access — relay back through that single outbound connection, so the partner VM needs **zero inbound ports** beyond :443 for customer traffic. This is the one hard constraint Komodo clearly satisfies that disqualified Kamal and Ansible-push.

2. **Komodo is Docker-only in practice.** It "uses Docker as the container engine"; Podman is "supported via the podman → docker alias," which in reality means aliasing binaries and bind-mounting `podman.sock` to `docker.sock`. There are open issues about rootless permission failures, SELinux relabeling, and stacks not auto-starting after reboot under Podman. It does **not** manage Quadlet units. This matters enormously on bootc/FCOS, which ship Podman and not Docker.

3. **"Git as source of truth" in Komodo is git → Core's MongoDB → nodes, not git → nodes.** Resource Sync ingests TOML from git into Core's database; the database remains the live control state. This is a _shadow source of truth_ competing with git, unlike the baseline where the git commit that the node pulled _is_ the truth.

4. **The `pdp-gate` fits as a Stack `pre_deploy` SystemCommand — but its fail-closed behavior is unverified.** I could not confirm from source whether a non-zero `pre_deploy` exit actually aborts `docker compose up`. This is the single most important open item and must be tested before relying on it. Compare to `ExecStartPre=` in a Quadlet unit, where non-zero exit reliably fails the unit start (documented systemd behavior).

5. **Komodo has no automatic rollback and does not gate deploy success on health.** `DeployStack` runs `docker compose up -d` (detached) and reports success when containers are _created/started_, not _healthy_. Health is evaluated asynchronously and only produces alerts. There is no revert-to-previous-digest on failure — you get a red status. This is weaker than the baseline's `Notify=healthy` + `TimeoutStartSec` + `OnFailure=`.

6. **Blast radius is the central security objection.** A compromised Komodo Core (or an admin account on it) can execute arbitrary commands on every connected node — 300 boxes. The baseline node holds only a read-only registry robot account and a read-only git pull; worst case is a bad commit that the whole fleet is at least equally exposed to. Komodo's PKI authenticates the channel but does not reduce Core's authority; RBAC/user-groups limit _operator_ accounts but not a Core compromise.

---

## Details

### A. Komodo v2 current state (verified against primary sources)

**Version / cadence / maintainer.** The canonical repo is `moghtech/komodo` (formerly `mbecker20/monitor`), Rust + TypeScript, GPL-3.0, 11.2k stars per the moghtech/komodo GitHub repo (Fork 318, Star 11.2k as shown on the v2.1.x release pages). Komodo v2.0.0 was released by mbecker20 on 24 Mar 2026, per komo.do/docs/releases/v2.0.0 ("v2.0.0 – March 2026"), introducing "Outbound periphery" and "PKI authentication: Core and Periphery now authenticate with auto-generated key pairs and automatic rotation." v2.1.0 was released by mbecker20 on 01 Apr 2026 (23:27), per the GitHub release page; v2.1.1 on 02 Apr, v2.1.2 on 10 Apr; the latest is v2.3.1, released 31 Jul 2026. v2.1.0 added Docker Swarm node management and Swarm stack env-file support; the v2.2.0 GitHub changelog reads "Stack: Allow excluding specific services from Global Auto Update by @mateuszziolkowski" — notably the feature was contributed by an outside contributor, not the maintainer. Development velocity is very high. The project is **effectively single-maintainer** — Maxwell Becker (`mbecker20`), operating as Mogh Technologies Inc. A handful of outside contributors appear in changelogs (`@mvanhorn`, `@mateuszziolkowski`, `@ChanningHe`, `@Dougley`), and a community member describes it as "great software, one maintainer, lot's of work." For a component in the trust path of a 300-node fleet, treat the **bus factor as ~1**.

**Outbound Periphery mechanics.** Periphery is a small, stateless binary receiving commands from Core over a bidirectional WebSocket. In outbound mode it is configured with `core_address` (e.g. `ws://…:9120` or `https://…` fronted by a reverse proxy), `connect_as` (the Server name/hostname), and a one-time `onboarding_key` generated in Core's Settings → Onboarding. On connect, a Noise handshake establishes mutual PKI trust; the private keys live in `/config/keys` (container) or `<root_directory>/keys` (systemd, default `/etc/komodo/keys`) and must be persisted. Everything — deploy, `docker compose logs`, browser terminals (`execute_server_terminal`, `execute_container_terminal`), file browsing — relays over that connection, analogous to K3s's reverse tunnel or `nomad alloc exec`. **No inbound access is required for any operator function.** On network partition, community operators report Periphery entering exponential backoff and sometimes needing `systemctl restart periphery` to reconnect (a known reconnection weakness); crucially, **the containers keep running during a partition — only Komodo visibility/control is lost.** Core **must be internet-reachable** by every Periphery, so hardening Core (2FA/TOTP, OIDC, IP allowlisting on the `/ws/periphery` path via the reverse proxy) is essential.

**Podman support.** Officially "supported via the podman → docker alias." Practically: run Periphery as a container and bind the host's `/run/podman/podman.sock` to `/var/run/docker.sock` (systemd Periphery has no config to change the socket path, per issue #544), alias `podman-compose`→`docker-compose`, and expect to add `security_opt: [label=disable]` to get past SELinux denials on the socket (Discussion #293). Known gaps: stacks don't auto-start on reboot under Podman even with `restart: unless-stopped` (issue #315); rootless is only partially working (issue #974). Issue #448 ("Does Komodo Support Podman Really?") captures the community's uncertainty. **Verdict: Podman works as a tech demo, not as a production substrate you'd bet 300 nodes on.** If you adopt Komodo, plan on Docker.

**Resource Sync / TOML.** Komodo can serialize and reconcile Servers, Stacks, Deployments, Builds, Repos, Procedures, Actions, Alerters, Builders, and ResourceSyncs themselves to/from TOML, sourced from UI, "files on host," or a git repo. Reconciliation (`RunSync`) fetches remote TOML, diffs against Core's database (Create/Update/Delete deltas), orders by dependency, and — by default — surfaces pending changes for **manual confirmation** in the UI. **Managed mode** grants Core full authority: resources matching the sync's filters but absent from TOML are _deleted_. Deployment ordering across servers is expressible via an `after` array (like a cross-server `depends_on`). **This gives real drift detection and a "git wins" force-redeploy semantic — but note the flow is git → Core DB, not git → node.** Core's database remains the live source of truth; TOML is imported into it. A "server with a region label converges automatically" pattern is _achievable_ (declare the Server + its Stacks in TOML, match by tag), but the node still has to be enrolled into Core first (see provisioning below), and auto-execution of syncs has rough edges — in moghtech/komodo issue #1120 a reporter set `deploy = true ... auto_update = true poll_for_updates = true` and noted "When I push changes to GitHub, the webhook triggers correctly" yet the stack did not redeploy.

**Digest pinning & signature verification.** Compose stacks can pin `image: repo@sha256:…` digests natively — that's just Compose, and it works. **There is no built-in cosign/sigstore verification.** You would implement signature verification as a `pre_deploy` SystemCommand (`cosign verify … && exit 0 || exit 1`) — which lands you back on the unverified question of whether a non-zero `pre_deploy` aborts the deploy. In the baseline, cosign verification is a natural `ExecStartPre=` step with well-defined fail-closed behavior.

**Procedures / Actions / hooks.** Procedures compose executions (RunBuild, DeployStack, BatchDeployStackIfChanged, etc.) into sequential stages of parallel steps. Actions are TypeScript scripts against a pre-initialized Komodo API client, schedulable via CRON. Per-stack `pre_deploy`/`post_deploy` are `SystemCommand` fields. **This is where `pdp-gate` lives** (a `pre_deploy` command that shells out to the PDP-window checker). The gap: (a) whether non-zero exit aborts is unverified in source; (b) Discussion #673 confirms hooks currently receive little resource metadata (no stack name in env), so the gate script must be self-sufficient; (c) there is no native "defer and retry later" — you'd emulate it by failing the deploy and relying on the next poll/webhook to re-attempt, which only works if fail-closed abort actually holds.

**Secrets.** Komodo Variables/Secrets are key-value pairs **stored in Core's database**; marking one "secret" hides it from non-admin users and logs but it is still in the DB. Alternatively, `[secrets]` blocks in Core's `config.toml` are queryable-by-key but values are never exposed via API; `[secrets]` in a _Periphery's_ config file are local to that one server and **never traverse the network** — the closest fit to your "node holds minimal, non-exportable" requirement, but it means seeding each node's Periphery config with material, which competes with the OpenBao model. Interpolation (`[[VAR]]`) happens **Core-side before dispatch to Periphery**, meaning secrets transit Core. **Komodo has no OpenBao/Vault integration**; the documented pattern is "use Vault alongside Komodo," typically by having a `pre_deploy`/pre-pull step render secrets (Infisical is shown in issue #324). For your semi-trusted-partner threat model, the cleanest arrangement keeps OpenBao-agent-to-tmpfs exactly as today and does _not_ route regional secrets through Core's DB.

**Per-service selective restart.** `DeployStack` with an explicit service list runs `docker compose up -d <svc1> <svc2>` — so selective restart is possible. Auto-update by default only redeploys services with a new image digest; `auto_update_all_services=true` redeploys the whole project. Services can be excluded from the auto-update set (v2.2.0 feature) or tagged `komodo.skip`. `destroy_before_deploy` runs `docker compose down` first (dangerous for stateful services). **This is the mechanism that would keep Postgres from being bounced under Alternative 2 — but it is opt-in configuration you must get exactly right, versus the baseline where Postgres is simply a different systemd unit that nothing touches.**

**Core footprint / DB / HA.** Per Salt Data Blog's "Managing Docker Across Multiple Servers with Komodo" (updated April 5, 2026): "Resource usage: Low — Core + MongoDB typically under 256 MB RAM combined." Database options, per komo.do/docs/setup: "MongoDB is the recommended database for Komodo. It stores all resource configuration, user accounts, audit logs, and system state ... If you setup Komodo using Postgres or Sqlite options prior to Komodo v1.18.0, you are using FerretDB v1." FerretDB v2 is a MongoDB-compatible shim over Postgres + DocumentDB extension; SQLite and FerretDB v1 were dropped in v1.18.0. **There is no HA story** — Core is a singleton. When Core is down or upgrading, **the fleet's containers keep running**, but you lose deploys, log/terminal access, and drift reconciliation until it returns. Core upgrades touch a database schema (with a documented `km database v1-downgrade` path for the v1→v2 jump).

**CVEs / security surface.** I found **no published CVEs or GitHub security advisories for `moghtech/komodo`** (the "Komodo" CVEs in databases are for an unrelated Komodo CMS and for "Comodo"). Absence of CVEs in a young, single-maintainer project is _not_ evidence of security maturity. The material fact is architectural: **Core can run arbitrary commands (including browser terminals) on every connected node.** Auth to the web UI supports local username/password with TOTP/passkey 2FA, plus OIDC/GitHub/Google. RBAC exists (below) but does not constrain a Core compromise.

**RBAC / scoping.** Permissions are assigned to Users or (preferably) User Groups at four levels — None / Read / Execute / Write — per resource, with additional feature gates (Logs, Terminal, Inspect, Processes). `KOMODO_TRANSPARENT_MODE` flips the default to Read. User Groups can be declared in Resource Syncs for scale. **This meaningfully bounds a compromised _operator_ account** (e.g. an operator with Execute-but-not-Write on only the app-tier stacks on their region's server). **It does not bound a compromised Core**, which is the higher-consequence event.

### B. Komodo on immutable (bootc / Fedora CoreOS) vs mutable (Debian 13 "trixie")

**Periphery install.** Periphery ships as (a) a single static-ish binary installed by `setup-periphery.py` as a systemd unit (default root `/etc/komodo`), or (b) an official container image (`ghcr.io/moghtech/komodo-periphery:2`). Komodo's own docs and the community strongly recommend the **systemd/host binary**, because running Periphery in a container while it must drive the container socket and read/write stack files on the host creates volume-mount and path-translation headaches (Discussions #180, #251). v2 binaries are built against OpenSSL 3 (Debian Bookworm base), so they run on **Debian 13 trixie** fine; they will _not_ run on OpenSSL-1 systems (Bullseye/Ubuntu 20.04) without the container or a self-compile.

- **On Debian 13 (mutable):** straightforward. Install `docker-ce` (Docker's repo) or Debian's `docker.io`, drop the Periphery binary in `/usr/local/bin` via the installer, done. This is the well-trodden path.
- **On bootc / FCOS (immutable):** two frictions. First, **Periphery-as-host-binary is not image-managed** — putting a binary in `/usr/local/bin` on an ostree system is writable but outside the image, defeating the immutability guarantee; the "correct" bootc approach is to layer it into the image (a `Containerfile` build for bootc, or `rpm-ostree`/Ignition for FCOS) or run containerized Periphery as a Quadlet unit. Second, and larger:

**Docker on bootc/FCOS.** FCOS and Fedora/CentOS bootc images ship **Podman, not Docker**. Getting Docker Engine on requires layering `docker-ce` (or the poorly-maintained `moby-engine`) — on FCOS via an Ignition systemd unit that runs `rpm-ostree override remove docker containerd runc` (or `podman-docker`) then `rpm-ostree install docker-ce docker-ce-cli containerd.io …` and reboots; on bootc via `RUN` steps in the image `Containerfile`. Per the Fedora moby-engine maintenance note (HackMD, @travier): "we will have to recommend to its users on Fedora CoreOS that they either switch to the official Docker packages ... or move to containerd or podman"; see also coreos/fedora-coreos-tracker issue #1723 "Consider dropping moby-engine from the base image." **This is fighting the platform.** It works, and community write-ups exist, but you are permanently swimming upstream on the OS you chose specifically to be boring and image-managed.

**The mixed-runtime problem (Docker + Podman on one box).** This is a documented minefield, not a clean coexistence:

- **Networking:** Docker inserts a `FORWARD` policy DROP plus `DOCKER-USER`/`DOCKER-ISOLATION` chains; Podman's netavark inserts `NETAVARK_FORWARD`/`NETAVARK_INPUT` chains. Verbatim from the Fedora Project Wiki, Changes/NetavarkNftablesDefault: "When using both rootful podman and docker together the podman containers will not have external network connectivity. This is because docker adds a iptables rule to block all forwarding." (Confirmed in podman issue #24486: Docker sets "a policy of DROP on the iptables FORWARD chain.") Fixing it means hand-writing iptables rules or forcing driver alignment. Podman 6.0 is moving off iptables to nftables entirely, adding another moving target.
- **Storage:** Docker uses `overlay2`; Podman uses `containers/storage`. The same image layers get stored twice — real disk cost on a single-VM appliance.
- **Cross-boundary service networking:** Caddy under Podman proxying to Ingot under Docker cannot use a shared Compose/pod network; you'd publish Ingot on `127.0.0.1:<port>` and have Caddy reverse-proxy to loopback, or use host networking. Workable, but it's a hand-managed seam exactly where you least want ambiguity (the :443 owner).

**Debian 13 specifics.** Podman 5.4.2 is in trixie (`Notify=healthy` support present; no `podman quadlet` _subcommand_ — Quadlet is driven by generators, not that subcommand, so this is fine). Docker via `docker-ce` or `docker.io`. Komodo-on-Debian is the reference environment in most community guides.

### C. The two alternatives, walked end-to-end

Throughout: the **baseline** for comparison is a pre-baked image (bootc/qcow2) that boots with Quadlet units + a git-pull systemd timer + read-only registry robot + OpenBao agent, converging **unattended** with zero human clicks per node.

#### Alternative 1 — Komodo manages Piri + Ingot only; Quadlet keeps Postgres, Caddy, OpenBao, Alloy

**1. Provisioning a new node.** Partner hands over a VM. FilOne boots it from a pre-baked image whose Ignition/cloud-init (a) lays down the Quadlet units for Caddy/OpenBao/Postgres/Alloy exactly as today, and (b) installs Docker + Periphery. Periphery starts, reads `core_address` + `connect_as` + a **one-time onboarding key**, dials Core, completes the Noise handshake, and appears as a Server. _Which stacks it runs_ can be declared in a Resource Sync keyed by a region tag so the Piri+Ingot stack attaches automatically — but the onboarding key must be seeded per node (via Ignition secret), and privileged onboarding keys can rewrite an existing Server's expected key, so key handling is security-sensitive. **Bootstrap ordering across the boundary is on you:** Compose `depends_on` cannot see the Quadlet-managed OpenBao/Postgres, so Piri/Ingot's Compose services need their own wait-for-readiness (healthcheck-gated entrypoint probing OpenBao's unseal status and Postgres). OpenBao unseal + initial secret material flow through the **existing** OpenBao-agent path — Komodo is not involved, which is good. **Time-to-serving:** comparable to baseline for the infra tier; the app tier adds the Core round-trip and (if not fully sync-automated) a human attaching the stack. **Net: more moving parts than baseline, and at least one security-sensitive per-node secret (onboarding key).**

**2. Image-version bump.** CI builds Piri/Ingot, pushes digests, and commits the pinned digest — but now to **two destinations**: the git repo (infra tier, consumed by the timer) and Komodo (app tier). For the app tier, either CI calls Komodo's API/webhook, or Komodo polls the registry. With outbound Periphery **there is no inbound webhook to the node** — the webhook hits _Core_, and Core pushes `DeployStack` down the existing outbound tunnel. Webhook path: seconds. Poll path: bounded by `KOMODO_RESOURCE_POLL_INTERVAL`. Only the changed service is recreated (`docker compose up -d ingot`). `pdp-gate` runs as `pre_deploy` **if** non-zero-exit abort holds (unverified). **The split-brain risk is real:** a change that must be atomic across tiers — e.g. a Caddyfile route change (Quadlet tier) plus an Ingot bump (Komodo tier) that depend on each other — has **no shared transaction**. Two independent controllers apply two halves on two schedules; you can land in a state where new Ingot is live but Caddy hasn't reloaded, or vice versa. The baseline applies both in one git commit under one reconciler.

**3. Config-file change.** For Caddy/Postgres/OpenBao configs, nothing changes — they flow through the **git-timer path** exactly as today (`git → copy into place → daemon-reload → selective systemctl reload/restart`). So **Komodo buys you nothing for the infra-tier config case.** For Piri/Ingot config: Periphery clones the whole git repo to `repo_dir` on the host and relative bind-mounts work (with systemd Periphery). But **detecting a config-only change is fragile** — Komodo's `DeployStackIfChanged` keys on the compose file / git hash and on `config_files` entries with `requires="Redeploy"`; moghtech/komodo issue #1381 (opened by D-N-Corre, 23 Apr 2026) documents: "config_files entries synced from TOML don't trigger auto-redeploy on git push — even with explicit requires: Redeploy ... The exact same entries added manually via the UI ... DO trigger auto-redeploy." It is now marked Closed but the v2.2.x fix status is unconfirmed. There is no reload-not-restart primitive; a config change recreates the container (or you script `caddy reload`/`pg_ctl reload` as an Action). **Under Alternative 1 the app tier gains little for config changes and inherits a known change-detection footgun.**

#### Alternative 2 — Komodo manages everything as Compose stack(s); no Quadlet (except maybe to bootstrap Periphery)

**1. Provisioning.** Cleanest runtime story: **one runtime (Docker)**, one control plane. Pre-baked image installs Docker + Periphery (Periphery itself ideally a systemd unit or a single Quadlet unit to bootstrap it). Everything else — Caddy, OpenBao, Postgres, Alloy, Piri, Ingot — is Compose services. Ordering can use Compose `depends_on: condition: service_healthy` + healthchecks, so OpenBao-unseal-wait and Postgres-readiness are expressible _within_ the project. **But:** OpenBao unseal still needs its material from somewhere; putting the unseal/KEK path into Compose env or Komodo Variables would route it through Core's DB — **exactly what constraint 6 forbids** — so you keep an OpenBao-agent sidecar pattern anyway. Caddy owning :443 and doing TLS-ALPN-01 works under Docker but now Docker owns the host's :443 NAT rules.

**2. Image-version bump.** Single path: CI → digest → Komodo (webhook to Core → push over tunnel, or poll). Only the changed service recreates _if_ you leave `auto_update_all_services=false`. **The database-bounce risk is the headline hazard:** if a mis-set `auto_update_all_services=true`, a `destroy_before_deploy=true`, or a whole-project `docker compose up` (e.g. because a shared top-level element like a network or `configs:` changed) sweeps the project, **Postgres and OpenBao get recreated.** Prevention is entirely configuration discipline: exclude Postgres/OpenBao from auto-update, never `destroy_before_deploy`, and always deploy Piri/Ingot by explicit service list. This is _achievable_ but it makes the safety of your database a property of getting several opt-in toggles right, versus the baseline where Postgres is a separate systemd unit that the app-tier reconciler structurally cannot touch. `pdp-gate` as `pre_deploy` (abort semantics unverified). **No automatic rollback**; a failed Piri deploy leaves a red status, not a revert. A `git revert` + re-sync rolls forward to the old digest, which is the practical rollback — but it's forward-only reconvergence, not Komodo detecting failure and reverting.

**3. Config-file change.** Now _all_ config (Caddyfile, `postgresql.conf`, `pg_hba.conf`, OpenBao policies, Piri/Ingot TOML) rides the Komodo path: committed to the stack's git repo, cloned to the host by Periphery, bind-mounted. **You inherit the change-detection fragility for everything**, including the most sensitive files. Config-only changes need `config_files` + `requires="Redeploy"` (or `Restart`) wired correctly, and the TOML-sync redeploy bug (#1381) is directly in your path if you drive config declaratively. A Postgres `pg_hba.conf` change would trigger a **container recreate** unless you add an Action to `pg_ctl reload` instead — reimplementing what `systemctl reload postgresql` gives you for free. **This is the case where the baseline is strongest and Komodo weakest.**

---

### Comparison table

| Dimension                        | Quadlet + git-timer (baseline)                                | Komodo — app tier only (Alt 1)                                | Komodo — everything (Alt 2)                                            |
| -------------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------- | ---------------------------------------------------------------------- |
| **Git as source of truth**       | Yes — the pulled commit _is_ the truth on the node            | Split: infra tier git-true; app tier git → Core DB → node     | git → Core MongoDB → node (DB is live truth; shadow of git)            |
| **Config delivery**              | git → copy → daemon-reload → reload/restart                   | Infra: git-timer. App: Periphery clones repo, bind-mount      | Periphery clones repo, bind-mount for all services                     |
| **Config-only change detection** | Reliable (timer re-applies every tick)                        | Infra reliable; app fragile (#1381, no reload primitive)      | Fragile for all; recreate-not-reload; #1381 risk                       |
| **Selective restart**            | Native (`systemctl restart <unit>`)                           | Per-service `compose up -d <svc>`                             | Per-service, but whole-project sweep risk to Postgres/OpenBao          |
| **pdp-gate placement**           | `ExecStartPre=` — documented fail-closed on non-zero          | `pre_deploy` SystemCommand — **abort on non-zero UNVERIFIED** | Same — `pre_deploy`, abort unverified                                  |
| **Health gating / rollback**     | `Notify=healthy` + `TimeoutStartSec` + `OnFailure=`           | `compose up -d` detached; no health gate; no auto-rollback    | Same; red status only; rollback = forward re-converge                  |
| **Secrets**                      | OpenBao agent → tmpfs; no sprawl                              | OpenBao unchanged for infra; app secrets risk Core DB         | Risk of routing secrets through Core DB unless kept in OpenBao sidecar |
| **Inbound ports on node**        | None (git pull + registry pull outbound)                      | None (Periphery dials out)                                    | None (Periphery dials out)                                             |
| **Credential blast radius**      | Node holds read-only pull + git read; worst case = bad commit | Core can exec on all nodes (app tier) + git-timer creds       | **Core can exec on all 300 nodes** (full box)                          |
| **bootc/FCOS compatibility**     | Native (Podman + systemd, image-managed)                      | Poor — needs Docker layered in + Docker/Podman coexistence    | Poor — needs Docker layered in (but single runtime)                    |
| **Moving parts**                 | Fewest (systemd + timer + git)                                | Most (2 runtimes, 2 control planes, 2 config paths)           | Medium (1 runtime, +Core+MongoDB centrally)                            |
| **Fleet visibility**             | Build it yourself (or none)                                   | Dashboard + logs + terminal for app tier                      | Dashboard + logs + terminal for whole box                              |

---

## Recommendations

**Stage 0 — Default position: keep the baseline for the partner fleet.** For single-node-per-region, singleton, semi-trusted-operator appliances on an immutable OS, the Quadlet + git-timer + commit-the-digest design is better aligned on every hard constraint except "nice dashboard." It keeps blast radius minimal (read-only creds, no central executor), keeps systemd's ordering/`Notify=healthy`/`ExecStartPre` fail-closed semantics that your `pdp-gate` and health-gating depend on, and stays native on bootc. Do **not** adopt Komodo to solve a problem the baseline already solves.

**Stage 1 — Address the actual gap (fleet visibility) cheaply first.** The genuine thing Komodo buys is a "which node is on which version" dashboard, aggregated logs without standing up Loki, and one-off break-glass terminal access. Before adopting a second control plane, close these with lower-blast-radius tools: ship the digest each node is running as a telemetry field through the Alloy path you already have; use Grafana for the version matrix; use your existing outbound telemetry for logs; and keep break-glass as an auditable, outbound-initiated session (e.g. a Tailscale SSH or boundary session) rather than a standing executor.

**Stage 2 — If you still want Komodo, pilot Alternative 1 on Debian 13, not bootc, and only for FilOne's own trust domain first.** Komodo is a materially better fit for **FilOne's own centrally-controlled, fully-trusted infrastructure** than for the semi-trusted partner fleet, because there the "Core can exec everywhere" property is inside your own trust boundary rather than exposing partners to your control plane and vice versa. Run Core with MongoDB, 2FA, OIDC, `/ws/periphery` IP-allowlisted, and per-server RBAC scoping so operators can Execute only their stacks. Manage only the frequently-changing app tier (Piri/Ingot).

**Falsifiable conditions under which Komodo becomes the right call for the partner fleet — all must hold:**

1. **`pre_deploy` non-zero exit provably aborts the deploy** (test it: a `pre_deploy` that `exit 2` must leave the old container running and mark the deploy failed). If it proceeds anyway, `pdp-gate` cannot fail closed and Komodo is disqualified for Piri.
2. **Config-file-only changes reliably trigger redeploy via your chosen declaration path** (verify #1381's fix in the running version; if you drive config via TOML sync and it silently no-ops, that's a data-integrity hazard).
3. **You accept Docker on the box** and either abandon bootc/FCOS or commit to maintaining a Docker-layered image and the Docker/Podman networking seam.
4. **You accept a central Core as a first-class production system** (HA-less singleton, DB backups, schema-migration upgrades, internet-reachable, in the partner trust path) and have a story for Core compromise ≈ fleet compromise.
5. **The webhook-speed deploy latency is worth the second control plane** — quantify: if merge-to-running on the baseline poll timer is already acceptable (minutes), the seconds Komodo saves rarely justify the cost.

**Benchmarks that would change the recommendation:** a released Komodo version with (a) documented fail-closed `pre_deploy`, (b) native health-gated deploys with auto-rollback to previous digest, (c) first-class Podman/Quadlet support, and (d) a second maintainer / governance beyond one person. Until at least (a)+(b) land, do not put Komodo in front of Piri's restart path.

---

## Caveats (what to re-verify at implementation time)

- **`pre_deploy` abort semantics (highest priority, UNVERIFIED).** Neither docs nor a source quote confirm that a non-zero `pre_deploy` exit aborts `docker compose up`. Test empirically before trusting `pdp-gate` to it. This is the load-bearing assumption for gating Piri restarts.
- **Config-file redeploy via TOML sync (issue #1381).** Reported broken at v2.1.2, marked closed, **fix in v2.2.x/v2.3.x unconfirmed.** If you drive config declaratively from git, verify a committed config change actually redeploys before relying on it.
- **No automatic rollback / no health-gated deploy success.** Confirmed from behavior (`compose up -d` detached, async health monitoring). If Komodo adds health-gated deploys later, revisit.
- **Podman support is second-class and version-sensitive** (issues #544, #315, #974, #448). Podman 6.0's move off iptables may further shift the Docker/Podman coexistence story.
- **Single-maintainer / bus factor ~1.** Fast-moving means behaviors and defaults change release-to-release (the current line has already reached v2.3.1 as of 31 Jul 2026); pin Periphery/Core versions and read each changelog.
- **Source-quality flags:** version/architecture facts here are from official docs (komo.do/docs), the GitHub repo/releases, `docs.rs/komodo_client`, and the source-cited DeepWiki. Operational reconnection/backoff behavior and the Docker/Podman coexistence pain are from community blogs (Salt Data, FoxxMD, homelabor.net) and GitHub issues/discussions, not official docs — treat them as strong-but-secondary. The `pre_deploy` behavior and #1381 fix status are explicitly unresolved. No CVEs were found for `moghtech/komodo`, but that is not a security clearance given the project's age and maintainer count.

---

# Operational Runbooks

## 0. The flow, and its moving parts

```
PR → main (staging dir)          ← humans + Dependabot; CI check: hashes valid, quadlet -dryrun
   → staging node converges      ← reconcile timer, unchanged
   → verify workflow             ← dispatched by the staging node on convergence; runs smoke tests
   → promotion                   ← manually dispatched; opens a PR copying staging state → prod dirs
   → prod nodes converge         ← each node applies only its own directory's diff
```

Conventions carried over from rev 1 and still binding: the reconciler is the only apply path; change detection is unit-file diffing (digests in `Image=`, config-subtree hashes in `Annotation=filone.config-hash=`); apply verbs from `services.yaml`; dependency ordering with `After=` + `Notify=healthy` and deliberately **no `Requires=`/`BindsTo=`** from apps onto Postgres/OpenBao.

New conventions introduced by this flow:

- **Rendering moves to pre-merge.** With no rendered branch, digest pins and config-hash annotations must be correct _in the PR_. `make render` updates hashes; a **required CI check** re-renders and fails on any mismatch, and runs `quadlet -dryrun` on the result. `main` never receives post-merge bot rewrites.
- **Update discovery: Dependabot (decided 2026-08-07).** One consequence must be designed around: **Dependabot has no Quadlet support** — it cannot parse `Image=` lines in `.container` files, nor pin digests in arbitrary file formats. The workaround is a **manifest shim**: a `deps/Dockerfile` (or minimal compose file) in the repo whose `FROM`/`image:` lines are the canonical upstream pins for Caddy, Postgres and OpenBao; Dependabot's `docker` ecosystem watches those, and `make render` propagates the pinned values into the staging units — the same render step the hash check already enforces, so a PR whose shim and units disagree fails CI. Piri/Ingot are unaffected (their digests are committed by our own CI on release, lock-step in one commit, no bot involved). Grouping and gating map as follows: Dependabot `groups` keeps related bumps in one PR; there is no `automerge:` key — auto-merge is GitHub's native PR auto-merge enabled by a small workflow on Caddy PRs only, and the human gate on Postgres/OpenBao is required-review/CODEOWNERS on their shim paths rather than a bot setting.
- **Node scoping.** Each node's reconciler diffs and applies **only its own directory**. A promotion commit touching ten node dirs is ten independent convergences.
- **Repo layout.** These runbooks assume the copy-the-directory layout (`nodes/staging/`, `nodes/<prod>/`, per-node overrides excluded from promotion). At fleet scale, prefer the pin-file variant — shared `stack/` + one `PIN` per node — where promotion is a one-line diff per node and canary waves are trivial; every "copy staging → prod dirs" step below becomes "bump PINs."
- **Verification is decoupled from merge, and push-triggered.** No workflow babysits a 60-minute Piri drain. When the **staging** reconciler completes a successful apply of a new SHA, its final step is an outbound `workflow_dispatch` call to the GitHub API (the applied SHA as an input), triggering the verify workflow. The workflow does **not** trust the payload: it independently confirms the reported SHA is the current tip of `main` for `nodes/staging/` and matches staging's pushed `deploy_sha` metric, then runs the smoke suite against staging's public surface: S3 put/get through Ingot, a read exercising a real transit unwrap, Caddy cert validity headroom, `pending_restart` count. A **low-frequency scheduled run remains as the backstop** for a missed dispatch (crash between apply and curl, GitHub API blip) — the webhook-plus-timer pattern, node-initiated.
- **Only staging holds a dispatch credential.** A fine-grained token scoped to `actions: write` on the config repo, host-bound in `systemd-creds` alongside the bootstrap set. **Decided 2026-08-07: `systemd-creds`, not OpenBao.** Storing it in the local OpenBao would be _permissible_ — unlike the deploy-loop credentials it is not self-bricking (a sealed OpenBao would merely skip the dispatch and the scheduled backstop covers verification) — but it buys only audit logging, not rotation: the local OpenBao is sealed by central, not replicated from it, so rotating the token would still mean an on-box write. Not worth the indirection on an internal box. If the fleet ever grows a central dispatch relay (a component beside the metrics store firing `workflow_dispatch` when staging's `deploy_sha` changes), revisit — that variant removes GitHub write credentials from all nodes entirely. Prod nodes never dispatch anything and hold no GitHub write credential of any kind — the blast-radius story for partner boxes is unchanged. The dispatch curl is best-effort: on failure the reconciler logs, pushes a `dispatch_skipped` metric, and exits zero — a missed dispatch is never a failed deploy.
- **Promotion is always manually dispatched** (`workflow_dispatch` or script) and **re-verifies at trigger time** — staging SHA current, tests green, staging deadman fresh — never trusting a stale check. It lands as a **promotion PR**, which is the change record. No tier auto-chains: a prod Caddy restart closes :443 for ~1–3 s, and we do not incur that without a human deciding when.
- **Promotion refuses (or loudly warns on) manual-tier diffs.** Postgres/OpenBao changes promote as _files_ but apply on each prod node only by hand; the script must surface this so "promoted" isn't mistaken for "applied."

---

## Threat model

**Decided 2026-08-07: Path B (mutable OS) is permitted on semi-trusted partner hardware.** This section records the reasoning, because it is a deliberate widening of the original design's assumptions, not a drift.

### Adversaries

1. **The partner** (semi-trusted): root on the VM and control of the hypervisor beneath it. Can read guest memory, modify any file, snapshot disks, intercept local traffic.
2. **External attackers**: reach the box only through the public surface (Caddy on :443 fronting Ingot) or by compromising the partner.
3. **Supply chain / us**: a bad or malicious change entering through the repo, the registry, or the enrollment path.

### What actually protects against a malicious partner — and note the OS is not on this list

Against adversary 1, an immutable OS provides approximately nothing: a party with hypervisor control can read unsealed secrets from memory, tamper with any running process, and modify the "immutable" disk image out of band. The controls that genuinely bound partner malice are architectural and **identical on both OS paths**:

- **The wallet is off the box** (signing service; keys never leave it) — the single highest-value control in the design.
- **OpenBao is transit-sealed by central** — protects secrets _at rest_ against disk theft/snapshotting, and gives central a startup-kill lever (revoke the seal credential and the region's OpenBao refuses to start). It does **not** protect secrets in the memory of a running node; that concession to adversary 1 was made on day one and is unchanged by the OS path.
- **Per-node, revocable credentials; no fleet-wide secrets.** One compromised partner exposes one node's read access. Single-use enrollment tokens make interception loud.
- **Outbound-only control plane** — a partner cannot pivot from their box into central; there is nothing listening to pivot into, and every credential their box holds is scoped to their node.
- **Image signature policy (cosign)** — what stops a partner running an image we did not sign is verification policy, not file permissions; a root partner can bypass it locally, but then their node fails our attestation/telemetry expectations, which is a detection story, not a prevention one.

**Blast radius of a fully malicious partner, either path:** their region's customer data readable, their region's service deniable, their node's scoped credentials abusable. Not: the wallet, other regions, central, or the fleet.

### What the immutable OS was actually buying — the real Path B concessions

Immutability's value was concentrated on adversaries 2 and 3 and on _negligence_, not malice:

1. **OS patch posture (adversary 2 — the serious one).** Path A patches the kernel/systemd/podman via atomic, FilOne-controlled image promotions. On Path B, an unpatched kernel or OpenSSL on an internet-facing box is an external-attacker risk that no amount of partner honesty mitigates. **Compensating control, mandatory:** `filone-enroll` configures unattended security upgrades (`unattended-upgrades` on Debian) as a hard requirement of enrollment, and reboot-requiring updates coordinate with `pdp-gate` via the same gated-reboot logic Path A uses.
2. **Drift and negligence.** A sloppy-but-honest partner (far more common than a malicious one) can break a mutable box in ways a read-only `/usr` prevents. **Compensating control:** the reconciler pushes an **OS-facts metric** each tick — kernel version, pending security updates count, podman version, unattended-upgrades enabled — so patch drift and tampering with the compensating controls are centrally visible per node. Alert on staleness and on any Path B node whose facts regress.
3. **No atomic rollback / A-B updates.** A botched OS update on Path B is an incident, not an automatic revert. Accepted; mitigated only by the boring maturity of Debian stable security updates.
4. **Weaker homogeneity.** Path A gives one fleet-wide OS digest to reason about; Path B nodes vary. Accepted; the OS-facts metric is the visibility floor, and §11 troubleshooting must not assume `bootc status` exists.
5. **Lost options:** bootc bound images, `/usr`-baked units, OS-image-level attestation. Accepted.

### Invariants that hold regardless of OS path

Wallet off-box; single-use enrollment; per-node scoped credentials; outbound-only; transit-sealed OpenBao with central startup-kill; digest-pinned, signature-verified images; deploy-loop credentials outside OpenBao; TPM-sealed `systemd-creds` where the hardware allows (Path B included — a TPM is a hardware property, not an OS one).

### Residual accepted risks (both paths, restated so nobody rediscovers them)

A malicious partner can read region-local plaintext in memory, serve stale-but-signed state, or deny service. A healthy-but-wrong deployment passes every gate below the smoke suite. Central OpenBao availability bounds regional recovery. These were accepted at design time and Path B does not change them.

## 1. Provision a new region

**Rewritten 2026-08-07: enrollment-based provisioning, two OS paths.** Nothing per-node is baked into any image. All provisioning logic lives in one idempotent, repo-versioned script, `filone-enroll`; the only per-node input is a **single one-time enrollment token**. Two supported OS paths converge on the same enrolled state:

- **Path A — immutable OS (default for partner fleet):** one _generic_ FCOS/bootc qcow2/ISO for the whole fleet, plus a per-node Ignition file reduced to ~20 lines — hostname/node-id, the `/var/mnt/filone` mount, and a oneshot that runs `filone-enroll <token>` on first boot. Keeps atomic updates, rollback, read-only `/usr`.
- **Path B — mutable OS, already-provisioned VM (supported for semi-trusted partners too — see the Threat model section for what this concedes and how it is compensated):** ops runs the same `filone-enroll <token>` on the existing box. The script performs what Ignition would have (data dir, journal-upload config, unit set) and its prereq check bites harder: **systemd + Podman ≥ 5.0 (`Notify=healthy`), hard-fail otherwise** — Debian 13 (5.4.2) passes, Ubuntu LTS archives do not without a backport. Accepted costs, stated so they are chosen and not discovered: no atomic OS rollback or A/B updates; OS patching becomes a mutable-OS process with an explicit owner; bootc bound-images and `/usr`-baked options are unavailable. The immutable OS does security and reliability work on semi-trusted hardware; permitting Path B there is a deliberate threat-model decision, recorded and analysed in the Threat model section below.

Everything downstream of enrollment (§2–§11) is byte-identical on both paths: units are distro-agnostic and all state was already under `/var/mnt/filone`.

**What `filone-enroll` does** (the whole provisioning surface, in one place): prereq check → data directory → `/etc/filone/node-id` → install reconcile script + timer + `filone-bao-bootstrap` oneshot → **redeem the enrollment token against central** (OpenBao response-wrapping: single-use, short-TTL, minted per node) for the real bootstrap set — git read credential, registry pull credential, OpenBao seal credential — → seal them into `systemd-creds` (TPM-backed where available, host-key otherwise) → enable the timer. Nothing app-specific; apps arrive from git on the first tick. A used or expired token fails loudly; an intercepted token is single-use and therefore noisy, not quiet.

The steps:

1. **Central prep (Terraform):** transit seal key + policy + per-node credentials in/around central OpenBao, and mint the **wrapped enrollment token**. The bootstrap set the token redeems into stays host-bound in `systemd-creds` — never in the local OpenBao (self-bricking rule).
2. **Repo prep:** add `nodes/<newnode>/` on `main` via the promotion script targeted at just this node — the node is born at the current verified-in-staging state, not at whatever staging has half-baked.
3. **Enroll:** Path A — boot the generic image with the node's Ignition + token. Path B — run `filone-enroll <token>` on the provisioned box.
4. **Node phones home outbound (first deploy metric, first journal batch); DNS pointed.** No inbound ports, ever, on either path.
5. **First convergence:** fetch `main`, validate, pre-pull, install units, start in dependency order. OpenBao first: unseal round-trip to central — confirm central reachability (and that the enrollment token hasn't expired) before scheduling.
6. **Acceptance:** deploy timestamp fresh; OpenBao unsealed and passing a real transit op; Ingot health exercising transit unwrap; Piri healthy; Caddy serving with first ACME issuance verified. Then announce.

## 2. Upgrade Piri or Ingot image version

Lock-step: one commit bumps both digests, applied from the same SHA, never partially.

1. **Stage:** our CI, on app release, opens a PR bumping both digests in `nodes/staging/` (no bot involved — Piri/Ingot pins are first-party); render check passes; merge.
2. **Staging converges:** reconciler pre-pulls both images (abort before any restart on failure), `pdp-gate wait` (paying Piri's drain, up to 60 min), restart `piri.service` then `ingot.service`, health-gated.
3. **Verify:** staging's reconciler dispatches the verify workflow on successful convergence; the workflow cross-checks the SHA, then runs the smoke suite against staging's public endpoint. Failure stops the world here — prod never sees the change.
4. **Promote:** dispatch the promotion workflow. It re-verifies, then opens the promotion PR copying the digest bumps into all prod dirs (or the chosen canary wave); merge is the go decision.
5. **Prod converges** per node, independently, each paying its own pdp-gate wait — the fleet naturally staggers on proving windows. Watch fleet-wide `deploy_sha` convergence on the dashboard; the deadman covers stragglers.

**Downstream impact:** per node, as rev 1 — Ingot drops in-flight requests (no graceful shutdown; SDK retries absorb most, browser pre-signed URLs don't); Piri's restart is externally invisible but slow. **Coalescing still applies twice over:** batch commits on staging, and batch promotions — promoting three times in an afternoon costs the whole fleet three drains.

## 3. Update Piri or Ingot configuration

Identical pipeline and identical cost to §2 — no SIGHUP reload exists, so `apply: restart` unconditionally; the diff just comes from the config-hash annotation instead of `Image=`. Batch config edits with pending image bumps into one staging merge and one promotion wherever possible.

## 4. Update Caddy config

1. Edit the Caddyfile subtree in `nodes/staging/`; render check updates/validates the hash; merge.
2. Staging reconciler syncs config (directory bind-mount — never a single file), sees the hash diff, runs `systemctl reload caddy`; Caddy validates during load, and an invalid config fails the reload leaving the old config serving.
3. Verify (the smoke suite's TLS + routing checks are the meaningful part here) → promote → prod nodes reload.

**Downstream impact:** zero-downtime everywhere; :443 never closes.

## 5. Upgrade Caddy image version

Caddy stays in the fast tier, but the pipeline now inserts staging between update discovery and the fleet:

1. Dependabot PR against the `deps/` shim; `make render` (CI) propagates the pin into the staging unit; GitHub auto-merge enabled for Caddy PRs — the human gate is removed, the staging gate is not.
2. Staging restarts Caddy (no pdp-gate), health-gated; verify workflow runs.
3. **Promotion is manual, like every other tier** — a prod Caddy restart closes :443 briefly, and that downtime is only ever incurred on a human's decision. Two consequences to manage: (a) for a _security_ bump, patch latency now equals human response time, so add an alert on "staging carries a Caddy digest ahead of prod for > N hours" — the promotion stays a choice, but an unpromoted security patch must not be silent; (b) **socket activation is what would dissolve the objection**: with systemd owning :443, an image swap queues connections in the accept backlog instead of refusing them, turning the restart from downtime into added latency. Verifying that Caddy consumes `LISTEN_FDS` is therefore no longer a nice-to-have — it is the precondition for ever reconsidering faster Caddy promotion.
4. On merge of the promotion PR, prod nodes restart Caddy on their next tick.

**Downstream impact:** ~1–3 s of :443 closed per node; socket activation (systemd owns :443, connections queue across the swap) remains the upgrade if that ever matters — still pending verification that Caddy consumes `LISTEN_FDS`. Post-restart checks include cert validity headroom, since TLS-ALPN-01 renewal failures are silent until expiry.

## 6. Update OpenBao config

**Reloadable stanzas (listener TLS, log level, declarative audit devices):**

1. Edit in `nodes/staging/`; merge; staging reconciler runs `bao-validate`, then `systemctl reload openbao` (SIGHUP).
2. Verify: unsealed _and_ passing a real transit op — the wedged-seal-client class (openbao #2915, gcpckms + file audit) is exactly what a liveness-only check misses. If file audit is enabled, the staging reload **is** the mandated test of SIGHUP-vs-transit-seal before prod ever reloads.
3. Promote; prod nodes reload on their tick.

**Seal/storage stanzas:** need a restart → full §7 treatment. `services.yaml` marks OpenBao `restart: manual`; the promoted diff surfaces as a drift metric per prod node, and a human runs §7 on each. The promotion script warns that this promotion applies nothing by itself.

## 7. Upgrade OpenBao version

Rare, announced, human-gated per node. A restart seals the vault; recovery requires the unseal round-trip to central — a total regional read outage for the duration.

1. **Stage the change:** Dependabot PR against the OpenBao shim entry (required review via CODEOWNERS — no auto-merge on this path), human approves, render propagates into `nodes/staging/`, merge. Staging reconciler applies nothing (manual tier) — a human runs the gated upgrade unit **on staging**, walks the full sequence below, and the verify suite (real transit read) passes. Only then promote the files.
2. **Per prod node, inside an announced window:** pre-flight cheapest-first — central OpenBao reachable and healthy (never overlap a central maintenance window); image pre-pulled; raft snapshot taken; downgrade/storage-format compatibility already checked in release notes _during staging_.
3. Optionally stop Ingot for a clean edge; human triggers the gated upgrade unit (oneshot defined in git — manual trigger, code-defined content).
4. Wait: start → unseal round-trip → healthy, where healthy = successful transit encrypt/decrypt. Alert distinguishes _unseal failure_ (check central) from _unhealthy_ (check locally).
5. Verify Ingot end-to-end reads; close the window; repeat per node — never fleet-parallel, since every one is a full regional read outage.

**Rollback:** revert the digest in staging → verify → promote the revert; each node's human re-triggers the upgrade unit, paying the seal/unseal cost again. If the new version migrated raft storage format, rollback is restore-from-snapshot — which is why compatibility is a staging-time check, not a prod-incident discovery.

## 8. Update Postgres config

1. Edit in `nodes/staging/`; merge; staging runs `pg_ctl reload` — unconditionally safe; `postmaster`-context params simply don't apply and set `pending_restart`.
2. The verify suite includes `SELECT count(*) FROM pg_settings WHERE pending_restart;` — **a nonzero count is now visible at verify time, before promotion**, not just as a standing alert. Promote knowing whether a restart window is owed.
3. Prod nodes reload on their tick; the standing `pending_restart` alert remains the backstop per node.

**Downstream impact:** none for the reload; any owed restart is a §9-style window.

## 9. Upgrade Postgres patch/minor version

1. Dependabot PR against the Postgres shim entry (required review via CODEOWNERS — no auto-merge on this path), human approves, render propagates into `nodes/staging/`, merge. Staging drift metric fires; human triggers the gated upgrade unit on staging: pre-flight (image pre-pulled, base backup verified, WAL current, no long transactions), `systemctl restart postgres.service` from the new digest, health gate, Ingot reconnect verified.
2. Verify suite passes on staging → promote files → each prod node shows drift → human triggers the upgrade unit per node inside a window.
3. Per node: same pre-flight, restart, verify Ingot metadata reads/writes.

**Downstream impact:** seconds-to-a-minute of Ingot metadata errors per node; no cascade (no `Requires=`); optional clean stop of Ingot for zero error noise — pick one policy fleet-wide so partner-visible behaviour is consistent.

**Rollback:** patch/minor formats are compatible in both directions in practice — revert in staging, verify, promote, re-trigger per node. Release notes still checked for the rare on-disk change, at staging time.

## 10. Upgrade Postgres major version

Still the one scenario where git revert is not the rollback: the data directory is rewritten; the revert path is restore-from-backup. The staging flow earns its keep most here — **the staging run is a full dress rehearsal of the migration, including the restore rehearsal.**

1. **Plan at staging time:** release notes, extension compatibility, `pg_upgrade` vs dump/restore (for a single-node metadata DB, dump/restore is often the honest choice and doubles as a backup test).
2. **Stage:** digest + volume-path change (new data dir `/var/mnt/filone/postgres-<ver>`) in one commit to `nodes/staging/`. Human runs the purpose-built oneshot migration unit on staging: stop Ingot, old cluster stopped, migrate into the new data dir, start from the new unit, `vacuumdb --analyze-in-stages`, real-query verification, start Ingot, smoke suite.
3. **Promote the files.** Per prod node, inside a long announced pdp-clear window: fresh base backup restore-rehearsed as part of the window plan; disk headroom for two data dirs; stop Ingot deliberately (a window this long as ride-it-out is an error storm); run the same migration unit; verify; start Ingot only after verification — the point of no return is chosen, not stumbled past.
4. Keep each node's old data dir through an agreed soak; remove via follow-up commit/cleanup.

**Downstream impact:** full Ingot outage per node for the window (tens of minutes, size-dependent). **Rollback:** retained old data dir + backups; writes accepted post-migration are lost on rollback, which is exactly why Ingot stays down until verification passes.

---

## 11. Detecting and troubleshooting a failed deployment

### 11.0 Telemetry without Alloy (2026-08-07)

With Alloy removed there is no agent on the box, and — because inbound connections are forbidden — **no scrape-based collection is possible at all**. Everything must push, outbound:

- **Deploy & alert metrics: curl from the scripts that own the events.** The reconciler pushes `deploy_last_success_timestamp_seconds`, `deploy_sha`, `deploy_failed{sha=…}` and the drift metric at the end of each tick; the `OnFailure=` oneshot pushes its failure counter. Target: a central Pushgateway or a remote-write/import endpoint (VictoriaMetrics `/api/v1/import/prometheus` takes exactly this shape), over HTTPS with the node's client credential. This replaces the textfile-collector pattern entirely — the textfile file no longer has a reader, so drop it.
- **The deadman inverts cleanly.** It was always a staleness alert evaluated centrally; with push, "node stopped pushing" and "deploys stopped succeeding" collapse into the same absent-metric condition. One `absent_over_time`/staleness rule per node covers dead timer, broken egress, and wedged reconciler alike — §11 troubleshooting step 5 tells them apart on the box.
- **Logs: `systemd-journal-upload` → central `systemd-journal-remote`,** stock systemd, HTTPS outbound, client-cert authenticated. Everything already logs to journald (units, containers via the journald log driver, the reconciler), so nothing on the node changes except the shipper. Central keeps journal format or converts into Loki if LogQL is wanted — a central concern, not a node one.
- **What is genuinely lost: host/system metrics.** CPU, memory, disk, network gauges had no pusher other than the agent. The options are honest but unequal: (a) accept the loss and lean on logs + deploy metrics + the smoke suite (defensible for an appliance whose health checks are service-level); (b) a small timer that runs `node_exporter --collector.* --web.disable` once… — no; realistically, a oneshot timer that scrapes a localhost-only node_exporter and pushes the output, which is ~an agent rebuilt from parts; or (c) reintroduce a shipper later under a different name. Decide (a) vs (c) explicitly — the failure mode of not deciding is a disk-full incident with no disk gauge. **Recommendation: (a) plus one exception — a tiny timer pushing disk-usage for `/var/mnt/filone` and the two OpenBao/Postgres state dirs, because disk-full is the one host condition that predictably takes the region down and is trivially cheap to push.**
- **Provisioning (§1) and acceptance change accordingly:** Ignition lays down the journal-upload config and the push credential/URL; acceptance checks become "central sees the node's first deploy metric and first journal batch," not "agent up."

### Detection — four layers now, cheapest first

1. **Restart fails synchronously.** `Notify=healthy` + `HealthCmd` + explicit `TimeoutStartSec` (bug #27290; generous for Piri) makes `systemctl restart` return non-zero; the reconciler sees it immediately.
2. **`OnFailure=` on every unit** → alert oneshot that pushes a failure metric directly to central (§11.0) and leaves a journald record. Catches crashes between deploys too.
3. **Deadman on `deploy_last_success_timestamp_seconds`** per node. Threshold > poll interval + worst-case drain (~poll + 60 min until the drain question is resolved).
4. **The staging verify gate — the new layer.** Bad changes are supposed to die on staging: convergence-then-smoke-tests means a failure class that previously reached the fleet now reaches one internal node. Corollary: **staging's health-check and smoke-suite quality is now the ceiling for the whole pipeline** — a healthy-but-wrong deployment that passes the suite promotes everywhere. Invest here first.

### Automatic response in the reconciler (per node, unchanged)

On failed apply: quarantine the bad SHA (per-node file keyed on commits touching _its_ directory — no flapping), revert working state to last-good and re-apply (pinned digests + unpruned images make this real), emit `deploy_failed{sha=…}`, leave last-success stale so the deadman escalates. Quarantine clears on a new commit touching the node's dir — the fix is always a commit, never a manual poke.

Known limits, unchanged: forward-only DB migrations make app-revert insufficient alone; partial lock-step failure reverts both Piri and Ingot; healthy-but-wrong sails through everything below the smoke suite.

### Failure at each pipeline stage

- **Staging apply fails:** staging quarantines; verify never goes green; **promotion is structurally impossible** (it re-checks staging state at trigger time). Fix forward or revert in `nodes/staging/`; the quarantine clears on the new commit.
- **Verify fails (converged but unhealthy/wrong):** nothing auto-reverts — staging is deliberately left in the failed state for debugging. Revert-in-staging is a human decision and a normal PR.
- **Verify never dispatched (staging never converged, or converged and failed to call out):** the scheduled backstop run picks it up and reports the state; distinguish "reconciler failed" (quarantine file exists → apply failure) from "still draining" (pdp-gate/drain logged distinctly) from "timer dead" (staging deadman). Each has a different owner.
- **Promotion succeeded, one prod node failed:** that node quarantines and self-reverts to last-good; the rest of the fleet is on the new state. This **skew is visible** as `deploy_sha` disagreement on the dashboard and is the signal to decide: node-specific problem (fix the node — hardware, network, stale image cache) or a latent change problem staging didn't catch (revert the promotion commit — one revert restores every prod dir atomically — then fix the smoke suite so it catches this class next time).
- **Promotion partially applied as a commit** can't happen — it's one commit (or one PR); git atomicity does the work.

### Troubleshooting sequence (human, per node — unchanged from rev 1)

1. `journalctl -u reconcile.service -n 200` — which phase (fetch, validate, pre-pull, apply); check the quarantine file for SHA + step.
2. `systemctl status <unit>`; `journalctl -u <unit> --since <deploy>`; `podman logs`. Usual buckets: pull/digest mismatch, invalid config that escaped the render check (fix the check too), health-timing vs actually-broken, dependency down — **whenever Ingot fails, check OpenBao seal status first**; it's the common cause with an uncommon-looking symptom.
3. Confirm last-good: `git rev-parse HEAD` vs quarantined SHA; units active; running digests vs unit digests via `podman inspect`.
4. Fix forward in git — through staging, like everything else. Touch the box only for a wedged reconciler (dead timer, corrupt checkout), and then only to restart the timer or re-clone.
5. Deadman without a deploy: dead timer, expired credential, broken egress, or (Path A only) a bootc OS rollback re-converging old app state — `bootc status` whenever a node looks impossibly old. Path B nodes have no OS rollback; "impossibly old" there means drift or a stale checkout.

### Per-tier wrinkles (unchanged)

- **Piri/Ingot:** log pdp-gate waits distinctly from restart failures so long drains don't cry wolf.
- **OpenBao:** never auto-reverted or auto-restarted; _sealed_ (→ central) alerts separately from _unhealthy_ (→ local). The manual tier exists precisely so no automation converts a config mistake into a read outage.
- **Postgres:** manual-tier logic; failed manual upgrades fall back to retained data dir / backup, not the reconciler.
