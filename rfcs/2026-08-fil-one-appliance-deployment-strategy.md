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

## Requirements

1. Setting up a new region must not require any engineering or advanced system administration on the
   side of the region provider/operator.
2. FilOne must be able to deploy security and bug fixes in a timely manner (hours, not days).
3. FilOne must have visibility into operational metrics and logs.
4. Upgrades must cause as short downtime as possible. We should aim for zero-downtime upgrades.
5. Upgrades must honours timing constraints, e.g. we cannot upgrade in the window where the Piri
   node is required to submit a PDP proof.
6. The deployment must be realised as immutable infrastructure fully driven by code
   (infrastructure-as-code).
7. We must pin versions of all services and dependencies, so that we always deploy a combination of
   versions that was tested in staging and is known to work together correctly.

## Components & Dependencies

Operated by us:

- Piri
- Ingot
- Postgres-compatible database
- Secure secret manager (Vault)
- Caddy (TLS termination, cert management)
- Grafana Alloy (ships logs & telemetry to our Grafana)

Provided by operator:

- A virtual machine we can run our stack on, with a public IP and open 443
- A persisted volume mounted to the VM for storing control-plane data
- An S3-compatible object storage for storing user data

External dependencies:

- Filecoin JSON RPC API like chain.love

## Hypothesis / Goals

TBD

## Design

TBD

## Alternatives Considered

TBD

## Open Questions

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

### Can we give up on zero-downtime service upgrades?

Podman auto-update restarts a unit in place — that is a stop-start, not a cutover. Getting true
zero-downtime service upgrades under Quadlet needs a proxy unit plus a blue/green unit pair plus a flip
script: modest custom work.

The real SLO is "outage shorter than client retry tolerance." S3 SDK clients retry with backoff,
PUTs are idempotent, multipart uploads are resumable. A sub-minute reboot is largely invisible to
well-behaved clients.

However, the above is not true for pre-signed URLs loaded by browsers, they don't implement any
retry.

---

---

# Claude Research Report

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
