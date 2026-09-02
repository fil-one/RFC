# RFC: Fil One Appliance Deployment Strategy

**Status:** Proposal

## Authors

- [Miroslav Bajtoš](https://github.com/bajtos)

## Motivation

Fil One Appliance is a set of services operating Fil One node on infrastructure provided by a regional
provider. Two main goals are shortening time needed to set up a new region and guaranteeing high
operational rigour even if the regional provider does not have the necessary skills or experience.

Our selling point to regional providers: you provide hard drives and a VM, we bring our software and
customers.

This document proposes how to deploy and operate the appliance.

## Requirements & Constraints

1. Setting up a new region must not require any engineering or advanced system administration on the
   side of the region provider/operator.
2. Fil One must be able to deploy security and bug fixes in a timely manner (hours, not days).
3. Fil One must have visibility into operational metrics and logs.
4. Upgrades must cause as short downtime as possible. We should aim for zero-downtime upgrades.
5. Upgrades must honour operator obligations on chain. E.g., we cannot upgrade in the window where
   the Piri node is required to submit a PDP proof.
6. The deployment should be managed using infrastructure-as-code.
7. We must pin versions of all services, so that we always deploy a combination of
   versions that was tested in dev and is known to work together correctly.
8. The process for managing the dev environment must match the process for managing the
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
1. Platform
1. Apps

It is not yet clear who will operate the OS, Platform and Apps layers - Fil One or the region operator.

### Hardware

This is the bedrock on which we build the rest of the stack.

1. The machine
2. Network connectivity: a public IP under a stable domain name, and open port 443
3. Storage - control plane: a persisted volume mounted as local FS in the machine
4. Storage - data plane: an S3-compatible object storage, in the same datacenter

### Operating System

A minimal Linux-based distro with a container runtime (Docker + Compose) and systemd.

Updated infrequently, primarily to apply bugfixes and security patches.

Easy to recreate, disposable.

### Platform Services

- A Postgres-compatible database
- A secure secret manager (OpenBao, unsealed using Fil One's central OpenBao instance)
- Caddy (TLS termination, cert management)
- Filecoin RPC API node (Lotus). Can be initially replaced with an external provider like chain.love.

Updated infrequently, primarily to apply bugfixes and security patches.

Backed up to an external location (not the control-plane volume).

### Apps Services

- Piri
- Ingot

Updated frequently to ship new features.

## Proposal

1. Two Docker Compose projects - one for "platform" services, the other for "apps" services
2. Config files and pinned image versions tracked in git.
3. systemd-timer with git-pull script to reconcile.
4. Rollback is implemented as `git revert` to the previous set of configs & image versions and another deploy.
5. We can run on any Linux distro with Docker Compose & systemd.
6. Continuous deployment for the dev instance.
7. Manual promotion from dev to other nodes, with changes applied automatically from definitions tracked in git.

## MVP

For the next few weeks to months, we prioritize time-to-market over robustness and scalability.

### Hardware

Deploy the dev node to a single AWS EC2 instance, to mimic a single VM running in the region datacenter.

Manage the AWS infrastructure using OpenTofu; store the state in S3; apply updates via a GitHub Actions workflow.

### OS

The dev node will use vanilla Ubuntu Server offered by AWS. We will upgrade the OS manually and very infrequently.

### Platform & Apps

Docker image versions will be pinned to a specific SHA.

Dev environment: Continuous deployment - every commit landed in one of Forge repositories will be
deployed to the appliance in about 5 minutes. Longer deployment time when we need to wait for Piri
to finish submitting the PDP proof.

Non-dev environments: We will upgrade the components in a scheduled maintenance window. The upgrade
is performed by promoting a set of IaaC definitions that are known to work well in dev. The actual
deployment will be handled by the automated reconciler script running on the appliance host.

Every deployment will perform a full restart of affected services. Changes in Platform
will stop all Apps before upgrading Platform, and bring Apps up after Platform is healthy again.

### Implementation details

We will create a new GitHub repository `fil-forge/infra-node` with the following layout:

```sh
# Configuration of appliance instances:
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
systemd/
  # definition of timers shared by all nodes

# AWS infrastructure (our nodes running on AWS)
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

## Updating & upgrading appliances

### Nodes we operate

Continuous deployments for the dev instance we operate ourselves:

1. A pull request modifies pinned image versions and/or config versions for the dev node
2. The pull request is merged
3. The dev node pulls the new commit (timer-based job) and reconciles the services
4. A GHA verification workflow triggered by the merged pull request actively polls node's state, waits until the changes are deployed
5. The workflow performs automated end-to-end smoke tests
   - create a new tenant, create a new access key, create a new bucket, upload/download object, etc.
6. If the tests pass, the workflow creates a new pull request to update the per-region infra
   definition files. If there is an already open pull request for the same region, the workflow
   updates it.

To upgrade the non-dev appliance:

1. A developer merges a PR changing a non-dev node configuration
2. The node pulls the new commit and reconciles the services
3. The GHA verification workflow triggered by the merged pull request actively polls node's state,
   waits until the changes are deployed, and runs non-destructive end-to-end tests

### Updating appliances operated by region providers

We need a slightly different approach in case the appliance services are operated by the region provider.

Straw-man proposal 1:

1. We add a new directory with image versions & base config files for provider-operated nodes, treat it as a new virtual region.
1. After the dev deployment passed the tests, the workflow opens a pull request to update that virtual region.
1. It's up to the region operator to decide when they want to pull the changes & apply them. They can use the same reconciler script as we use, just not install the systemd timer.

Straw-man proposal 2:

1. Provider-operated nodes don't use git-based IaaC, they use Docker tags instead.
2. After the dev deployment passed the tests, if Piri or Ingot has a new image version, the
   workflow creates a new Docker tag for both Piri & Ingot images, sharing the same version number.
3. It's up to the appliance operator to watch new Docker image versions and apply the updates, e.g.
   using podman auto-update.
4. It's up to the appliance operator to manage their config files and apply any necessary changes.

Straw-man proposal 3:

1. Distribute the two Apps (Ingot and Piri) as a single Linux distro package (apt for Debian/Ubuntu, rpm for RHEL/Fedora).
2. After the dev deployment passed the tests, we publish a new version of the appliance package.
3. It's up to the appliance operator to apply updates and manage the platform dependencies.

## Post-MVP Hardening

### No secrets in Ingot & Piri config files

Improve Piri & Ingot to support alternate options for loading secrets, e.g. env vars or OpenBao.

### Graceful shutdown in Ingot

Ingot needs to implement graceful shutdowns. When the shutdown is initiated, it will stop accepting
new requests, but keep running until all existing in-flight requests are handled to completion or
time out.

### Zero-downtime Ingot upgrades

Audit for where Ingot is storing state that isn't fundamentally multiprocess compatible, this is
likely a very few places. Rework these areas to support multiple Ingot processes running
concurrently, and then move quickly to blue-green deployments via Caddy as a load balancer.

### Zero-downtime Piri upgrades

Perhaps we can separate the API interface of Piri which is near stateless from the task scheduling
parts. This may not be as hard as we think as I think the Harmony task scheduler is pretty
multiprocess friendly.

Once cleared up, move to blue-green deployments via Caddy as a load balancer.

### Zero-downtime Caddy upgrades

**Config changes**

Caddy is able to pick up config changes without the need to restart. We should improve the
reconciler script to detect config-only changes and apply them via SIGHUP instead of restarting Caddy.

**Version upgrades**

Caddy's upgrades are deployed to prod regions manually. As a result, we can easily end up running
Caddy in production with known security vulnerabilities for weeks.

We should add an alert on "dev carries a Caddy digest ahead of prod for > N hours". This way we can
keep the upgrades a manual process scheduled for maintenance windows, but still be alerted when a
security patch was not applied fast enough.

In the longer term, we should investigate how to configure the host to allow zero-downtime upgrades
of Caddy: let systemd listen on `:443`, so that a Caddy image swap queues connections in systemd's
accept backlog instead of refusing them, turning the restart from downtime into added latency. This
requires verification whether Caddy consumes `LISTEN_FDS`.

### More robust OpenBao upgrades

Upgrading OpenBao requires restart that seals the vault requires the unseal round-trip to central —
a total regional read outage for the duration. That's one more reason why OpenBao's upgrades must be
manual and carefully planned.

We should implement a similar alert as we have for Caddy: "dev carries an OpenBao digest ahead of prod for > N hours".

When the reconciler is performing the upgrade, it should start with the cheap pre-flight checks:

1. Central OpenBao reachable and healthy (never overlap a central maintenance window);
2. Image pre-pulled;
3. Raft snapshot taken.

Upgrades must be rolled per node — never fleet-parallel, since every one is a full regional read outage.

### More robust Postgres upgrades

**Config changes**

Postgres is able to pick up most configuration changes via SIGHUP without restart . We should
improve the reconciler script to detect such changes and apply them without restarting the server.

See https://www.heatware.net/postgresql/reload-config-with-pg-ctl/

**Patch/minor version upgrades**

Implement a metric & alert to let us know when a node is not up to date.

The reconciler should run pre-flight checks before deploying the upgrade:

1. image pre-pulled,
2. base backup verified,
3. WAL current,
4. no long transactions.

After the upgrade & restart, it should run the following checks:

1. PG server is healthy
2. Piri & Ingot are connected

Rollback:

Patch/minor formats are compatible in both directions in practice — revert in dev, verify,
promote, re-trigger per node. Release notes still checked for the rare on-disk change, at dev
time.

### Postgres major version upgrades

The one scenario where git revert is not the rollback: the data directory is rewritten; the
revert path is restore-from-backup.

**The dev run is a full dress rehearsal of the migration, including the restore rehearsal.**

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

### Adopt an immutable OS

Why an immutable OS:

- Prevents a sloppy operator from breaking the box
- Prevents [snowflake servers](https://martinfowler.com/bliki/SnowflakeServer.html)
- Unattended security updates
- A botched OS update is automatically reverted

Options to explore: Fedore CoreOS or bootc; Podman + Quadlet instead of Docker Compose.

### More comprehensive testing

We should implement additional tests to verify that the appliance works correctly after changes were deployed:

- End to end tests. E.g. create tenant, create bucket, upload object, download object
- Soak tests

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

- Distro version sensitivity. Ubuntu LTS ships Podman too old for Notify=healthy.
- The reconciler would need a custom diff/restart logic.
- Weaker newcomer ergonomics and smaller ecosystem. "One YAML, docker compose up -d" is a friendlier mental model. We already use Docker Compose in Smelt.
- Implementing zero-downtime upgrades requires more manual work with Quadlet than with Docker Compose.
- The biggest benefit - first-class support on FOCS/bootc - it not relevant, since we decided to use mutable OS in MVP for simplicity.

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

**apt/rpm packages**

- This is similar to Podman+Quadlet in the sense that we don't have full control over which version is installed at what time.
- To manage the node via IaaC & git, we would need to implement a manual process for installing the particular version of each service.
- Running two versions of the same service is not trivial, the path to zero-downtime blue/green deployments is not clear.

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
