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
8. The process for managing the staging environment must match the process for managing the
   production environment. Only the configuration and image versions should vary.

The following services do not support more than one instance running concurrently, therefore upgrades must be implemented as in-place restarts:

- Ingot
- Piri
- Postgres
- Vault/OpenBao
- Caddy

Zero-downtime upgrades are not possible now, this remains an aspirational future goal.

## Components & Dependencies

Our stack can be organised into the following layers:

1. Hardware
1. Operating System
1. Platform Services
1. Forge Services

It is not yet clear who will operate which layer - FilOne or the region operator.

### Hardware

This is the bedrock on which we build the rest of the stack.

1. The machine
2. Network connectivity: a public IP under a stable domain name, and open port 443
3. Storage - control plane: a persisted volume mounted as local FS in the machine
4. Storage - data plane: an S3-compatible object storage, in the same datacenter, accessible via S3 (HTTPS)

### Operating System

A minimal Linux-based distro with a container runtime (Podman) and systemd.

Updated infrequently, primarily to apply bugfixes and security patches.

Easy to recreate, disposable.

### Platform Services

- A Postgres-compatible database
- A secure secret manager (OpenBao, unsealed using FilOne's central OpenBao instance)
- Caddy (TLS termination, cert management)
- Filecoin RPC API node (Lotus, Forest). Can be initially replaced with an external provider like chain.love.

Updated infrequently, primarily to apply bugfixes and security patches.

Backed up to an external location (not the control-plane volume).

### Apps Services

- Piri
- Ingot

Updated frequently to ship new features.

## Proposal

1. Two Docker Compose projects - one for "platform" services, the other for "apps" services
1. Config files and pinned image versions tracked in git.
1. systemd-timer with git-pull script to reconcile.
1. Rollback is implemented as `git revert` to the previous set of configs & image versions and another deploy.
1. We can run on any Linux distro with Docker Compose & systemd.
1. Post-MVP: If we own the OS, then let's use an immutable image-based OS like Fedora CoreOS or bootc with Podman+Quadlet replacing Docker Compose.
1. Continuous deployment for the dev instance, manual deployments for everything else.

### Why the immutable OS

- Prevents a sloppy operator from breaking the box
- Prevents [snowflake servers](https://martinfowler.com/bliki/SnowflakeServer.html)
- Unattended security updates
- A botched OS update is automatically reverted

## MVP

For the next few weeks to months, we prioritize time-to-market over robustness and scalability.

### OS

We will upgrade the OS manually in a scheduled maintenance window, by following Runbook instructions.

### Platform & Apps

Docker image versions will be pinned to a specific SHA.

Dev environment: Continuous deployment - every commit landed in one of Forge repositories will be
deployed to the appliance in less than 10 minutes, unless we need to wait for Piri to finish
submitting the proof.

Non-dev environments: We will upgrade the components in a scheduled maintenance window. The upgrade
is performed by promoting a set of IaaC definitions that are known to work well in dev. The actual
deployment will be handled by the automated reconciler script running on the appliance host.

A longer downtime of 30-60 minutes is acceptable.

See the post-MVP sections for more details, we will implement a trimmed-down process for MVP:

- [Update Postgres config](#update-postgres-config)
- [Upgrade Postgres patch/minor version](#upgrade-postgres-patchminor-version)
- [Update Caddy config](#update-caddy-config)
- [Upgrade Caddy image version](#upgrade-caddy-image-version)
- [Update OpenBao config](#update-openbao-config)
- [Upgrade OpenBao version](#upgrade-openbao-version)

**Downstream impact**

- Piri's restart is externally invisible but slow.
- Ingot drops in-flight requests (no graceful shutdown; SDK retries absorb most, browser pre-signed URLs don't)

**TODOs**

- Rework Piri & Ingot config schemes so that secrets are stored in external files or Vault/OpenBao.
- Improve Ingot to support graceful shutdowns
- Investigate if Caddy can queue connections while Ingot is restarting

### Implementation details

We will create a new GitHub repository `fil-forge/infra-node` with the following layout:

```sh
# Docker Compose manifests, templates and env files
nodes/
  dev/
    apps/
      compose.yml
      # etc.
    platform/
      compose.yml
      # etc.
    node.env
  production/
    # same structure as above
  staging/
    # same structure as above
terraform/
  envs/
    # AWS resources shared per AWS account:
    # S3 bucket for OpenTofu state
    bootstrap/
      nonprod/
      production/
    # EC2 node & supporting AWS infra
    # for the "dev" stage
    dev/
  modules/
    # OpenTofu definitions shared by more than one env
systemd/
  # definition of timers
scripts/
  # scripts executed by GitHub Actions workflows
  ci/
    smoke-test.sh
    # etc.
  # scripts executed on the node
  host/
    pdp-gate.sh
    reconcile.sh
    # etc.
  # scripts executed on the operator (dev) machine
  operator/
    ssm-session.sh
    # etc.
```

### Provision a new region

1. If the region is using AWS infra, then bring up the EC2 & related services using `tofu`.
2. Obtain the IP address of the machine, e.g. from Tofu outputs if using AWS.
3. Ask infra-central operators to mint OpenBao unseal token bound to machine's IP address
4. Once you have the token, run `scripts/host/provision-platform.sh` on the machine.
5. Next, run `scripts/host/onboarding-request.sh`. It will print the information you need to send to
   infra-central operators to register the node (Piri's DID, wallet address, delegations, etc.)
6. You will get back the delegation proof needed by Ingot, store it via `scripts/host/store-hilt-proof.sh`
7. Now you can provision the apps by running `scripts/host/provision-apps.sh`
8. Finally, enable the systemd timers to periodically pull changes from git and refresh OpenBao tokens

A more detailed runbook will live in the infra-nodes repository, see
[docs/RUNBOOK.md](https://github.com/fil-forge/infra-nodes/blob/main/docs/RUNBOOK.md). There is also
an infra-central counter-part in
[docs/appliance-onboarding.md](https://github.com/fil-forge/infra-central/blob/main/docs/appliance-onboarding.md).

## Continous deployment

For the dev instance we operate ourselves:

1. A pull request modifies pinned image versions and/or config versions for the dev node
2. The pull request is merged
3. The dev node pulls the new commit (timer-based job) and reconciles the services
4. A GHA verification workflow triggered by the merged pull request actively polls node's state, waits until the changes are deployed
5. The workflow performs automated end-to-end smoke tests
   - create a new tenant, create a new access key, create a new bucket, upload/download object, etc.
6. If the tests pass, the workflow creates a new pull request to update the per-region infra
   definition files. If there is an already open pull request for the same region, the workflow
   updates it.
7. A developer merges the PR for the staging/production region
8. The staging/production node pulls the new commit and reconciles the services
9. The GHA verification workflow triggered by the merged pull request actively polls node's state,
   waits until the changes are deployed, and runs non-destructive end-to-end tests

### Provider operates the appliance

We need a slightly different approach in case the appliance services are operated by the region provider.

Straw-man proposal 1:

1. We add a new directory with image versions & base config files for provider-operated nodes, treat it as a new virtual region.
1. After the staging deployment passed the tests, the workflow opens a pull request to update that virtual region.
1. It's up to the region operator to decide when they want to pull the changes & apply them

Straw-man proposal 2:

1. Provider-operated nodes don't use git-based IaaC, they use Docker tags instead.
2. After the staging deployment passed the tests, if Piri or Ingot has a new image version, the workflow creates a new Docker tag for both Piri & Ingot images, sharing the same version number.
3. It's up to the appliance operator to watch new Docker image versions and apply the updates, e.g.
   using podman auto-update.
4. It's up to the appliance operator to manage their config files and apply any necessary changes.

## Post-MVP Hardening

### Caddy Config Changes

Caddy is able to pick up config changes without the need to restart. We should improve the
reconciler script to detect config-only changes and don't restart Caddy.

### Caddy Upgrades

Caddy's upgrades are deployed to prod regions manually. As a result, we can easily end up running
Caddy in production with known security vulnerabilities for weeks.

We should add an alert on "dev carries a Caddy digest ahead of prod for > N hours". This way we can
keep the upgrades a manual process scheduled for maintenance windows, but still be alerted when a
security patch was not applied fast enough.

In the longer term, we should investigate how to configure the host to allow zero-downtime upgrades
of Caddy: let systemd listen on `:443`, so that a Caddy image swap queues connections in systemd's
accept backlog instead of refusing them, turning the restart from downtime into added latency. This
requires verification whether Caddy consumes `LISTEN_FDS`.

### OpenBao Upgrades

Upgrading OpenBao requires restart that seals the vault requires the unseal round-trip to central —
a total regional read outage for the duration. That's one more reason why OpenBao's upgrades must be
manual and carefully planned.

We should implement a similar alert as we have for Caddy: "dev carries an OpenBao digest ahead of prod for > N hours".

When the reconciler is performing the upgrade, it should start with the cheap pre-flight checks:

1. Central OpenBao reachable and healthy (never overlap a central maintenance window);
2. Image pre-pulled;
3. Raft snapshot taken.

Upgrades must be rolled per node — never fleet-parallel, since every one is a full regional read outage.

### Postgres Config Changes

Postgres is able to pick up most configuration changes without restart. We should improve the
reconciler script to detect such changes and apply them without restarting the server.

See https://www.heatware.net/postgresql/reload-config-with-pg-ctl/

### Postgres Upgrades

Implement a metric & alert to let us know when a node is not up to date.

**Patch/minor versions**

The reconciler should run pre-flight checks before deploying the upgrade:

1. image pre-pulled,
2. base backup verified,
3. WAL current,
4. no long transactions.

After the upgrade & restart, it should run the following checks:

1. PG server is healthy
2. Ingot is connected

**Rollback**

Patch/minor formats are compatible in both directions in practice — revert in staging, verify,
promote, re-trigger per node. Release notes still checked for the rare on-disk change, at staging
time.

**Major versions**

The one scenario where git revert is not the rollback: the data directory is rewritten; the
revert path is restore-from-backup.

**The staging run is a full dress rehearsal of the migration, including the restore rehearsal.**

1. **Plan before upgrading the dev node:** release notes, extension compatibility, `pg_upgrade` vs dump/restore
   (for a single-node metadata DB, dump/restore is often the honest choice and doubles as a backup
   test).

2. **Manual upgrade on the dev node:**

   1. Stop platform & apps
   2. Migrate PG into the new data dir (`/var/mnt/filone/postgres-<ver>`)
   3. Start from the new unit
   4. `vacuumdb --analyze-in-stages`
   5. Real-query verification
   6. Start Ingot
   7. Smoke suite
   8. Commit the new image digest & volume-path change to git

3. **Manual upgrade per production node:**

Inside a window when Piri does not need to post PDP proofs:

1.  Rehearse the DB restore from a fresh backup as part of the window plan
2.  Disk headroom for two data dirs
3.  Stop platform & apps
4.  Run the same migration as in dev
5.  Verify;
6.  Start Piri & Ingot only after verification.

Keep each node's old data dir through an agreed soak test period; remove via follow-up commit/cleanup.

**Downstream impact:**

Full Piri & Ingot outage per node for the window (tens of minutes, size-dependent).

**Rollback**

Retained old data dir + backups. Writes accepted post-migration are lost on rollback, which is exactly why Ingot & Piri stays down until verification passes.
ollback re-converging against an old app state: check `bootc status` for a rollback whenever node behaviour looks "impossibly old."

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

**Immutable OS**

- Too complex for our initial deployments.
- Not offered in AWS EC2 standard images

**Podman + Quadlet**

- Adding another tool, we already use Docker Compose in Smelt
- Popular distros like vanilla Ubuntu on AWS ship an older version missing some of the features we need
- The biggest benefit - first-class support on FOCS/bootc - it not relevant, since we decided to use mutable OS for simplicity

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

## References

- https://www.thelinuxvault.net/blog/how-to-run-podman-containers-under-systemd-with-quadlet/
- Decisions made during infra-central implementation: https://github.com/fil-forge/infra-central/blob/main/docs/decisions/2026-08-region-onboarding.md
- Decisions made during infra-nodes implementation: https://github.com/fil-forge/infra-nodes/blob/main/docs/decisions/2026-08-initial-design.md
