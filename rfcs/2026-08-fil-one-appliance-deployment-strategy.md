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
6. The deployment should be realised as immutable infrastructure fully driven by code
   (infrastructure-as-code).
7. We must pin versions of all services and dependencies, so that we always deploy a combination of
   versions that was tested in staging and is known to work together correctly.

The following services does not support more than one instance running concurrently, therefore upgrades must be implemented as in-place restarts:

- Ingot
- Piri
- Postgres
- Vault
- Caddy

Zero-downtime upgrades are not possible now, this remains an aspirational future goal.

## Components & Dependencies

There are four layers in our stack:

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

A Linux-based OS with a container runtime (Docker or Podman).

Updated infrequently, mostly to apply bugfixes and security patches.

Easy to recreate, disposable.

### Infra Services

- A Postgres-compatible database, with point-in-time backup & recovery
- A secure secret manager (Vault)
- Caddy (TLS termination, cert management)

Updated infrequently, mostly to apply bugfixes and security patches.

Backed up to an external location (not the control-plane volume).

### Forge Services

- Piri
- Ingot

Updated frequently to ship new features.

### External Dependencies

- Filecoin JSON RPC API like chain.love

## Hypothesis / Goals

TBD

## Design

TBD

## Alternatives Considered

TBD

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
