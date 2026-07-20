# RFC: Forge blob removal

Status: Experimental

## Authors

- [Forrest](https://github.com/frrist)

## Motivation

The Forge network can currently only accumulate data. A space can `/blob/add` content and `/upload/add` an index for it, but there is no protocol-level way to walk either action back: deleting an object is a client-side bookkeeping exercise that releases no network storage, and an abandoned upload leaves bytes parked on a storage node forever.

This document proposes the capabilities that close that gap:

- `/blob/remove` — release a space's claim on an **accepted** blob.
- `/blob/abort` — abandon an in-flight upload of a **parked** (never-accepted) blob.
- `/blob/reject` — the storage-node verb that retires a parked blob; the "don't accept" inverse of `/blob/accept`.
- `/upload/remove` — delete an upload's root→shards index entry.

## Goals

- A space can release everything it stores: accepted blobs, parked (in-flight) blobs, and upload indexes.
- Removal is multi-tenant safe. Content addressing deduplicates bytes across spaces, so releasing one space's claim must never destroy bytes another space still claims.
- Removal is multi-tenant live. One space's activity on a digest must never strand another space's ability to unwind its own state.
- Every step is idempotent and retryable, so partial failures are recovered by re-invoking rather than by out-of-band repair.
- Physical deletion is deliberate: bytes are destroyed only after claims are re-verified, and only once any on-chain proof obligations for them are retired.

## Concepts

### Roles

| Name     | Description                                                                    |
| -------- | ------------------------------------------------------------------------------ |
| Agent    | A client of the Forge network holding delegations from a space (e.g. Ingot).   |
| Sprue    | The Forge network upload service.                                              |
| Piri     | A Forge network storage node (a.k.a. storage provider).                        |
| Space    | A namespace (cryptographic key pair) that owns content. A bucket is a space.   |
| Ingot    | An S3 facade typically co-located with a Forge Piri node.                      |

### Blob lifecycle

A blob upload proceeds through `/space/blob/add` (agent → upload service), which drives `/blob/allocate` (upload service → storage node) and an HTTP PUT of the bytes to the node. At this point the blob is **parked**: allocated and uploaded, but not committed. A parked blob has exactly two mutually exclusive exits:

- `/blob/accept` — commit. The node enqueues the blob for PDP aggregation, mints an `/assert/location` claim, and the upload service registers the blob against the space. The blob is now **accepted**, and the only way out is the claim-release path of `/blob/remove`.
- `/blob/reject` — drop. The node releases the allocation and deletes the bytes as if the upload never happened.

Removal is therefore split along the same line as the lifecycle: `/blob/remove` operates on accepted blobs, `/blob/abort`/`/blob/reject` operate on parked ones. The boundary is **per space**: a storage node MUST refuse to reject a blob that the invoking space has accepted (see [`/blob/reject`](#blobreject)), so the two paths cannot race each other into an inconsistent state. Another space's acceptance of the same digest is irrelevant to this guard — each space exits its own lifecycle independently, and shared bytes are protected by claim counting, not by the reject guard.

### Claim counting

Storage nodes key allocations and acceptances by `(digest, space)`. Every removal verb drops **one space's claim**; a node performs physical deletion only once *no* space holds an allocation or acceptance for the digest. This is what makes content-addressed deduplication safe: two tenants storing the same bytes do not observe each other's deletes.

Because the space identifies whose claim is being dropped, it is an explicit argument on every removal command (matching `/blob/allocate` and `/blob/accept`) rather than being inferred from the invocation.

### Two delegation regimes

The removal flow is a two-hop chain — agent → upload service → storage node — and each hop is rooted in a different authority:

**Space-rooted (agent → upload service).** The UCAN subject is the **space DID**. The space issuer delegates `/blob/remove` and `/blob/abort` to the agent, alongside the existing upload capabilities, and the agent invokes with the upload service as audience.

**Provider-rooted (upload service → storage node).** The UCAN subject is the **provider DID**. At registration time the storage node delegates `/blob/remove` and `/blob/reject` to the upload service — the same registration delegation that already carries `/blob/allocate` and `/blob/accept`. The upload service invokes under this delegation, and the space travels in the invocation *arguments*, not the subject.

The same command string (`/blob/remove`) is used on both legs with different subjects. `/blob/abort` exists only on the space-rooted leg; the upload service *translates* it into a provider-rooted `/blob/reject`.

## Capabilities

### `/blob/remove`

Releases a space's claim on an accepted blob. Served at two levels:

| Leg              | Issuer         | Audience     | Subject      | Proof root |
| ---------------- | -------------- | ------------ | ------------ | ---------- |
| Agent → Sprue    | Agent          | Sprue        | The space    | Space      |
| Sprue → Piri     | Sprue          | Piri         | The provider | Provider   |

**At the upload service** (subject = the space): validates the caller's space authority, recovers every storage node holding the blob — the primary via the registration's receipt chain (see [receipt-chain routing](#receipt-chain-routing)) plus any non-failed replicas — forwards `/blob/remove` to each, and deregisters the blob. Forwarding is best-effort: an unreachable node MAY be skipped, because the node-side handler is idempotent and unclaimed allocations expire, so stragglers are reconciled by provider-side hygiene. The upload service SHOULD deregister last, so the receipt chain to the primary remains available for a retry if every forward fails.

**At the storage node** (subject = the provider DID): drops the space's allocation, acceptance and location claim. Bytes are physically deleted only when no space claims the digest anymore — and an accepted blob's bytes MAY additionally be retained until its PDP aggregate root is fully retired on-chain. Physical deletion is always asynchronous: the removal machinery MUST re-verify zero claims before every destructive step.

Idempotent: removing an unknown or already-removed blob MUST succeed.

#### Arguments

**IPLD schema**

```ipldsch
type RemoveArguments struct {
  space  String # DID of the space releasing its claim
  digest Bytes  # multihash of the blob
}
```

<details>
<summary>Go syntax</summary>

```go
type RemoveArguments struct {
  Space  did.DID             `cborgen:"space"`
  Digest multihash.Multihash `cborgen:"digest"`
}
```
</details>

e.g.

```jsonc
// Encoded as dag-json for readability
{
  "iss": "did:key:agent",
  "aud": "did:web:up.forge.example.com",
  "sub": "did:key:space",
  "cmd": "/blob/remove",
  "args": {
    "space": "did:key:space",
    "digest": { "/": { "bytes": "EiCcvAfD..." } }
  }
}
```

#### Result

Successful removal returns a unit result (`{}`).

### `/blob/abort`

* Issuer: Agent
* Audience: Sprue
* Subject: The space

Abandons an in-flight upload of a parked blob. This is the client-facing abandon verb: an upload ends in exactly one of `/blob/accept` (commit) or an abort that the upload service translates into `/blob/reject` on the storage node holding the blob.

A parked blob has no registration or acceptance to look the storage node up by, so the invocation MUST carry `cause` — the link to the `/space/blob/add` task that started the upload. The upload service recovers the node from the cause's receipt chain and forwards a `/blob/reject`. A missing or unknown cause MUST fail with the named error `MissingCause`. The abort mutates no upload-service state (registration happens only at accept, which a parked blob never reached), so a failed abort is safely retryable.

Blobs the space has already accepted MUST be released via `/blob/remove` instead. If the storage node refuses the translated reject with `BlobAccepted`, the upload service SHOULD surface that named failure in the abort receipt (rather than a generic execution error), so the client can distinguish "use `/blob/remove`" from a retryable fault.

Idempotent: aborting an unknown or already-rejected blob MUST succeed.

#### Arguments

**IPLD schema**

```ipldsch
type AbortArguments struct {
  space  String # DID of the space abandoning the upload
  digest Bytes  # multihash of the blob
  cause  Link   # the /space/blob/add task link
}
```

<details>
<summary>Go syntax</summary>

```go
type AbortArguments struct {
  Space  did.DID             `cborgen:"space"`
  Digest multihash.Multihash `cborgen:"digest"`
  Cause  cid.Cid             `cborgen:"cause"`
}
```
</details>

e.g.

```jsonc
// Encoded as dag-json for readability
{
  "iss": "did:key:agent",
  "aud": "did:web:up.forge.example.com",
  "sub": "did:key:space",
  "cmd": "/blob/abort",
  "args": {
    "space": "did:key:space",
    "digest": { "/": { "bytes": "EiCcvAfD..." } },
    "cause": { "/": "bafyreie5b3futh5tibz57xosgv3wetro3ldnkvd6fmvcgezvyrjc6rfrqe" }
  }
}
```

#### Result

Successful abort returns a unit result (`{}`).

### `/blob/reject`

* Issuer: Sprue
* Audience: Piri
* Subject: The provider

The "don't accept" inverse of `/blob/accept`: retires a parked blob. Invoked by the upload service under its registration delegation, typically translating a client `/blob/abort`. The `cause` from the abort is NOT forwarded — it is upload-service routing metadata, meaningless to the node.

The node drops the space's allocation and deletes the bytes once no space holds an allocation or acceptance for the digest. A blob that **the invoking space** has accepted MUST be refused with the named error `BlobAccepted` — a space's accepted blobs are released via `/blob/remove`, never rejected. (Storage nodes MUST write the acceptance before enqueueing the PDP pipeline, so "an acceptance exists" is a conservative superset of "entered the pipeline" and this guard cannot race a concurrent accept in the same space.)

The guard is scoped to the invoking space, NOT the digest. Content-addressed deduplication means another space may have accepted the same bytes while this space's copy is still parked; that acceptance MUST NOT block the reject. The node simply drops this space's allocation and retains the bytes, because the other space still claims them. (A digest-scoped guard would strand the parked allocation: the rejecting space has no acceptance or registration, so its `/blob/remove` would be a no-op at the upload service, leaving allocation expiry as the only way out. See [Alternatives considered](#digest-scoped-blobaccepted-guard).)

Rejecting a space's parked allocation while other spaces hold claims is safe for the same reason `/blob/remove` is: bytes are released only at zero allocations *and* zero acceptances, and physical deletion is deferred to a sweep that re-verifies claims before every destructive step.

Idempotent: rejecting an unknown or already-rejected blob MUST succeed.

#### Arguments

**IPLD schema**

```ipldsch
type RejectArguments struct {
  space  String # DID of the space whose allocation is dropped
  digest Bytes  # multihash of the blob
}
```

<details>
<summary>Go syntax</summary>

```go
type RejectArguments struct {
  Space  did.DID             `cborgen:"space"`
  Digest multihash.Multihash `cborgen:"digest"`
}
```
</details>

#### Result

Successful rejection returns a unit result (`{}`).

### `/upload/remove`

* Issuer: Agent
* Audience: Sprue
* Subject: The space

Deletes an upload's root→shards index entry on the upload service. It does NOT remove the shard blobs — blob removal is a separate per-digest `/blob/remove` decision owned by the client's reference accounting, since shards may be shared between uploads (content addressing). A client deleting an upload and all of its content composes `/upload/remove` with a `/blob/remove` per shard whose reference count reaches zero.

Idempotent: removing an unknown root MUST succeed.

#### Arguments

**IPLD schema**

```ipldsch
type RemoveArguments struct {
  root Link # root CID of the upload
}
```

<details>
<summary>Go syntax</summary>

```go
type RemoveArguments struct {
  Root cid.Cid `cborgen:"root"`
}
```
</details>

#### Result

Successful removal returns a unit result (`{}`).

## Receipt-chain routing

The upload service does not maintain a table mapping blobs to storage nodes. Instead it recovers the node holding a blob by walking stored UCAN receipts:

1. Start from the `/space/blob/add` task (the registration's `cause`, or for parked blobs the `cause` carried in `/blob/abort`).
2. Its receipt's `site` field is a promise awaiting the `/blob/accept` task.
3. The subject of that `/blob/accept` invocation **is** the provider DID.

This keeps the invocation log the single source of truth for placement and is the reason `cause` is a required argument on `/blob/abort`: for a parked blob the receipt chain is the *only* route to the node.

## Idempotency and best-effort forwarding

Every removal handler in the chain is idempotent — unknown digest, unknown root, already-removed and already-rejected all return success. This is the retry story for the whole feature:

- The upload service's fan-out to storage nodes MAY be best-effort. A missed node is not an error; the node-side handlers are idempotent and unclaimed allocations expire.
- Clients SHOULD treat removal as best-effort at the call site (log and continue) while relying on the network's hygiene sweeps to reap stragglers.
- Physical deletion on a storage node is always deferred to an asynchronous sweep that re-verifies zero claims (and, for aggregated pieces, on-chain PDP root retirement) before deleting bytes.

## Example flows

### Delete an object (accepted blob)

An S3 gateway keeps a reference index over the blobs backing its objects. When DeleteObject drops a blob's last reference:

##### 1. Agent

Invoke `/blob/remove` on Sprue:

```jsonc
{
  "iss": "did:key:agent",
  "aud": "did:web:up.forge.example.com",
  "sub": "did:key:space",
  "cmd": "/blob/remove",
  "args": { "space": "did:key:space", "digest": { "/": { "bytes": "..." } } }
}
```

Proofs: space → agent delegation for `/blob/remove`.

##### 2. Sprue

Looks up the blob registration (not found → success, idempotent). Recovers the primary node via the receipt chain and lists replicas. Forwards to each node:

```jsonc
{
  "iss": "did:web:up.forge.example.com",
  "aud": "did:key:provider",
  "sub": "did:key:provider",
  "cmd": "/blob/remove",
  "args": { "space": "did:key:space", "digest": { "/": { "bytes": "..." } } }
}
```

Proofs: provider → upload service registration delegation. Then deregisters the blob.

##### 3. Piri

Deletes the space's location claim, acceptance and allocation. If no other space holds a claim, queues the piece for the asynchronous removal sweep, which retires the PDP root on-chain before deleting bytes.

### Abort a multipart upload (parked blobs)

An S3 gateway implementing multipart upload can park each part's blob at UploadPart (allocate + PUT, no accept) and conclude them into acceptance only at CompleteMultipartUpload. On AbortMultipartUpload (or a session-expiry sweep), for each parked blob:

##### 1. Agent

Invoke `/blob/abort` on Sprue with `cause` = the parked blob's `/space/blob/add` task link.

##### 2. Sprue

Resolves the storage node from the cause's receipt chain and forwards:

```jsonc
{
  "iss": "did:web:up.forge.example.com",
  "aud": "did:key:provider",
  "sub": "did:key:provider",
  "cmd": "/blob/reject",
  "args": { "space": "did:key:space", "digest": { "/": { "bytes": "..." } } }
}
```

No upload-service state changes — a parked blob was never registered.

##### 3. Piri

Refuses with `BlobAccepted` if **this space** has accepted the blob (e.g. a concurrent session in the same space completed with the same content-addressed part — the space now claims the bytes and must release them via `/blob/remove`). Otherwise drops the space's allocation; if another space has accepted or allocated the same digest the bytes are retained for it, and they are deleted only once no space holds an allocation or acceptance.

## Open questions

- **Straggler reaping.** Best-effort forwarding relies on provider-side hygiene to reconcile missed removals (allocation expiry for parked blobs; a reaper for anything else). The reaper for orphaned allocations and missed forwards is not yet specified.
- **Replica consistency.** Removal is forwarded to replicas best-effort; there is no receipt-level confirmation that every replica released its claim. Whether replica removal deserves promises/awaits (as replication itself has) is open.
- **Revocation interaction.** Removing a blob does not revoke outstanding delegations or location claims held by third parties; `/ucan/revoke` exists but the interaction is unspecified.

## Evaluation criteria

- Deleting an object demonstrably releases network storage: bytes are deleted on the storage node once the last claiming space removes and the PDP root retires on-chain.
- Aborted or expired multipart sessions leave no parked blobs behind.
- Multi-tenant safety: a removal by one space never deletes bytes claimed by another.
- Multi-tenant liveness: a space can always unwind its own parked upload, regardless of what other spaces have done with the same digest.
- No stuck states: every partial failure is recoverable by retrying the same idempotent invocation.

## References

- [UCAN spec](https://github.com/ucan-wg/spec)
- [RFC: Forge S3 tenant management](./2026-06-forge-s3-tenant-management.md) — the space/bucket and delegation model these capabilities operate within.
