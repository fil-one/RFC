# RFC: Unified Login & Enrollment for Forge Networks

**Status:** Draft
**Scope notes:** Revocation is deferred to a follow-on RFC (see §7). A full security analysis
section will be added after design review. This document assumes a greenfield network — no
migration path is required; nothing described here has ever booted.

---

## 1. Motivation

Every pair of Forge services that interact via UCAN needs the same thing: a delegation chain
from an authority to an agent, persisted somewhere it can be replayed as proofs at invocation
time. Today we deliver those chains through **five different mechanisms**:

| Mechanism | Used by | How authority arrives |
|---|---|---|
| Interactive email login (`/access` flow) | guppy | `/access/request` → email click → `/access/claim` → `tokens.cbor` |
| Registrar-brokered enrollment | piri | `piri init` → delegator allow-list check → minted chains frozen as base64 into TOML (`piri/cmd/cli/setup/register.go`) |
| Manual proof files | delegator | `delegator gen` run by a human **with both services' private keys in hand**; file paths in `delegator.yaml` |
| Admin registration | sprue | operator runs `sprue provider register <did> <url> <proofs>`; stored in DB |
| Nothing (validator-only) | indexing-service, etracker | hold own key; their chain roots are extracted *from* them via `delegator gen` |

(ingot is a sixth story: its `login` command was retired when Hilt took ownership of tenancy,
leaving its token store documented as unpopulated — `ingot/module.go:207-215`.)

The costs of this divergence:

- **Humans are the protocol on the worst edges.** Minting the root delegations requires
  physically co-locating private keys (`delegator gen -f <other-service-key.pem>`).
- **Proofs frozen in config can't rotate.** piri's chains are base64 blobs in TOML, fixed at
  `init` time; rotation means re-init and config rewrite.
- **Allow-listing means copying DIDs.** SP onboarding requires the SP to extract a DID and the
  network operator to insert it into DynamoDB (e.g. `smelt/systems/piri/register-did.sh`).
- **Bootstrap ordering is load-bearing.** smelt encodes proof-generation → delegator →
  piri-init ordering in compose dependencies; any deviation fails hard.
- **Four persistence formats** for the same concept: `tokens.cbor`, inline TOML base64, DB
  registries, in-memory startup minting.
- **Two divergent account-onboarding systems** (guppy's access flow vs. ingot's Hilt).

### Goal UX

- **Network owner:** deploy the services, log each in with the company email, done.
- **Storage provider:** share your email with the network owner out of band, install the node,
  run `login`, get registered.

One verb — `login` — on every service.

---

## 2. Design Overview

### 2.1 One mechanism, roles as policy

Login does exactly one thing: after proving control of an email address, it binds an agent DID
to an account. **What that account may then claim is a policy decision on the server side, not
a property of the mechanism.** The two user roles fall out:

- **Network owner** logs sprue, indexing-service, etracker in with `ops@company.com` — every
  service agent binds to the same account, which becomes the network's governance root.
- **Storage provider** logs piri/ingot in with their own email, which the network owner has
  pre-authorized on an allow-list (an email you already know, not a DID you must be sent).

A role claim ("I am your indexer") is accepted only from the owner account; from any other
account, only storage-node roles are grantable. The role appears explicitly in the
confirmation email so a hijacked login attempt cannot silently enroll as an owner service.

### 2.2 The Authority

One service — the **Authority** — merges what are today the delegator's brokering, Hilt's
tenancy, and sprue's `/access` hosting. It is deployed first and is the only service that
never logs in. It provides:

- **Email verification** for login ceremonies (SMTP).
- **Registry:** role→DID bindings, node records (endpoint, status), the service-DID map
  validators consult (replacing hardcoded `presets.PrincipalMapping`-style tables).
- **Allow-list:** invited SP emails.
- **Mailbox:** the delegation store behind `/access/delegate` / `/access/claim` (§2.3).
- **PLC directory** for identity resolution (§3).
- **Tenancy** (Hilt duties): minting tenant spaces, issuing space delegations to ingot.

