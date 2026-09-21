# RFC: Regional security principles and key management deployment proposal

**Status:** Design
**Author:** Hannah Howard
**Audience:** Fil One + Forge engineers
**Date:** 2026-08-06

---

## TL;DR

A region can defend exactly two security properties, and we should commit to both: **nothing on the appliance's disks is readable at rest, and a revoked appliance never comes back up.** Both are gated on discovery. Against an operator with sustained, undiscovered access to the running system, no design protects the data: the appliance decrypts objects to serve them, and the operator owns the hardware it runs on.

The deployment that delivers the two winnable properties is a local OpenBao on each appliance, auto-unsealed by a central OpenBao. It holds the Region KEK as a non-exportable key, wraps each Blob CEK with transit AES-GCM (amending [the encryption RFC](./2026-06-fil-one-encryption-design.md)'s A256KW region wrap), and becomes the home for every secret the appliance currently keeps in plain files. Measured against OpenBao 2.6.1, a wrap costs ~0.1 ms and sustains 40,000–70,000 operations per second on a 16-core machine (~19,000 constrained to two cores), well above the ceilings that have motivated more complex designs. Published to get holes poked in it before we write the code.

## Motivation

Three documents currently touch regional key management, each from a different angle and none from shared principles. [The encryption RFC](./2026-06-fil-one-encryption-design.md) assigns the region wrap to "secure storage" in a footnote and calls the choice "a policy knob we can tweak." [FIL-547](https://linear.app/filecoin-foundation/issue/FIL-547) implements that knob as in-process A256KW: Ingot imports the Region KEK into memory that our own code locks and zeroes. The [appliance deployment strategy RFC (PR #19)](https://github.com/fil-one/RFC/pull/19), still in draft, poses where the Region KEK should live as an open question, with appended analysis exploring an intermediate-KEK layer motivated by secrets-manager throughput. These decisions interlock. Getting them wrong is expensive in both directions: too little care and a stolen disk reads plaintext; too much ambition and we ship security machinery we cannot maintain.

The appliance's disk is already the problem. The provider wallet key that adds pieces to PDP is plaintext bytes in a LevelDB at `<data-dir>/wallet`; Piri's Ed25519 service identity is an unencrypted PEM; the setup installer writes S3 credentials into a world-readable `piri-config.toml`; Ingot's config carries the Postgres DSN and the S3 root credentials in plain YAML. TLS keys join this list as regions terminate their own hostnames. Whatever we decide about the Region KEK is also the decision about where all of this lives.

## Principles

**1. Two properties are winnable at the region, and we build for both.** A powered-off disk, imaged after theft, RMA, or decommissioning, yields no object plaintext, no key material, and no credential that still works. And a region we have decided to cut off does not come back up: revoking it centrally prevents the appliance from starting. Every design decision below serves these two properties.

**2. Discovery is the caveat on both.** The revocation lever helps once we know to pull it. A malicious operator or a stolen machine that we discover is contained from the next boot. Discovered too late, there is very little left to do: the system has been serving that adversary plaintext all along. This is a limit of the problem itself, and no design spend moves it. Remote kill of a *running* appliance (a central-triggered seal) would narrow the window between discovery and containment; it is worth looking at and is lower priority than getting the at-rest and startup properties right.

**3. The running system has no defense against its own operator.** The appliance holds plaintext CEKs and object bytes in memory whenever it serves a read, and the operator controls the hypervisor, the kernel, and the network path. PR #19's threat model states this plainly: "The region operator has hypervisor access and can read the SSH host key and any authorized-keys material." Neither of the obvious conclusions follows. Reaching for exotic protection against this adversary (trusted execution environments as a requirement) spends complexity on a property we cannot have. Concluding that at-rest custody is pointless because memory is readable anyway gives away the two properties we can have.

**4. No novel security moves.** We are a six-engineer team shipping a storage product. Every security-critical mechanism we deploy should be a stock, maintained feature of a dedicated security project, because we will not out-engineer the failure modes specialists have already hit. Memory locking is the cautionary case. OpenBao removed the mlock implementation it inherited from Vault, judging it buggy ([openbao#354](https://github.com/openbao/openbao/issues/354), [openbao#363](https://github.com/openbao/openbao/pull/363)), and moved memory protection to host configuration. FIL-547 hand-builds page-locked allocation in our codebase, and the locked buffer cannot even follow the key it guards: `crypto/aes` expands the key schedule into ordinary garbage-collected heap the moment a wrap runs. When the dedicated project deletes a mechanism as too subtle to get right, we should not be writing our own copy of it.

## Design

### One home for secrets: a local OpenBao, rooted centrally

Each appliance runs OpenBao, listening on a unix socket, with raft storage on the appliance disk. It is the single home for regional secrets: the Region KEK (a non-exportable transit key), the provider wallet key, TLS leaf keys, Postgres credentials, service identity PEMs, and the S3 credentials the installer currently writes into config files. Nothing secret sits in a plain file.

The local OpenBao's storage is sealed by a transit key held at a central OpenBao (`seal "transit"`). At boot the appliance authenticates to central, unwraps its barrier key, and unseals. With central unreachable, or the seal credential revoked, OpenBao 2.6.1 refuses to start at all (verified behavior). This is the startup-kill lever from Principle 1: revoking one credential at central makes everything on the disk permanently unreadable. The boot credential is CIDR-bound to the region's egress and single-use where the deployment permits, so an imaged disk replayed elsewhere fails, and any use of a stolen credential is visible at central. Hardware binding of the boot identity (TPM sealing at enrollment) is the eventual strengthening; it needs host-image work and is out of scope here. The boot-time dependency on central has the same failure envelope the appliance already accepts for auth, where Hilt-derived credentials expire daily; steady-state reads never call central, preserving the encryption RFC's Independence criterion.

Two wallets participate in PDP, with different custody. The payer wallet (Fil One's, the funded one) lives centrally behind `piri-signing-service`: the appliance's Piri is a consumer of that service, which completes each PDP call so it validates, and the payer key never touches the box (PR #19 stresses keeping it that way, and we agree). The signing service's own keystore belongs in the central OpenBao rather than in key files on its host. The provider wallet, which signs pieces into PDP, does live on the appliance: in the plaintext LevelDB above today, in the local OpenBao under this design. Central credential patterns do not cover the region wrap, which needs a local unwrap path that runs at petabyte scale.

### The region wrap: transit AES-GCM, context-bound

We amend the encryption RFC's region wrap. Region-KEK(Blob-CEK) becomes a transit `encrypt` call against the local OpenBao: key type `aes256-gcm96`, created with `derived=true`, context bound to (space, blob digest). Ingot mints the CEK itself (one constructor over `crypto/rand`; nothing else creates or accepts raw key bytes), wraps it through transit, and stores the ciphertext and key version in the [FIL-480](https://linear.app/filecoin-foundation/issue/FIL-480) columns on `blob_locations`, which need no schema change beyond a format tag on the wrapped bytes. GET unwraps with the same context. Ingot's token permits encrypt, decrypt, and rewrap; export of the KEK is impossible (non-exportable is the transit default). The tenant wrap is untouched: ECDH-ES+A256KW to the tenant key runs under a per-operation ephemeral secret, so it holds no long-lived key material in Ingot and none of the concerns below apply to it.

AES-GCM is the correct algorithm for this job, and the original A256KW choice was an error. NIST SP 800-38F, the AES-KW specification itself, states: "there is no requirement to protect cryptographic keys with a distinct cryptographic method. Previously approved authenticated-encryption modes … are approved for the protection of cryptographic keys, in addition to general data" (§3.1), and FIPS 140-3 Implementation Guidance D.G names AES-GCM as an approved key-wrapping method. Deployed practice matches: AWS KMS wraps every data key with AES-256-GCM and binds the encryption context as associated data, GCP Cloud KMS is AES-256-GCM, and Vault's own purpose-built envelope endpoint (`transit/datakey`) wraps data keys the same way. The encryption RFC chose AES for "compatibility with key management systems (especially hardware security modules)," and the compatibility argument runs the other way for software: no software KMS (Vault, OpenBao, AWS, GCP) exposes A256KW as a callable operation. A256KW is native to HSMs, which is exactly where it returns if we later put a PKCS#11 provider behind the same interface. FIL-547's memory-protection machinery exists only because A256KW forced the KEK into Ingot's process; with the wrap inside OpenBao, that machinery is deleted. FIL-547's `Provider` interface survives with the binding context added to its signature.

The switch also adds a capability A256KW cannot offer. Context binding means a wrapped CEK authenticates only against its own (space, blob digest): transplanting wrap material between rows fails outright (verified: wrong context yields an authentication failure). And because `derived=true` gives each context its own subkey, no single GCM key accrues invocations toward NIST's per-key bound no matter how many billions of objects a region holds. Binding rides `context` rather than `associated_data` deliberately: OpenBao's rewrap endpoint honors context and rejects AAD-bound ciphertexts, and rewrap is how rotation stays clean — rotate the key, then batched `rewrap` moves stored ciphertexts from `vault:v1:` to `vault:v2:` entirely inside OpenBao, with no CEK ever appearing in any process of ours (verified end to end). Old versions keep decrypting until `min_decryption_version` advances, so rewrap campaigns can be lazy.

GCM fails badly under nonce reuse. The realistic trigger is a cloned VM image or an entropy-starved first boot rather than request volume. Context derivation already reduces any one subkey to a handful of invocations; the remaining discipline — seeded entropy at first boot, no snapshot reuse of a running appliance — goes on the host checklist below.

### Host hardening replaces process-level memory protection

Following OpenBao's own post-mlock guidance: swap disabled (`memory.swap.max=0` in the unit's cgroup) or encrypted, core dumps off, seeded entropy at first boot, no snapshot/clone reuse. One host checklist protects OpenBao's process and Ingot's alike. The irreducible key material in Ingot is one object's CEK for the duration of its own request, which is the blast radius we accept for being the process that serves plaintext. The dead-disk property carries one wiring dependency alongside this checklist: the PUT pipeline must encrypt bytes before they rest in the spool (the [FIL-481](https://linear.app/filecoin-foundation/issue/FIL-481)/[FIL-482](https://linear.app/filecoin-foundation/issue/FIL-482) ordering), or the spool must live on a bound volume.

## Performance

A wrap costs about 0.1 ms, and even a two-core configuration sustains ~19,000 per second; the secrets manager is not the throughput limit. Method: OpenBao 2.6.1, transit `aes256-gcm96` (plain and derived-context variants), 32-byte plaintexts, unix-socket and loopback listeners, persistent connections, on a 16-core Apple M3 Max, with additional runs constraining both OpenBao and the client to 4 and 2 cores (`GOMAXPROCS`) to approximate a small appliance VM. A server vCPU is slower than an M3 core; halving the constrained figures gives a conservative floor.

| Configuration | Single op, p50 | Sustained | Batched |
|---|---|---|---|
| 16 cores, plain key | 92–104 µs | 40,000–70,000 ops/s | ~900,000 wraps/s |
| 4 cores, derived key with context | 102 µs | ~33,000 ops/s | ~530,000–620,000 wraps/s |
| 2 cores, derived key with context | 100 µs | ~19,000 ops/s | ~320,000–360,000 wraps/s |

A file audit device adds roughly 30 µs per operation and costs about 30% of the sustained ceiling.

Single-operation latency is a serial round trip and does not vary with cores at all; a slower core moves it from ~0.1 ms toward ~0.2 ms, which stays invisible inside a GET that already walks the MST and fetches from Piri. The per-request cost decomposes into roughly 100 µs of connection, JSON, and auth handling against 6–9 µs of actual key wrap per item. The bottleneck is request mechanics rather than cryptography, which is exactly what `batch_input` amortizes: a 2-core configuration still wraps over 300,000 keys per second because a batch pays the request overhead once.

The Region-KEK comparison appended to PR #19 (in its analysis notes) puts Vault transit at "0.3–1 ms" per operation, a "3k–10k PUT/s" ceiling, and "~1 hour" to rotate 10M parts, and concludes "this is putting a ceil on the maximum throughput we can achieve." Measured locally, the per-operation cost is three to ten times lower; the two-core floor roughly doubles the top of that estimate, and roughly meets it after halving for slower server cores. A 10M-part rotation at constrained batch rates is under a minute of transit time, with the Postgres row updates dominating the campaign. The intermediate-KEK layer that table motivates (held unwrapped in Ingot's memory, keeping the manager off the per-part path) addresses a throughput limit the local measurements do not reproduce, and it would carry more long-lived key material in application memory while giving up per-part custody, audit, and context binding. The same table's YubiHSM column is right, though: ~10 ms per operation makes an HSM on the per-part path infeasible. An HSM belongs at the root of trust: the central unseal today, a PKCS#11 provider later.

If the read or write path ever needs more than ~20,000 wrap operations per second (the two-core sustained rate), batching raises it and is simple to implement: concurrent requests append to an in-memory accumulator, and a single submitter flushes them through `batch_input`. Roughly 250 items fit per request at OpenBao's default JSON-complexity limit once each item carries its context, and the limit is raisable in listener configuration. Callers still see per-object wrap and unwrap; the change is internal to the provider. Measured batch rates run roughly fifteen to twenty times the per-operation rate on the same cores.

## Alternatives Considered

**FIL-547 as drafted: in-process A256KW with locked memory.** Keeps the RFC's algorithm at the cost of hand-built memory protection our Principle 4 exists to rule out, and the protection is weaker than it looks: the locked buffer guards 32 bytes while the AES key schedule sits in ordinary heap. Rotation campaigns would also pass every CEK through Ingot's memory. Rejected; its `Provider` seam is kept.

**Intermediate KEKs in application memory (explored in PR #19's notes).** The standard cloud-KMS envelope pattern, motivated by a per-part limit the measurements do not show. It increases the key material held by the application (an intermediate KEK exposes every part beneath it, where a CEK exposes one object), and it forfeits per-part audit and context binding. Not needed at measured rates.

**Region KEK in a local file.** Fails the dead-disk property outright; an imaged disk decrypts the region. Rejected.

**Central OpenBao on the read path.** Wrapping against central directly would put a WAN round trip and central availability inside every GET, which the encryption RFC's Independence criterion forbids: "Reads in a region succeed with Hilt unreachable." Rejected.

**Local OpenBao with local unseal material.** An unattended appliance must hold its own unseal secret on the same disk it protects, which reduces to the local-file alternative with extra steps. The central transit seal is what makes the local OpenBao meaningful. Rejected.

**Trusted execution environments (Confidential Metal and kin).** Aimed at the adversary Principle 3 rules out, at the cost of a bet on operationally novel technology. Worth revisiting if the threat model or the technology's maturity changes.

**LUKS + Clevis/Tang volume binding (from PR #19's appended notes).** The same central-root principle applied at the volume layer, and complementary: it extends dead-disk coverage to Postgres and spool contents wholesale. It has no custody, audit, or rotation semantics, so it supplements the secrets home rather than substituting for it.

## Open Questions

- Remote kill of a running appliance: a central-triggered seal of the local OpenBao would close the discovery-to-restart window. Lower priority, per Principle 2.
- Boot identity hardening: when the host image work (bootc, measured boot) lands, TPM-seal the boot credential; sequencing against the appliance hardening release (R1.5).
- Signing-service (payer) keystore custody: Design places it in the central OpenBao; who lands that change, and when, is open. Coordinate with PR #19.
- Audit device on the appliance: per-wrap audit costs about 30% of sustained throughput, which we do not currently need. On by default, or off?
- Spool and encryption ordering: the dead-disk property requires the PUT pipeline to encrypt before bytes rest in the spool, or the spool to live on a bound volume. This belongs to the [FIL-481](https://linear.app/filecoin-foundation/issue/FIL-481)/[FIL-482](https://linear.app/filecoin-foundation/issue/FIL-482) wiring.
- A separated Piri on its own machine would change what an intruder can reach: Piri holds only FEE ciphertext, so access to a running Piri host could yield the provider wallet but not decrypted customer data. The Principle 3 concession attaches to the machine that decrypts, so a split deployment could keep bulk storage on less-trusted hardware. Relevant to future deployments.
- Who operates the central OpenBao, and its availability target; ties into PR #19's open question of who operates which layer.

## Evaluation Criteria

Ideally, we'll be able to write Smelt tests around these criteria, following the encryption RFC's practice.

- **Dead disk.** Image every volume of a powered-off appliance: no object plaintext, no key material, and no still-working credential is recoverable, for any object in the region.
- **Startup kill.** After central revocation, a restarted appliance fails to start its secrets service and serves no encrypted object.
- **Independence.** Steady-state reads succeed with central unreachable, unchanged from the encryption RFC.
- **Stock machinery.** Every security mechanism in the shipped design maps to a documented feature of OpenBao or the host OS; the diff adds no security-critical code beyond glue, and FIL-547's memory-management code is deleted.
- **Throughput.** On appliance hardware, one instance sustains at least 20,000 wrap operations per second, and wrap latency at the region's observed peak request rate holds p99 under 1 ms.
- **Rotation.** A 10M-part region rotates its KEK with no CEK observable outside OpenBao, in minutes.

## References

- [RFC: Fil One Object Encryption](./2026-06-fil-one-encryption-design.md) — the design this RFC amends (region wrap algorithm and custody); tenant wrap, FEE, and cryptoshredding unchanged.
- [RFC: FilOne Appliance Deployment Strategy (PR #19)](https://github.com/fil-one/RFC/pull/19) — deployment context, in draft; this RFC picks up its open question of where the Region KEK lives, and shares its position that the payer wallet stays off the appliance.
- [FIL-547](https://linear.app/filecoin-foundation/issue/FIL-547) — region key provider interface (the `Provider` seam is kept, the in-process wrap implementation retired) · [FIL-480](https://linear.app/filecoin-foundation/issue/FIL-480) — wrap material on `blob_locations` · [FIL-481](https://linear.app/filecoin-foundation/issue/FIL-481)/[FIL-482](https://linear.app/filecoin-foundation/issue/FIL-482) — PUT-path encryption and wiring.
- [NIST SP 800-38F](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-38F.pdf) §3.1 (AEAD modes approved for key protection) · [FIPS 140-3 IG](https://csrc.nist.gov/csrc/media/Projects/cryptographic-module-validation-program/documents/fips%20140-3/FIPS%20140-3%20IG.pdf) D.G (AES-GCM as approved key wrapping) · [NIST SP 800-38D](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-38d.pdf) §8 (GCM invocation bounds).
- [AWS KMS Cryptographic Details](https://docs.aws.amazon.com/kms/latest/cryptographic-details/crypto-primitives.html) (AES-256-GCM data-key wrapping with context AAD) · [RFC 7518 §4.7](https://datatracker.ietf.org/doc/html/rfc7518#section-4.7) (A256GCMKW in JOSE).
- [openbao#354](https://github.com/openbao/openbao/issues/354) / [openbao#363](https://github.com/openbao/openbao/pull/363) (mlock removal and host-level guidance) · [OpenBao transit seal](https://openbao.org/docs/configuration/seal/transit/) · [OpenBao unix listener](https://openbao.org/docs/configuration/listener/unix/).
