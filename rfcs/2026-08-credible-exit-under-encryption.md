# RFC: Credible exit under encryption

Status: Experimental

## Authors

- [Arkadiy Kukarkin](https://github.com/parkan)

## TL;DR

The [MST bucket design](https://github.com/fil-one/RFC/pull/4) rests on a
falsifiable test: hand a departing customer a CAR file and they reconstruct
an interoperable bucket anywhere, with no vendor code. The [encryption
design](./2026-06-fil-one-encryption-design.md) breaks this test: the exit
CAR is now FEE ciphertext, and the only keys that open it live in Hilt, with
no release mechanism specified. Exit degraded from "a file" to "a file plus
a key-escrow handoff that doesn't exist."

This RFC defines what "credible" must survive, then proposes: a
**key-manifest CAR export** -- per-part CEKs re-wrapped to a customer key,
carried as a second root in the exit CAR; O(parts) key work, blobs and CIDs
untouched -- as the standard cooperative exit; **versioned tenant key
release** for full-tenant departure; and an explicit yes/no on whether
vendor-failure-resistant exit ("Tier 2") is promised. Tier 2 is provisional
throughout: the RFC establishes its true cost, but ships Tier 1 only.
**Position as of review: Tier 1 is implemented now; Tier 2 is deferred --
planned, not promised.** The pre-positioning set is specified in a follow-up
RFC before any Tier 2 implementation or customer-facing claim.

## Motivation

### The promise

From the MST RFC (unmerged, but describing shipped code; quoted as the
reference): a bucket is "Web3" only if a customer can walk away with it --
"a content-addressed, self-describing artifact a recipient can reconstruct
without [vendor]-specific code." The exit artifact is a single CAR: MST
root, nodes, manifests, body blocks.

### The break

Under encryption, every stored blob is a FEE envelope: chunked AES-256-GCM
under a per-part CEK, plaintext never persisted. The envelope header carries
the CEK wrapped to a tenant KEK, so the exit CAR is cryptographically
*complete* -- but the tenant KEKs' private halves are custodied by Hilt, and
no mechanism exists, planned or specified, for a tenant to obtain them (the
encryption RFC's recovery flow is future work, framed as internal disaster
recovery). FEE is publicly specified with open implementations, so "no
vendor code" survives -- once key material is in the customer's hands.

The result: an exit bundle the customer cannot read, decryptable solely by
the platform they are leaving.

### Why now

Pre-GA, the exit story is a promise with no customer base to migrate;
post-GA it is a compliance and trust surface. The decision also gates the
MST RFC's merge (the exit test needs a home on `main`) and the
reconciliation of the MST object model with per-part FEE blobs.

## Threat model: what must "credible" survive?

**Tier 1 -- cooperative exit.** The platform is intact and willing; exit is
an offboarding ceremony. "Credible" means: after handoff, the customer needs
nothing further from Fil One, ever, and completeness is verifiable at
handoff time. Options A and B provide this.

**Tier 2 -- vendor-failure-resistant exit.** Exit survives Hilt data loss,
insolvency, or refusal. No departure-time ceremony can provide this; Tier 2
requires pre-positioning, while the platform is healthy, of everything the
customer would otherwise ask for at departure:

1. **Key material** (Option D).
2. **The current bucket root -- at the right durability.** A withheld root
   turns exit into silent rollback (stale checkpoint) or nothing. Closure
   validation proves completeness *relative to* a root, never that it is
   current. And under async PUT-ack, S3 200 promises regional durability
   only: the network-visible root (`forge_root_cid`) lags the locally
   committed one, and a local-only root is not fetchable with independent
   authority until flush. Checkpoint semantics MUST defer to the
   S3-boundary durability decision and MUST NOT present a locally committed
   root as globally durable.
3. **Retrieval authority that survives the platform.** Fetching the closure
   requires authorization; if the space key and its delegations are
   custodied, that authority dies with the custodian.
4. **Retention.** Ciphertext must still be on the network when the customer
   returns for it; no key arrangement outlives the storage lifecycle or
   payment rail.

Central and regional failure differ: after central failure, a flushed root
remains recoverable if the region is reachable and the customer holds
retrieval authority; spooled writes need the region to survive. Loss of the
region destroys data either way -- v1's accepted single-region durability,
which no exit design can compensate for.

Baseline position: **Tier 1 is what this document specifies and must land
before GA. Tier 2 is provisional everywhere below** -- Option D and the
Tier 2 prerequisites establish feasibility and cost so the promise decision
is made with open eyes, but this RFC commits to none of them. An affirmed
Tier 2 graduates to a follow-up RFC, which must also resolve what this
document only names: rotation-survival for exit grants, identity
presentation, and checkpoint semantics against the async-ack decision.

## Options

### Option A: versioned tenant key release (full-tenant departure)

Hilt releases the tenant's private KEKs, re-wrapped to a customer-supplied
public key, as a documented offboarding ceremony.

Tier-1 rotation retains multiple tenant key versions, with envelopes naming
theirs by `kid`, so the release artifact is a **versioned bundle** covering
**every Hilt-retained version** -- a superset of anything referenced, which
is cheaper than computing "referenced" by envelope scan and costs nothing
(the extras are the tenant's own keys). O(retained versions), small but not
one key. After handoff the customer opens any envelope of theirs, forever,
with no per-object material.

The bundle gets the same format discipline as Option B's manifest: a
**`TenantKeyBundleV1`** IPLD block -- versioned, canonically encoded, bound
to the tenant and the receiving-key fingerprint, exactly one entry per
retained `kid`, each carrying the complete customer-wrap structure
(algorithm, ephemeral public key, wrapped private KEK with byte encoding
specified) -- decodable by public, independent tooling. No raw-key sidecar
exists at any point.

Since a tenant owns many buckets, the exit artifact is a **collection of
per-bucket CARs, each carrying the `TenantKeyBundleV1` block as an
additional root**: the bundle is tiny, so duplication is free and every CAR
is independently decryptable.

- **Aligned with the custody trajectory.** The [unified login
  proposal](https://github.com/fil-one/RFC/pull/14) anticipates
  tenant-space custody transfer; key release is the encryption-layer half
  of the same motion.
- **The scoping problem.** Tenant KEKs are per-*tenant*: the bundle opens
  every envelope the tenant ever wrote, in every bucket and epoch --
  including deleted-but-unpurged blobs and buckets staying on the platform.
  A is only clean for full-tenant departure. (Bucket-scoped release needs
  per-bucket keys -- the encryption RFC's deferred question. Option B
  removes exit as the forcing function, but the blast-radius argument
  stands on its own.)
- **Ceremony requirements.** Releasing tenant KEKs is
  irreversible tenant-wide disclosure; the controls belong here, not in a
  later doc. Minimum: proof of possession of the receiving key -- X25519
  cannot sign, so this MUST be a challenge-decrypt (random challenge
  encrypted to the key, plaintext returned) or a binding to a separately
  authenticated signing DID, and the canary check below satisfies it if run
  before export generation; tenant-wide (not access-key) authorization
  with out-of-band confirmation; an audit record; a decrypted-canary
  validation before the bundle counts as delivered; retry-safe export; and
  no platform-side deletion until the customer validates. The surface
  (console vs UCAN) may stay open; these controls may not.

### Option B: key-manifest CAR export (standard cooperative exit)

The envelope separates payload from key material, so cooperative exit never
touches payload bytes (except actual egress). At export, each part's CEK is
unwrapped and re-wrapped to a customer-supplied X25519 key; the results form
an **`ExitKeyManifest`** -- an IPLD block included in the exit CAR as a
**second root** beside the bucket root. One CAR is the whole artifact;
every blob and CID is untouched.

A bare `blob CID → wrapped bytes` map is not enough: ECDH-ES+A256KW is only
decryptable given the full recipient. Each entry MUST carry the complete
COSE recipient (algorithm, `kid`, export-time ephemeral public key, wrapped
CEK). The manifest MUST be versioned (`ExitKeyManifestV1`), bind the exact
bucket root CID and customer-key fingerprint, contain exactly one entry per
encrypted part reachable from that root and no others, and be decodable by
a public, independent tool. Completeness is verifiable at handoff: the
manifest is content-addressed, its closure checks against the root's, and a
canary decrypt proves the key. Closure validation cannot prove the root is
*current*, so the ceremony MUST have the customer acknowledge the pinned
(bucket, root CID, timestamp) -- a stale but internally valid snapshot must
not pass as the intended exit state.

- **O(parts) key work.** Rewrapping never decrypts, re-encrypts, or
  rewrites ciphertext. Materializing or transferring the CAR is inherently
  O(bytes) -- the CAR *is* the data -- but that cost is common to every
  mechanism and involves no payload encryption or decryption (validation
  still hashes bytes).
- **Scoped under today's per-tenant keys.** One bucket's manifest says
  nothing about any other.
- **Stock crypto only.** (True proxy re-encryption does not retrofit onto
  ECDH-ES+A256KW, and buys nothing anyway: the classic PRE win is a proxy
  that does not hold the key, and Hilt holds it by design.)
- **Still Tier 1.** The ceremony requires Fil One or the region to run.

**Rejected variant -- re-emitted multi-recipient envelopes.** COSE
recipients sit outside the content encryption ([RFC 9052
§5.1](https://datatracker.ietf.org/doc/html/rfc9052.html#section-5.1)), so
adding the customer as a second recipient without re-encrypting is valid,
and FEE supports it. But the header is part of the stored blob: every
re-emitted envelope is a new blob with a new CID -- read and re-hash every
ciphertext byte, rewrite every reference and manifest, recompute MST paths
and root, rebuild the CAR and claims. O(bytes) plus a full DAG rewrite, for
nothing the manifest lacks. **Rejected variant -- payload re-encryption**
(decrypt with the region wrap, re-encrypt to the customer key): strictly
dominated.

### Option C: downgrade the promise

Document that encrypted buckets are portable as ciphertext with
platform-mediated recovery only, and align marketing. There is precedent
for explicitly renegotiating an inherited claim (the deployment proposal's
handling of dropped replication). But credible exit is why the bucket
format is an MST at all; taking this path should be a stated product
decision, not drift.

### Option D: pre-positioned customer key material (provisional -- Tier 2 sketch)

*A feasibility sketch informing the Tier 2 promise decision, not a design
this RFC adopts.*

Necessary -- but not sufficient -- for Tier 2: an enrolled bucket carries a
customer-supplied X25519 public key, and the write path adds a second COSE
recipient to every envelope at PUT time. FEE's multi-recipient model exists
for exactly this; the marginal cost is one wrap per write.

Claiming Tier 2 additionally requires the other three pre-positioned items,
each cheap while the platform is healthy:

- **Customer-held root checkpoints.** Expose *both* roots continuously --
  the local root on write responses, the network-durable root via a read
  API or an async durability receipt at flush -- so checkpoints accrue as a
  side effect of use. Only the network-durable root is a Tier 2 checkpoint;
  the gap between the two is Tier 2's explicit RPO, sized by flush cadence.
- **Durable retrieval authority.** A retrieval delegation from the space to
  the customer's DID, issued at enrollment, held by the customer. This does
  not survive by default: under the unified-login design, validators check
  *current* keys, so space rotation implicitly revokes the delegation and
  re-issue runs through the custodian -- the dependency Tier 2 exists to
  escape. Tier 2 requires one of: two-phase rotation (replacement exit
  delegation delivered and acknowledged before the new key publishes);
  historical-key verification scoped to exit grants; customer custody of
  the space authority; or a dedicated non-rotating recovery verification
  method in the space's DID document that signs only exit grants. The
  artifact must also bundle the DID/PLC operation-log material for offline
  issuer verification -- and holding the log is only preservation;
  validation needs a presentation path (see Prerequisites).
- **Retention** is the bounded residual: under insolvency, exit is a race
  against the payment rail, and Tier 2 marketing should say so.

Notes: enrollment covers only subsequent writes -- enabling Tier 2 on an
existing bucket is a one-time Option B backfill. Responsibility shifts to
the customer: a lost customer key removes the property; a compromised one
exposes data outside platform control and outside cryptoshred (deleting the
region wrap no longer blinds the customer's key holder). This is the
encryption RFC's "customer-held keys" non-goal arriving on a schedule;
nothing in v1's format blocks it.

## Recommendation

**B as the canonical cooperative exit** -- scoped, O(parts) key work, stock
crypto, single artifact. **A for full-tenant departure**, versioned bundle,
ceremony controls up front. **Tier 2: deferred -- planned, not promised.**
Per review, cooperative exit is implemented now and the adversarial tier is
planned even though it is not being built: a follow-up RFC specifies the
pre-positioning set (D plus root checkpoints plus durable retrieval
authority, retention as the bounded residual) as an opt-in feature, and the
customer-facing Tier 2 promise is decided there, not here. Until that RFC
lands, documentation makes no vendor-failure-resistance claim. C for the
whole promise only as an explicit, argued product decision.

## Prerequisites (normative)

Option B is compatible with the shipped envelope format without payload
re-encryption, but it does not work today:

1. **MST ↔ FEE reconciliation.** The MST RFC's `ObjectManifest` predates
   encryption and does not define how a manifest reaches the ordered
   per-part FEE blobs. The exit closure cannot be specified until this
   lands (also the encryption RFC's own "schemas will need to merge" debt).
2. **Stable-root export.** The manifest binds one root; export must pin
   that root *and its reachable closure* against both concurrent mutation
   and physical deletion until the CAR is produced and validated. The MST's
   CAS root makes the pin well-defined, but a pin alone does not stop
   blocks orphaned by mid-export overwrites or deletes from being reaped --
   the closure needs an explicit GC/removal exemption for the export's
   duration.
3. **A Hilt/region rewrap API.** Neither unwrap-and-rewrap-to-external-key
   path exists as a service. Note the [regional key-management
   design](https://github.com/fil-one/RFC/pull/21) interaction: OpenBao
   transit rewraps only between its own versions, so wrapping to an
   external X25519 key passes each CEK through export-process memory -- a
   bounded per-CEK exposure (blast radius of one object, same as a GET) to
   state as an explicit exception to that design's "no CEK in our
   processes" property.
4. **Public export schemas and decoder.** `ExitKeyManifestV1` and
   `TenantKeyBundleV1` as versioned canonical IPLD/CBOR schemas, with at
   least one independent decoder covering both.
5. **(Provisional -- binds the Tier 2 follow-up, not this RFC) An
   identity-presentation path.** Holding the PLC log preserves it, but
   validators *resolve* issuers; they do not accept logs supplied with a
   request. Tier 2 must pick: the retrieval request carries the
   self-certifying log in-band (the only custody-free path -- the directory
   was trusted for availability alone, and presentation removes even that);
   providers pre-cache or mirror logs; or another protocol makes resolution
   independently available.

## Open Questions

- Key scoping: per-tenant vs per-bucket is no longer forced by exit (B
  scopes without it), but the blast-radius argument stands -- decide
  deliberately rather than letting the deferral harden into the default.
- Where the exit-test definition lives: land the MST RFC as the reference
  (revised, or marked as predating encryption), or extract the
  exit-artifact spec here?
- Should an Option A disclosure statement cover soft-deleted
  (cryptoshredded, unpurged) objects, given the bundle can open them
  anyway? (B excludes them naturally: not reachable from the root.)
- Object Lock / legal hold: lock is service state that does not travel;
  does exit of held objects need a compliance gate?
- **(Resolved in review, 2026-08)** Is Tier 2 promised? **Deferred:
  implement Tier 1 now; plan Tier 2 without building it.** A follow-up RFC
  specifies the pre-positioning set and carries the customer-facing promise
  decision; until it lands, no vendor-failure-resistance claim is made.

## Evaluation Criteria

- **Reconstruction, per mechanism.** Provision a tenant, PUT objects
  (single and multipart), produce the exit artifact -- a CAR carrying the
  key-manifest root (B) or the key-bundle root (A) -- and reconstruct
  byte-exact plaintext on a clean machine using only public specs and a
  genuinely independent implementation: no Fil One code, no network.
- **Rotation.** Tests pass for a tenant spanning key epochs. A's bundle
  carries every retained `kid`. B's *output* is `kid`-indifferent but its
  export path is not: the test MUST cover every historical epoch via `kid`
  → archived-KEK resolution, including with the region-wrap path
  unavailable.
- **Scoping.** A one-bucket B manifest provably yields nothing for any
  other bucket.
- **Completeness.** A manifest missing an entry, carrying a duplicate or
  extra, or built against the wrong key is detected before handoff
  completes.
- **Concurrency and retries.** Export under concurrent mutation *and
  deletion* binds the pinned root exactly -- every block of the pinned
  closure is present in the artifact even when orphaned or reaped from
  current state mid-export; a partial failure is retry-safe and converges
  to one valid artifact.
- **Versioned and deleted objects.** Artifacts cover the pinned root's
  closure: historical versions in it decrypt; cryptoshredded objects
  outside it are provably absent from B's manifest.
- The documented promise, the marketing claim, and the passing tests
  agree -- including which tier is promised.

## References

- [RFC: Bucket metadata -- canonical MST and operational database
  (PR #4)](https://github.com/fil-one/RFC/pull/4) -- the credible-exit test
- [RFC: Fil One Object Encryption](./2026-06-fil-one-encryption-design.md) --
  envelope format, key table, rotation tiers (`kid` versioning), deferred
  key-scoping question
- [RFC: Regional security principles and key management (PR
  #21)](https://github.com/fil-one/RFC/pull/21) -- OpenBao transit custody
  the export path must interoperate with
- [RFC: Unified Login & Enrollment
  (PR #14)](https://github.com/fil-one/RFC/pull/14) -- tenant-space custody
  transfer
- [Fil One on Forge deployment
  proposal](./2026-05-filone-forge-deployment-proposal.md) -- precedent for
  explicitly renegotiating an inherited promise
- [FEE: Filecoin Encrypted Envelope (FIP discussion
  #1253)](https://github.com/filecoin-project/FIPs/discussions/1253)
- [RFC 9052 -- COSE: Structures and Process,
  §5.1](https://datatracker.ietf.org/doc/html/rfc9052.html#section-5.1) --
  recipient structure independence from content encryption
