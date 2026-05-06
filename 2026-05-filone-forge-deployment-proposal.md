# Fil One on Forge — Deployment Proposal: Co-located Guppy + Piri

**Status:** Proposal for discussion
**Author:** Hannah Howard
**Audience:** Forge engineers (assume full Forge context)
**Date:** 2026-05-05

---

## TL;DR

We've been struggling with how to graft Fil One's S3 product onto Forge
without losing either (a) the operational visibility that's the whole reason
for building Fil One on Forge in the first place, or (b) the regional
latency/egress economics that make it sellable.

The crux of this proposal: **co-locate Guppy and Piri at the storage provider,
while keeping the Upload Service (and probably the Indexing Service) at Fil
One.** Guppy stops being "the client in front of the upload service" and
becomes "the regional S3 facade and data-prep stage that happens to be next
door to its Piri." Bytes never cross regions; only UCAN control plane
traffic does.

This was non-obvious to me because Guppy and Piri don't talk directly in the
current architecture — they communicate through the Upload Service. So
co-locating them looked irrelevant. It isn't.

Naming, since we've changed it twice:

- **Service Orchestrator (SO)** = Fil One in this proposal.
- **Provider** = runs Guppy + Piri together (not just Piri).

## Recap: Proposals #1 and #2

**#1 — Centralized Forge stack at Fil One.** Fil One runs Guppy + S3 facade +
Upload Service + Indexing Service; providers run Piri only. *Killed by:* a NA
client uploading to a NA Piri via an EU Guppy pays two transatlantic
round-trips per object plus 2× the bandwidth bill. Unworkable.

**#2 — Full stack per region.** Each provider runs Guppy + Upload Service +
Indexing Service + Piri as a regional Forge deployment; Fil One becomes a
tenant-management layer on top. *Killed by:* (a) huge software footprint per
provider — operationally we're back to "providers do everything differently,"
the current Fil One problem we're trying to escape; and (b) Fil One loses the
centralized Upload Service / Indexing Service vantage point that was the
entire reason for adopting Forge over the existing DNS-router architecture.
We solve egress at the cost of becoming a router again.

Both proposals share an unstated assumption: **services that are
architecturally adjacent must be deployment-adjacent.** Proposal #3 breaks
that assumption.

## Proposal #3: The Architecture

### The split

| Component | Runs at | Why there |
|---|---|---|
| **S3 facade + Guppy** | Provider, regional | Receives client bytes; data prep next to where bytes will land |
| **Piri** | Provider, regional | Storage |
| **MST content catalog** | Provider, regional | Per-bucket — single-writer-per-bucket falls out naturally |
| **Upload Service (sprue)** | Fil One, central | Sees every blob/add, index/add, upload/add — full control plane |
| **Indexing Service** | TBD: central at Fil One, OR regional with IPNI as the only global indexer | See "Where the Indexing Service lives" below |
| **KMS, Delegator, Etracker, Signing Svc** | Fil One, central | Same as #2 |

### Data plane vs control plane

The architectural insight is to split the upload pipeline into two layers
with different locality requirements:

- **Data plane** — actual blob bytes. Stays in-region. Client → provider's
  Guppy → provider's Piri. No bytes cross to Fil One.
- **Control plane** — UCAN invocations: `blob/allocate`, `blob/accept`,
  `index/add`, `upload/add`, content claims. Cheap (KB-scale) and
  cross-region. Goes provider → Fil One.

This mirrors S3's own internal architecture (data plane regional, IAM/control
plane global) and preserves the Forge invariant that **every operational
event is a UCAN-receipted invocation Fil One sees and signs off on**. That's
the visibility property we kept failing to preserve in #2.

### The upload path

Three pieces compose:

**(1) Affinity routing on `blob/add`.** Sprue needs one new rule: every Space
has a home Provider, and `web3.storage/blob/allocate` for that Space must go
to the home Provider's Piri. No load-balancing across Piris within a region
(internal provider problem if they have multiple Piris). No cross-region
allocation (would defeat data-plane locality). The same rule
**future-proofs the direct-Guppy-not-colocated case** — a customer running
their own Guppy on-prem invokes `space/blob/add`, the same rule fires,
allocate goes to the home Provider's Piri, customer's Guppy does an actual
HTTP PUT over the public internet to that Piri. Co-located is just the
optimization where the PUT happens to be localhost.

**(2) Skip the transport hop.** The standard sub-flow per blob is
`blob/allocate` → presigned URL → HTTP PUT → `ucan/conclude` (carrying the
http/put receipt) → `blob/accept`. In a co-located deployment the HTTP PUT
is silly — Guppy already holds the bytes and Piri is across the rack.
Instead:

1. Guppy issues `blob/allocate` to Fil One's Upload Service.
2. Upload Service forwards to provider's Piri.
3. Guppy hands bytes to Piri locally (local HTTP endpoint, shared volume,
   or shared object backend — see "What's still hard").
4. Guppy issues `ucan/conclude` with a synthetic http/put receipt — bytes
   are demonstrably at Piri.
5. `blob/accept` fires; Piri verifies the multihash from local bytes and
   publishes the location claim.

Piri still hashes and verifies on accept — the trust invariant that "Piri
proves what it stores" is intact. We're short-circuiting the *transport*
hop, not the *verification* hop.

**(3) Async PUT-ack at the S3 boundary.** The full chain through Fil One has
~3-4 cross-region RTTs (~300-500ms for a small object PUT, before object
size). Acceptable but not great. Mitigation: Guppy ACKs the S3 PUT to the
client as soon as bytes are durable on local Piri *and* the MST entry is
written locally; the rest of the chain (`blob/accept`, IPNI publication,
`index/add`, `upload/add`) runs asynchronously. This breaks one Forge
invariant — `upload/add` is no longer recorded before the client sees
success — but it's a controlled break: Fil One eventually sees the full
chain, and reconciliation is a background job at Guppy plus a "stuck
uploads" alarm.

### The retrieval path

GETs follow from the same observation: the provider's Guppy is by
construction the regional endpoint for the buckets it serves, and every
blob in those buckets lives at the local Piri.

In-region GET path:

1. Client GET hits provider's Guppy.
2. Guppy resolves S3 key → root CID via the local MST.
3. Guppy queries local Piri directly for the blob (and byte ranges if
   known from the local index).
4. Streams back to client.

No Indexing Service hop, no cross-region RTT, no dependency on the
Indexing-Service-cache-write-through path for read-on-write. Read-on-write
is provided structurally: bytes at local Piri after PUT, MST entry at local
Guppy, GET stays in-region.

Indexing data still gets published — `assert/location`, `assert/index`, IPNI
advertisements — as part of the async PUT pipeline. The Indexing Service
keeps a global view for observability, the future global-network case, and
out-of-region or non-Guppy retrieval. It's just no longer on the in-region
GET hot path.

The narrowing assumption that makes this safe: provider-Guppy serves only
buckets whose home provider is itself, and only retrieves from local Piri.
If we ever federate a bucket across providers (cross-region replication,
multi-provider durability), that bucket's GETs fall back to the standard
Indexing-Service-resolved path. The fast path is opt-in by topology, not a
hard architectural commitment.

### Where the Indexing Service lives

The remaining topology question. Two viable answers:

**(a) Central at Fil One.** Simplest deployment; same Indexing Service we
have today. Cold queries — rare, since most queries hit local
Guppy+MST+Piri — pay one cross-region RTT.

**(b) Regional at the provider, IPNI as the only global indexer.** The
Indexing Service is fundamentally a cache + query amplifier in front of
IPNI; IPNI is already the durable, globally-queryable backing store. Move
the Indexing Service into the provider's deployment alongside Guppy and
Piri; `claim/cache` and `assert/index` go local; IPNI handles the
cross-region role IPNI was designed for. The bucket-pinned-to-provider
topology dissolves the cross-region cache-consistency problem that's been
blocking regional indexers on the Forge roadmap.