### 2.3 The mailbox

The Authority's delegation store behaves as a **mailbox keyed by audience DID**:

- `/access/delegate` deposits a delegation into the mailbox of whoever its `aud` names.
  Deposits are durable and deduplicated by CID.
- `/access/claim` (subject = caller's DID) reads the caller's own mailbox.

Everything in this design is these two verbs plus policy. Both already exist in the codebase
(guppy/ingot clients, sprue handlers); they have simply never been pointed at
service-to-service edges.

### 2.4 The Authority's two distinct parts — and the trust boundary between them

**Chain intermediary — only where fan-out is needed.** The indexer and etracker delegate *to*
the Authority (`aud: authority`) because their capability must reach a recipient set that
doesn't exist yet: every future piri node. The Authority is a real link in those chains
(`indexer → authority → piri-N`) so it can mint the last hop per enrollment. These are the
only capabilities the Authority ever *holds by delegation*. (One caveat: in its tenancy role
the Authority also *custodies tenant-space keys* — §3.1. That is key possession, not
delegated capability, and it is a strictly larger trust grant; the deferred security
analysis must treat custody separately from these two delegations.)

**Courier — everywhere else.** piri's blob grant names sprue directly (`aud: sprue`); the
Authority stores and forwards it but is not in the chain and can never use or extend it.
Tenant-space chains to ingot, likewise. The sprue→indexer edge involves no delegation at all —
identity-based trust via the registry.

**Rule:** delegate to the Authority only when it must make grant decisions on your behalf
(a dynamic recipient set). When the recipient is known, address them directly; the Authority
is a mailbox. The wiring is centralized; the trust mostly isn't.

### 2.5 Chains are key-rooted; email roots nothing

`did:mailto` + `ucan/attest` was purpose-built (by Storacha, whose design the access flow
inherits) for the one case where key custody is impossible: human accounts. Services custody
keys fine. Therefore:

- Every service-to-service chain terminates at the key of the service that owns the resource
  (`iss == sub`). There is no shared cryptographic ancestor and no attestation machinery on
  service edges; validators need no trusted-attestor configuration.
- The account (email) is the **exchange rail and governance root** — it authorizes bindings
  in the registry — never a chain root.
- `did:mailto` roots remain exactly where they are today: the guppy human-client edge.

This follows the pattern every analogous system converged on (ACME, SPIFFE, OAuth device
flow): human-friendly identity authenticates the enrollment ceremony; stable keys root the
operational trust.

---

## 3. Identity

### 3.1 DID methods

| Principal | Method | Senior rotation key | Operational key |
|---|---|---|---|
| Authority | `did:plc` | Owner cold key (offline) | Authority service key |
| sprue / indexing-service / etracker | `did:plc` | Owner cold key | Per-service key |
| piri / ingot nodes | `did:plc` | **SP's own cold key** | Node key |
| Tenant spaces | `did:plc` | Authority (until custody transfer) | Space key, Authority-custodied |
| Ephemeral agents (CLI processes, per-request agents) | `did:key` | — | — |
| Human accounts (guppy edge) | `did:mailto` + attestation | — | — |

**`did:web` is not used as an identity method.** Domains retain two support roles only:
TLS transport, and verified aliases (§3.4).

Rationale for `did:plc` on durable principals:

1. **Chains survive key rotation.** UCAN delegations name DIDs. With `did:key`, rotating a
   service key kills every chain touching it — for a design built on `exp: never` roots, that
   would make rotation a network-wide re-enrollment event. With `did:plc`, validators resolve
   the DID's *current* key; rotation is private to the service.
2. **Identity is decoupled from DNS custody.** A `did:plc` document is controlled by rotation
   keys; the directory cannot forge operations. Domain compromise breaks a label, not an
   identity.
3. **A cryptographic governance root.** Genesis operations *list* rotation public keys — no
   signature from the senior key is needed at minting — so the owner's cold key sits offline
   above every owner service, and identity recovery is a signed operation from a safe, not an
   email round-trip. SP nodes' senior keys belong to the SP: providers are sovereign; the
   network's leverage is registry state, never their identity.

**What a tenant space is.** Unlike every other row in the table, a tenant space is a
principal with nothing behind it — no server, no endpoint, no process. It names the tenant's
data namespace, and it gets a DID because UCAN authorization chains must terminate at the
resource's owner: the space DID is the `sub` of every storage operation on that tenant's data,
so every such chain is self-rooted (`iss == sub == space`) and verifiable offline. The
alternative — spaces as opaque bucket IDs — would put data ownership back inside a service's
policy engine. At provisioning, the Authority generates the space keypair, mints the
`did:plc`, signs the space's gateway grants (§4.2), and custodies the key thereafter —
hosted-wallet semantics, appropriate because ingot tenants speak S3, not UCAN. `did:plc`
rather than `did:key` (guppy's human-managed spaces) because delegations, index claims,
egress records, and billing all reference the space DID durably: the plc rotation mechanism
lets the key rotate after custodian compromise, and lets custody itself transfer — a tenant
graduating to self-custody, or migrating their space to another provider or network — without
orphaning a single reference.

### 3.2 Self-certification and the untrusted directory

A `did:plc` identifier is the hash of its genesis operation; every subsequent operation chains
to its predecessor under rotation-key signatures. Resolution = fetch the operation log, verify
the chain against the DID string itself. **The directory is trusted for availability only** —
it can withhold or serve stale logs, never forge. Mirrors are therefore trustless, and cache
poisoning is structurally impossible (a verified state is immutable-by-construction).

The residual risk is **rollback/withholding** (a truncated log omitting a recent rotation),
mitigated operationally: short TTLs, mirror cross-checks on sensitive resolutions
(enrollment, role binding). See §5 defaults.

Cost accepted: validators implement a plc verifier (hash/CID computation, multi-curve
signature checks, branch-nullification rules) instead of a JSON fetch — a few hundred lines of
well-specified code, in exchange for removing DNS from the trust base. Implementations exist
in the atproto ecosystem, including Go.

### 3.3 The network descriptor

The genesis artifact, shipped with SP binaries / service config:

```
{
  network:            "forge-mainnet",
  authority_did:      "did:plc:...",
  authority_url:      "https://auth.forge.company.com",
  directory_urls:     ["https://auth.forge.company.com/plc", ...mirrors],
  owner_recovery_key: "<owner cold public key>"
}
```

Because `did:plc` is self-certifying, resolution requires nothing beyond the pinned DID
string. The one other piece of trust material, `owner_recovery_key`, is provisioning input —
the senior rotation key owner services list at minting (§4.4) — not resolution input.

### 3.4 Domains as verified aliases

atproto-style bidirectional binding: the DID document lists
`alsoKnownAs: indexer.forge.company.com`, and a DNS TXT record
(`_forge.indexer.forge.company.com = did:plc:...`) points back. Humans get legible names and
an independent second binding channel; validators pin DIDs and never depend on names.

### 3.5 Resolver surface

Exactly two resolvers in every validator: `did:key` (inline) and `did:plc` (directory +
cache). No `did:web` resolver, no preset principal maps — the registry serves name→DID and
the DIDs self-verify.

---

## 4. Protocol

### 4.1 Who signs what: DIDs and signatures in the graph

Every name appearing in the graphs below (`indexer`, `authority`, `piri-N`, `tenant-space`,
…) is a **`did:plc`** — per §3.1, all durable principals are. `did:key` appears in this
protocol only for ephemeral agents on the human edge, and `did:mailto` only as the guppy
account root. So in a delegation like `{iss: indexer, aud: authority, sub: indexer}`, all
three fields hold `did:plc` strings.

**Signature mechanics.** A DID cannot sign; a key does. A delegation issued by `indexer` is
an envelope signed by the **operational signing key currently listed in
`did:plc:indexer`'s document**. A validator checking it resolves `iss` through the PLC
directory (§3.2) to the current verification key and checks the envelope signature against
that. Invocations verify the same way against their `iss`. Tenant-space grants are signed by
the space's key at creation time, while the Authority custodies it.

**What key rotation does and does not break** — the corollary that makes the scheme coherent:

- Delegations that merely **reference** a DID (as `aud` or `sub`) are untouched by that DID's
  rotation — the chain names the identity, not the key. This is the survival property §3.1
  claims (with `did:key`, rotation changes the DID itself and severs every reference).
- Delegations **signed by** the rotated-away key stop verifying, because validators resolve
  the *current* document. This is deliberate policy: signatures verify against current keys
  only, so rotation implicitly revokes everything the old key signed — which is exactly what
  you want when rotating away a compromised key.
- Recovery is automatic and cheap: after rotating, a service re-issues its outstanding
  delegations (same `{iss, aud, sub, cmd}`, new signature) and re-pushes them via
  `/access/delegate`; counterparties' claim loops pick up the re-minted copies. Chains
  re-assemble across the swap because links are matched by DID continuity at invocation
  time, not by CID reference between envelopes — so when the indexer rotates, only its root
  is re-issued; the Authority-signed extended hops remain valid and compose with the new
  root. Only if the *Authority* rotates must the extended hops themselves be re-minted
  (they bear its signature). No identity changes, no re-enrollment, no config edits.

(The alternative — accepting signatures from historically-valid keys — would preserve old
envelopes across rotation, but a compromised old key would stay dangerous forever. We take
the strict-current-key policy and let the mailbox absorb the re-issuance cost.)

### 4.2 The delegation graph

All delegations written as `{iss, aud, sub, cmd}`. Names abbreviate `did:plc` identifiers
(per §4.1).

**Roots pushed at owner-service login** (deposited via `/access/delegate` into the
Authority's own mailbox — these are the only chains the Authority is *in*):

```
{ iss: indexer,  aud: authority, sub: indexer,  cmd: /claim/cache,         exp: never }
{ iss: etracker, aud: authority, sub: etracker, cmd: /space/egress/track,  exp: never }
```

Note `sub` = issuer's own DID, not the audience — required for downstream chain validation
(the existing codebase documents this constraint at `delegator/cmd/gen.go:87-93`).

**Extended hops minted by the Authority at SP enrollment** (deposited into the node's
mailbox together with the root):

```
{ iss: authority, aud: piri-N, sub: indexer,  cmd: /claim/cache }
{ iss: authority, aud: piri-N, sub: etracker, cmd: /space/egress/track }
```

**Self-rooted grants pushed at SP-node login** (couriered; Authority not in chain):

```
{ iss: piri-N, aud: sprue, sub: piri-N,
  cmd: /blob/allocate | /blob/accept | /blob/replica/allocate | /pdp/info,
  exp: never }
```

**Tenant-space grants minted at tenant provisioning** (Authority custodies the space key at
creation; clean root, couriered to the gateway node):

```
{ iss: tenant-space, aud: ingot-N, sub: tenant-space,
  cmd: /blob/add | /index/add | /content/retrieve }
```

**No delegation at all:** sprue → indexer. The indexer trusts sprue's DID directly, learned
from the registry.

**Validation walk** (example — piri publishes a claim):

```
invocation: { iss: piri-N, aud: indexer, sub: indexer, cmd: /claim/cache,
              prf: [root, hop2] }
```

Invocation issuer is `hop2`'s audience; `hop2`'s issuer is `root`'s audience; `root`'s issuer
equals the subject — self-rooted, valid. No attestations, no third-party trust config.

### 4.3 Per-service mailbox behavior

| Service | Pushes (at login) | Claims (boot + loop) |
|---|---|---|
| indexing-service | `/claim/cache` root → authority | registry facts only (e.g. sprue's DID) |
| etracker | `/space/egress/track` root → authority | registry facts only |
| sprue | nothing | every enrolled piri's blob grant (+ endpoint/weight from node registry) → provider registry |
| piri-N | blob grant → sprue's box | indexer chain, etracker chain → token store |
| ingot-N | (enrollment only) | per-tenant space grants → token store |
| authority | — | the two roots (from its own box) |

piri's login is exactly **one push and two claims**. ingot's tenant claims populate what is
today the documented gap in its token store. (Registry facts travel via `/network/registry/*`
reads, not the mailbox — listed here because they ride the same boot + poll loop.)

### 4.4 Service login flow

1. First boot: service auto-generates its operational key and mints its `did:plc`. The
   genesis op lists the operational key plus a senior rotation pubkey: for owner services,
   the `owner_recovery_key` from the network descriptor; for SP nodes, the SP's own cold
   pubkey, generated or supplied locally (it is *not* in the descriptor, which is
   network-wide).
2. `svc login <email>` → `/network/enroll` invocation to the Authority: agent DID, claimed
   role, endpoint. Authority sends the confirmation email **stating the claimed role**.
3. Operator clicks; Authority applies policy: owner email → any role; allow-listed SP email →
   storage-node roles only, endpoint-serves-DID verification (existing
   `assertEndpointServesDID` behavior), node registered.
4. The CLI polls the Authority until confirmation lands (the same poll-until-claimed
   pattern as guppy's login), then completes: pushes its self-issued delegations (per §4.3),
   starts its claim loop, persists everything to its token store. Restarts are
   non-interactive.

Human login (guppy) is the unchanged `/access` flow, now hosted by the Authority.

### 4.5 Genesis sequence (network owner)

1. Generate a cold rotation keypair, offline. This plus the owner email is the entire genesis
   trust input.
2. Boot the Authority — config: owner email, SMTP credentials, domain. It mints its own
   `did:plc` (owner cold pubkey senior), starts directory, login service, registry, mailbox.
3. Emit the network descriptor.
4. Boot each owner service with the descriptor; `login ops@company.com` on each; click N
   links. Roots land, claim loops converge.
5. `authority invite <sp-email>` to open enrollment. The admin CLI is itself just an agent
   logged in as the owner account (`/network/invite` is owner-only, §5) — there is no
   separate admin credential system.

Owner ongoing duties: approve enrolled SPs (weights, on-chain contract approval — the
existing `RequestContractApproval` path).

### 4.6 SP sequence

1. Share email with network owner (out of band). Owner runs the invite.
2. `piri login sp@example.net`; click the link. Enrollment, chain delivery, and provider-grant
   push all happen underneath.
3. Wait for owner approval; `piri serve`. Total: one command, one click.
4. ingot: identical; tenant provisioning arrives via the Authority's tenancy duties.

### 4.7 Ordering and convergence

Dependencies are real, but **no ordering is fatal**, because of three properties:

1. Pushes land in a durable mailbox, not a live counterparty.
2. Claims are a retried loop, not a one-shot init. An empty mailbox is a normal answer.
3. Missing proofs degrade rather than fail: a service without a needed chain treats the
   capability as "not yet available" and retries.

Worst-case orderings converge: sprue before any piri → grants appear as SPs enroll; a piri
enrolled before the indexer ever logged in → its mailbox lacks the indexer chain until the
root lands, then the claim loop delivers it — nothing is re-run or re-configured. The trust
graph is eventually consistent. The single true sequencing requirement: **the Authority boots
first.**

This also collapses smelt's job from "synthesize the entire trust graph" (keygen +
`generate-proofs.sh` + mounted files + DynamoDB seeding + compose ordering) to "run the same
logins a human would" via dev-mode auto-confirm — meaning dev exercises the production
enrollment path instead of a parallel one that can drift.

---

## 5. Proposed defaults

Stated as proposals; push back in review.

- **Command surface:** new `/network/*` namespace for enrollment — `/network/enroll` (agent,
  role claim, endpoint), `/network/invite` (owner-only), `/network/registry/*` (read APIs for
  validators/services). The mailbox reuses `/access/delegate` and `/access/claim` verbatim.
  The human-account flow keeps `/access/request`/`/access/claim` unchanged.
- **Role pinning:** once a role is bound to a DID, rebinding requires approval from an
  already-enrolled owner agent — never a fresh email verification alone. Email authority is
  genesis-and-recovery only.
- **Mailbox semantics:** deposits deduplicated by CID; claims are non-destructive
  (re-claimable); entries garbage-collected when every contained delegation is expired or
  superseded. No pagination in v1 (bounded by network size).
- **Claim loop:** claim at boot, then poll with jitter — 30 s initially, backing off to 5 min
  steady-state. A push notification channel (SSE hint from the Authority) is a later
  optimization, not required for correctness.
- **Dev/CI mode:** Authority flag `--auto-confirm` scoped to a configured list of agent DIDs
  (or unrestricted in throwaway environments). smelt uses this.
- **PLC directory:** self-hosted (reference implementation) inside the Authority, with the
  descriptor listing mirror URLs. Resolution cache TTL: 5 min for enrollment/role-binding
  resolutions with a mirror cross-check; 1 h otherwise.
- **Alias convention:** `_forge.<domain>` TXT record ↔ `alsoKnownAs`, verified bidirectionally
  before the registry displays the alias.
- **Token store:** one shared implementation in libforge (the `tokens.cbor` container store),
  used by every service. No proofs in service config files, ever.

---

## 6. Open questions

1. **PLC directory hosting** — self-host from day one (recommended above) vs. lean on the
   public `plc.directory` until self-hosting. This is an ops-burden question only; the
   identity model is unaffected either way. It should not reopen the identity design.
2. **SP approval workflow surface** — where weights and on-chain approval are actioned
   (authority admin CLI vs. dashboard), and whether approval gates serving entirely or only
   traffic weighting.
3. **Multi-owner and succession** — one `network_owner_email` suffices for genesis; real
   deployments will want ≥2 administrators and an owner-departure story. The cold-key
   hierarchy answers identity recovery; registry policy (multiple owner accounts? quorum for
   role rebinding?) is undesigned.

## 7. Follow-on work (explicitly out of scope here)

- **Revocation & deregistration** (next RFC): lifecycle for kicking an SP, compromised node
  keys, and role rebinding under `exp: never` roots — including where validators check
  revocation state and the offline-verification trade-off. Until then, the registry's
  node-status field is the interim enforcement point for enrollment-derived activity.
- **Security analysis** section for this document, after design review.

---

## 8. Relationship to existing code

Greenfield ≠ rewrite. The design re-plumbs mostly-existing parts:

| Existing | Fate |
|---|---|
| `/access` client (guppy `pkg/client`, ingot `forgeclient/` — request/poll/claim/delegate) | **Reused** as the login + mailbox client in libforge |
| sprue's `/access/*` handlers (`access_delegate.go` etc.) | **Move** to the Authority |
| delegator registrar (allow-list, endpoint-serves-DID check, chain minting in `internal/services/registrar/delegator.go`) | **Absorbed** into the Authority; DID allow-list becomes email allow-list; `RequestProofs` becomes mailbox deposits |
| `delegator gen`, `generate-proofs.sh`, proof-file config fields | **Deleted** — replaced by login-time pushes |
| `piri init` self-delegations (`register.go:806-819`) | **Reused** shapes; delivery becomes `/access/delegate`; TOML proof fields deleted in favor of the token store |
| `sprue provider register` admin CLI | **Replaced** by sprue's claim loop + Authority node registry (weight-setting remains an owner action) |
| Hilt tenancy (space minting, `did:plc`) | **Absorbed** into the Authority; per-tenant grants delivered via mailbox (fills ingot's token-store gap) |
| etracker (go-ucanto) | **Built** on ucantone from the start (greenfield assumption) |
| `presets.PrincipalMapping` and kin | **Deleted** — registry + self-certifying DIDs |
| smelt trust-graph synthesis | **Replaced** by real logins with `--auto-confirm` |