Cost of (b): one more component in the provider deployment (still far less
than #2 — Indexer is pure Go, much smaller than sprue), and each region
runs its own IPNI publisher (need rate-limited / aggregated publication;
IPNI doesn't love a thousand small publishers).

I'm leaning **(b)**: the cross-region consistency problem was the only thing
that had been stopping regional indexers, and proposal #3's bucket-affinity
model resolves it for free. Fil One can run a small Indexing-Service-style
cache for observability/dashboards, but it isn't on a hot path. Wants team
input before locking in.

## Claude's Evaluation

I asked Claude (the AI assistant) to red-team this proposal. The section
below is its read, not mine — useful as an external check before circulation.

### What we lose

**1. Replication across providers.** Forge's network-orchestrated replication
is gone. Each bucket lives on one provider; durability inside the region is
whatever that provider's redundancy gives us. **Strongest critique.** Counter:
AWS S3 has the same model — single-region durability is internal to the
provider, cross-region replication is opt-in. Service-Orchestrator-grade
providers have erasure coding, multi-rack, and SLAs that look more like AWS
than like a commodity Filecoin SP. We're aligning with the S3 product mental
model, not regressing from it. Customer-driven CRR can be added later (Fil
One issues a `replicate` capability to a second provider's Piri). **Net:
fair tradeoff for the SO product, but the proposal should be explicit about
it in the marketing rather than implicitly carrying the "no single-provider
dependency" claim from the global-network era.**

**2. Client-side data-prep verifiability.** Same as #1 and #2 — any S3 facade
loses this. The S3 client trusts the facade to hash correctly; the
direct-Guppy path preserves the original property for customers who need it.
Unchanged across all three proposals.

**3. Async PUT completion is a new Forge invariant break.** PUT-ack happens
before `upload/add`. The customer's S3 contract is preserved; the audience
that cares is internal observability/billing. Reconciliation loop has to be
designed and fault-tested.

### What we win (vs proposal #2)

**1. The MST concurrency problem largely evaporates.** Q2 scoping flagged
this as the top risk: indigo/mst is single-writer-per-repo, but the original
plan had multiple S3 facade replicas writing the same bucket concurrently.
In #3 a bucket lives at one provider with one writer. Single-writer falls
out of the topology.

**2. Far less software per provider.** #2's provider runs Upload Service +
Indexing Service + KMS + Delegator + Etracker + Signing Svc + w3clock-or-
replacement + Postgres + MinIO + Guppy + Piri. #3's provider runs Guppy +
Piri (+ optionally Indexer). ~5× reduction in operational surface, and the
bits we keep are the bits providers are operationally expert in. Control
plane goes to Fil One, who runs control planes for a living.

**3. Fil One sees everything that matters.** Every blob/add, blob/accept,
index/add, upload/add, retrieval auth, egress, billing flows through Fil
One. The original "central control plane" benefit Forge promised, on
infrastructure we operate.

**4. Single-region S3 latency stays single-region.** Bytes never leave the
provider's data center for in-region traffic; control plane latency hides
behind async PUT-ack and the retrieval fast-path.

### What's still hard / unresolved

**1. The Guppy↔Piri local handoff** is unspecified, and unspecified things
are usually harder than they look. Three plausible options: **(a) local HTTP
import endpoint on Piri** (simple, fits Piri's existing model);
**(b) shared volume / object backend** (clean if Piri's storage backend
abstraction is already pluggable, which it is); **(c) library mode**
(probably overkill). Recommendation: **(b)** if it works cleanly, **(a)**
otherwise. Piri change, not Forge-protocol change. If Piri's storage
abstraction can't accommodate (b), the ripple extends to the global-network
deployment too.

**2. Tenant onboarding flow.** Fil One creates Spaces (holds keys), issues
delegations to provider Guppy DIDs, pushes (API_key → Space + delegation)
to the provider's Guppy via an admin API that doesn't exist today. Modest,
but a real protocol with real edge cases (missed pushes, restart with stale
state, partitions during rotation) — not a single endpoint.

**3. Async PUT-ack reconciliation.** "Eventually-fully-registered uploads"
is a real new failure mode. Has to be defensible — fault-injected before
launch.

**4. Cross-region UCAN RTT is hand-waved.** Should be benchmarked on
representative regions before async-PUT-ack becomes a hard requirement
rather than an optimization. Most likely fine, but unverified.

**5. Cross-region replication coordination.** Out of v1 scope, but plausibly
Fil One issues a `space/replicate` to a second provider's Piri, which
fetches from the source. Adds a hop, but only on opt-in.

**6. Failure modes.** Fil One Upload Service down: GETs continue (local MST
+ local Piri handle the whole path). PUTs continue *from the customer's
perspective* too — the async PUT-ack design acks once bytes are durable on
local Piri and the local MST entry is written; the upstream chain
(`blob/accept` → IPNI → `index/add` → `upload/add`) just queues for when
Fil One is back. Roughly equivalent to #2 on customer-facing availability
during a Fil One outage. Provider down: that region's bucket unavailable.
Same as #2.

## Changeset vs Current Q2 Scope

What changes if we adopt #3 versus the current Q2 scoping doc. Trying to be
precise about scope deltas, not rewriting work areas wholesale.

### Per work area

**WA1: S3 API Facade** — *moves from "runs on Upload Service" to "runs at
provider alongside Guppy".* Code structure ~unchanged; deployment topology
changes. New work: tenant management API at the provider, admin API between
Fil One and provider's Guppy (~3d), async PUT-ack semantics +
reconciliation (~5d). Removed: facade↔upload-service co-location plumbing.
**Net: ~+0.25 EM.**

**WA2: Upload Pipeline** — *mostly unchanged, plus local handoff.* Same
flat-file pipeline, encryption, multipart. New: local handoff between
Guppy and Piri — Piri-side `/local-import` (~3d) or shared-storage-backend
config (~3-5d). `ucan/conclude` short-circuit on the Guppy side. **Net:
+0.25 EM at most.**

**WA3: Upload Service (sprue)** — *simpler, plus affinity routing.* Sprue
stays at Fil One. New: Space → home Provider affinity routing on
`space/blob/add` (~3d, one new field on Space). Note: every blob/add is now
over-WAN; sprue should batch and retry idempotently. **Net: ~unchanged.**

**WA4: UCAN 1.0** — unchanged. No topology effect.

**WA5: MST Catalog** — *simpler concurrency model.* MST per provider per
bucket. Single-writer-per-bucket falls out of topology; the Q2 #1 risk
("multi-writer with indigo/mst") evaporates. HA within a provider is
internal leader election, not cross-region. **Net: -0.25 EM.**

**WA6: Object Lock + Versioning** — unchanged. No topology effect.

### Previously unscoped, now in play

**WA7: Auth & Tenancy.** Marked out-of-scope in Q2, but the team has
correctly pointed out *something* auth/tenancy-shaped has to ship for any
of this to be operable. Proposal #3 makes the minimum unavoidable surface
concrete:

- API-key → Space + UCAN delegation lookup at the provider's Guppy.
- Tenant onboarding admin API (Fil One → provider's Guppy) for delegation
  + key push, with reconciliation for missed pushes / partitions / rotation.
- AWS Signature V4 validation in the S3 facade (fiddly but the same
  surface in any architecture, not a #3-specific cost).
- A finished design for "S3 API key as session-key-for-UCAN-delegation."
  We have a sketch; the cryptographic binding (does the key sign anything?
  replay protection?) is still open.

Roughly the +0.5 EM the changeset adds. **Suspicion: it's a wash** — #2
would need the same work to be operable; #3 just makes it explicit. If
anything #3's tenant model is *simpler* (one well-defined location for the
delegation) and admits things that were out of reach in #2 — most notably
multi-region API keys (out of Q2 scope, but the architecture admits it
cleanly). +0.5 EM is the floor, not the ceiling, but neither is it worse
than what was implicitly carried in #2.

**On-chain payment flow.** Also unscoped in Q2, also unblocked by #3.
Honestly, **proposal #3 is the only path I see that gives us any hope of
getting this done**, and that should weight the comparison vs #2
substantially.

The problem in #2: the provider ran Guppy *and* the Upload Service *and*
Piri, with no independent witness for what they were being paid for. PDP
proves the provider has the bytes they registered, but the provider was
also the registrar — they'd be attesting to themselves, and if they ran the
signing service too, signing their own paychecks. Every version we sketched
collapsed under its own complexity.

Proposal #3 gives the design a viable shape: Fil One runs the Upload
Service (independent witness), Fil One runs the signing service (clear
authority), provider runs Piri (proves possession of bytes Fil One
independently registered), smart contract pays the provider directly
on-chain. Registration → proof → signed payment attestation → on-chain
payout.

Real work still remains within that shape: the provider's Guppy is still
trusted on the S3 path to hash and register what was actually uploaded
(end-to-end customer verification only on the direct-Guppy path); lifecycle
("when do we *stop* paying for a blob?") is a state machine that doesn't
exist in Forge today; per-customer attribution off aggregated proof sets is
a separate ledger; the existing piri-signing-service isn't yet wired to
"Upload Service view of paid data"; and inserting Fil One as wholesaler is
an economic structure (fiat ingress, on-chain payouts, treasury
reconciliation) as much as a technical one.

So this is not an easy win — it's substantial work either way. The point
is: under #2 we couldn't even define the implementation cleanly; under #3
we can. Worth weighting in the architectural comparison rather than
treating as out-of-scope noise.

### Net delta

Additions: tenant-config admin API (~3d), local-handoff endpoint (~3-5d),
async PUT-ack reconciliation (~5d), retrieval fast-path in Guppy (~3-5d),
sprue affinity routing (~3d). **~3-4 weeks total.**

Subtractions: ~1 week of cross-region MST concurrency work, plus simpler
Indexing-Service-cache-write-through requirements thanks to the retrieval
fast-path.

**Net delta: +0.5 to +0.75 EM.** Within the existing 1.5 EM padding.

Removed/deferred from #1/#2: network-orchestrated replication (explicit
out for v1; opt-in CRR later); provider-runs-control-plane operational
packaging (the elephant in #2); cross-region MST coordination.

## Open Questions for the Team

1. **Piri storage backend abstraction**: does it support a "Guppy writes,
   Piri verifies and accepts" pattern cleanly today? (For Forrest/Paul.)
2. **Cross-region UCAN RTT** between provider Guppy and central sprue —
   measured latency on representative regions. Determines whether
   async-PUT-ack is a hard requirement or just an optimization.
3. **Tenant onboarding API shape** (Fil One ↔ provider Guppy) — push, pull,
   webhook?
4. **Indexing Service: central at Fil One, or regional with IPNI as the
   only global indexer?** Leaning regional — bucket-affinity topology
   dissolves the cross-region consistency problem that's been blocking
   regional indexers, and IPNI already plays the global-indexer role. Want
   pushback before committing.

## Recommendation

Adopt proposal #3. It threads the needle between #1 and #2 by recognizing
that the data plane and control plane have different locality requirements,
which we'd been treating as one problem.

If the team agrees, the immediate next steps are:

1. A focused design doc on the Guppy↔Piri local handoff.
2. A latency benchmark of the cross-region UCAN path on representative
   regions.

Both are small and targeted, and would de-risk before we restructure WA1's
deployment topology.
