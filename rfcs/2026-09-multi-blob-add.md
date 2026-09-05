# RFC: Multi-blob add

Status: Experimental

## Authors

- [Hannah Howard](https://github.com/hannahhoward)

## Motivation

Uploading one blob today costs the client three POSTs to the upload service plus a receipt poll, and costs the upload service two invocations against the storage node, all serial: `/blob/add`, an HTTP PUT of the bytes to the node, `/ucan/conclude` carrying a self-issued `/http/put` receipt, then `GET /receipt/{cid}` polled at one-second granularity until the `/blob/accept` receipt appears. An object of M blobs pays that chain M times. Client-side parallelism hides some of the latency but none of the message count, and the poll adds up to a second per blob after the bytes have already landed.

Two of those messages carry no information. The `/http/put` receipt is signed with a key derived from the blob digest, so anyone who knows the digest can mint it — sprue's own source comments that it proves nothing — and its only job is to trigger the accept. The receipt poll exists only because the accept completes out of the client's sight, even though it runs synchronously inside the conclude request.

The clients that drive volume know every digest before the first network call. Ingot spools a PutObject to disk and shards it before uploading; any client that pre-shards or erasure-codes an object holds its full digest list up front. Nothing in the protocol lets them say so.

This RFC proposes a batch upload protocol:

- `/blob/batch/add` — announce M blobs in one invocation; the upload service plans placement for all of them and returns, per blob, a selected node plus delegations that let the agent invoke that node directly.
- The agent drives the node legs itself: one container of `/blob/allocate` invocations per node, the PUTs in parallel, one container of `/blob/accept` invocations per node. The node verbs are today's verbs, unchanged.
- `/blob/batch/commit` — return the node-signed accept receipts; the upload service verifies them and registers the blobs.
- `/blob/batch/extend` — replacement placements when a node fails and the spares are exhausted.

Cost for an M-blob object on N nodes: two round trips to the upload service, two to each node, M PUTs in parallel. No `/http/put`, no `/ucan/conclude`, no receipt poll.

## Goals

- Client↔upload-service round trips are independent of blob count: two per batch.
- No storage-node changes. Nodes keep serving `/blob/allocate` and `/blob/accept` exactly as today; only the invoker changes. The protocol deploys against existing piri.
- Every node-side action remains traceable to a client-signed intent: the allocate's `cause` names the client's batch invocation, and the delegation policy binds it.
- Partial success is first-class. Blobs succeed and fail independently; commit is idempotent and resumable, so a straggler commits later under the same batch.
- The self-issued `/http/put` receipt and the receipt poll are removed from the flow entirely.

## Concepts

### Roles

| Name  | Description                                                                  |
| ----- | ---------------------------------------------------------------------------- |
| Agent | A client of the Forge network holding delegations from a space (e.g. Ingot). |
| Sprue | The Forge network upload service.                                            |
| Piri  | A Forge network storage node (a.k.a. storage provider).                      |
| Space | A namespace (cryptographic key pair) that owns content. A bucket is a space. |

### The third delegation regime

The existing upload protocol has two delegation regimes: space-rooted (the space delegates upload capabilities to the agent, which invokes on sprue) and provider-rooted (the node delegates `/blob/allocate` and `/blob/accept` to sprue at registration, and sprue invokes on the node). This RFC adds a third, composed from the second: **re-delegated provider-rooted**.

Sprue already holds the provider's registration delegation for `/blob/allocate` and `/blob/accept`. For each blob it places, sprue re-delegates those two commands to the agent — issuer sprue, audience the agent, subject the **provider DID** — with a policy that pins the invocation arguments to exactly the placement sprue decided:

```jsonc
// Policy on the /blob/allocate delegation (dag-json for readability)
[
  ["==", ".space", "did:key:zSpace"],
  ["==", ".blob.digest", { "/": { "bytes": "EiCcvAfD..." } }],
  ["==", ".blob.size", 1048576],
  ["==", ".cause", { "/": "bafy...batchaddtask" }]
]
```

The agent invokes on the node under the chain provider → sprue → agent. Validation needs nothing new: ucantone's validator checks the subject and applies delegation policies conjunctively at every hop, so the agent can invoke only the announced digest, at the announced size, for the named space and cause, at the node sprue selected, until the delegation expires. Escalation is rejected by the existing validator; there is no issuer allow-list on these handlers to bypass.

The regime grants the agent nothing it does not already have in effect: today's flow produces the same two invocations on the same node at sprue's discretion. Here sprue pre-authorizes precisely that pair and steps off the data path.

The baseline is one delegation pair (allocate, accept) per blob. For large batches sprue MAY compact to one pair per provider, binding the digest set with the policy language's `or` operator. Sprue MAY additionally include a policy-bound `/blob/reject` delegation per blob, letting the agent retire its own parked pieces early instead of waiting for allocation expiry (see [Compatibility](#compatibility)).

### Batch lifecycle

1. **Plan** — `/blob/batch/add`. Sprue selects a node per blob over its provider directory in one pass, mints the delegations, and returns placements. Sprue contacts no node and writes no placement state; blobs already registered in the space are returned as such, with no placement.
2. **Node legs** — for each node holding placements, the agent sends one container batching that node's `/blob/allocate` invocations. Each allocate receipt returns the upload address (or none, when the node already holds the bytes — the warm path, unchanged). The agent PUTs all pieces in parallel, then sends one container per node batching the `/blob/accept` invocations. Accept behaves exactly as today: the node verifies presence, mints its location claim, publishes it synchronously, and enqueues the PDP pipeline.
3. **Commit** — `/blob/batch/commit` relays the node-signed accept receipts. Sprue verifies them and registers each blob against the space.
4. **Repair the tail** — a failed or stalled PUT retries against a spare placement with no sprue call. Only exhausting the spares costs a `/blob/batch/extend`.

### Transport

Batching travels as ucantone container contents: a single HTTP exchange carries a container holding any number of invocations, and receipts ride container metadata the same way. This stack has no ucanto-style effects field, and none is needed.

## Capabilities

### `/blob/batch/add`

* Issuer: Agent
* Audience: Sprue
* Subject: The space

Announces a batch of blobs for upload. Sprue verifies the caller's space authority and quota exactly as `/blob/add` does, plans placement for the whole batch in one selection pass, and returns one placement per blob plus spares, with the minted delegations carried in the response container. Digests already registered in the space are listed in `registered` and receive no placement.

The upload address is not part of the result: it comes from the node itself, in the `/blob/allocate` receipt, as today.

Idempotent: re-announcing a digest is safe — the node keys allocations by `(digest, space)`, and re-registration at commit tolerates the existing entry.

#### Arguments

**IPLD schema**

```ipldsch
type BlobEntry struct {
  digest Bytes # multihash of the blob
  size   Int   # size in bytes
}

type BatchAddArguments struct {
  blobs [BlobEntry]
}
```

<details>
<summary>Go syntax</summary>

```go
type BatchAddArguments struct {
  Blobs []blob.Blob `cborgen:"blobs"` // libforge commands/blob Blob{Digest, Size}
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
  "cmd": "/blob/batch/add",
  "args": {
    "blobs": [
      { "digest": { "/": { "bytes": "EiCcvAfD..." } }, "size": 1048576 },
      { "digest": { "/": { "bytes": "EiDpQrst..." } }, "size": 524288 }
    ]
  }
}
```

#### Result

**IPLD schema**

```ipldsch
type Placement struct {
  digest   Bytes  # multihash of the blob
  provider String # DID of the selected storage node
  proofs   [Link] # delegations in the response container: /blob/allocate, /blob/accept, optionally /blob/reject
}

type BatchAddOK struct {
  placements [Placement] # primary placement per blob
  spares     [Placement] # alternates, usable on failure without a further sprue call
  registered [Bytes]     # digests already registered in this space; no placement issued
}
```

<details>
<summary>Go syntax</summary>

```go
type Placement struct {
  Digest   multihash.Multihash `cborgen:"digest"`
  Provider did.DID             `cborgen:"provider"`
  Proofs   []cid.Cid           `cborgen:"proofs"`
}

type BatchAddOK struct {
  Placements []Placement           `cborgen:"placements"`
  Spares     []Placement           `cborgen:"spares"`
  Registered []multihash.Multihash `cborgen:"registered"`
}
```
</details>

### `/blob/batch/commit`

* Issuer: Agent
* Audience: Sprue
* Subject: The space

Closes out a batch. The request container MUST carry, per committed blob, the `/blob/accept` invocation and its node-signed receipt; the arguments reference each receipt by link. Sprue verifies each pair (see [Commit verification](#commit-verification)), registers each verified blob against the space with the batch-add task as its cause, persists the relayed invocations and receipts to its agent store, and returns a per-blob outcome.

A batch never has to commit whole. Commit registers whatever verifies and names the rest; a later commit under the same `cause` picks up stragglers. Registration tolerates an existing entry, so overlapping or repeated commits converge.

Idempotent: committing an already-registered blob MUST succeed.

#### Arguments

**IPLD schema**

```ipldsch
type CommitEntry struct {
  digest Bytes # multihash of the blob
  accept Link  # the node-signed /blob/accept receipt, carried in the request container
}

type BatchCommitArguments struct {
  cause Link # the /blob/batch/add task this commit closes
  blobs [CommitEntry]
}
```

<details>
<summary>Go syntax</summary>

```go
type CommitEntry struct {
  Digest multihash.Multihash `cborgen:"digest"`
  Accept cid.Cid             `cborgen:"accept"`
}

type BatchCommitArguments struct {
  Cause cid.Cid       `cborgen:"cause"`
  Blobs []CommitEntry `cborgen:"blobs"`
}
```
</details>

#### Result

**IPLD schema**

```ipldsch
type CommitFailure struct {
  digest Bytes
  reason String # named error, e.g. ReceiptNotFound, InvalidReceipt, UnknownProvider
}

type BatchCommitOK struct {
  registered [Bytes]         # digests now (or already) registered in the space
  failed     [CommitFailure] # entries that did not verify; re-commit after repair
}
```

<details>
<summary>Go syntax</summary>

```go
type CommitFailure struct {
  Digest multihash.Multihash `cborgen:"digest"`
  Reason string              `cborgen:"reason"`
}

type BatchCommitOK struct {
  Registered []multihash.Multihash `cborgen:"registered"`
  Failed     []CommitFailure       `cborgen:"failed"`
}
```
</details>

### `/blob/batch/extend`

* Issuer: Agent
* Audience: Sprue
* Subject: The space

Requests replacement placements for blobs whose primary and spare placements have failed. Each entry MAY exclude providers the client has already failed against; sprue MUST NOT place an entry on an excluded provider. The result is a `BatchAddOK` — fresh placements and spares with fresh delegations — and the returned delegations bind `cause` to the **original** batch-add task, so the whole batch commits under one cause.

Idempotent: extending a digest that has since been accepted or registered simply returns no placement for it, listed in `registered`.

#### Arguments

**IPLD schema**

```ipldsch
type ExtendEntry struct {
  digest  Bytes
  exclude [String] # provider DIDs to avoid
}

type BatchExtendArguments struct {
  cause Link # the original /blob/batch/add task
  blobs [ExtendEntry]
}
```

<details>
<summary>Go syntax</summary>

```go
type ExtendEntry struct {
  Digest  multihash.Multihash `cborgen:"digest"`
  Exclude []did.DID           `cborgen:"exclude"`
}

type BatchExtendArguments struct {
  Cause cid.Cid       `cborgen:"cause"`
  Blobs []ExtendEntry `cborgen:"blobs"`
}
```
</details>

#### Result

`BatchAddOK`, as for [`/blob/batch/add`](#blobbatchadd).

### The node legs: `/blob/allocate` and `/blob/accept`, unchanged

The storage-node verbs, their argument types and their results are exactly today's. What changes is the invoker and two argument conventions:

- **`cause`** on `/blob/allocate` is the `/blob/batch/add` task CID (bound by the delegation policy), so every allocation row a node writes names the client invocation that motivated it.
- **`_put`** on `/blob/accept` is vestigial on this path: there is no `/http/put` task to await, because the node itself verified presence when the bytes arrived. The field carries the corresponding `/blob/allocate` task link for wire compatibility, and implementations MUST NOT dereference it. Deprecating the field is left to a future revision of the accept capability.

The warm path is preserved: an allocate receipt with no upload address means the node already holds the bytes, the agent skips that PUT, and the blob's accept rides the accept container with the rest.

## Commit verification

Commit verification is stateless with respect to placement: sprue does not need to remember what it minted, because the proof of placement travels with the commit. For each entry, sprue:

1. Resolves the receipt link in the request container and the invocation it `Ran`.
2. Checks the invocation: command `/blob/accept`, arguments matching the entry's digest and the invocation subject's space, and a proof chain that terminates in a delegation **sprue itself signed** — the placement delegation minted at batch-add, which names the provider, digest, size and cause. Sprue verifying its own signature is the placement check.
3. Checks the receipt: issued and signed by the provider DID that is the invocation's subject, `Ran` matching, and a success result.
4. Registers the blob with `cause` = the batch-add task, and persists the invocation and receipt to the agent store.

An entry failing any check lands in `failed` with a named reason and blocks nothing else.

Implementation note: the container accessor for receipts is a linear scan. Matching M receipts to M entries naively is O(M²); index the container's receipts by `Ran()` once before the loop.

## Failure handling and idempotency

- A failed or stalled PUT retries against a spare placement; no sprue call. Exhausted spares cost one `/blob/batch/extend`.
- A node that accepted before failing still yields a valid receipt; those blobs commit. Only bytes that never reached an accept re-place.
- Allocations that never receive bytes, and spares that go unused, expire on the node's existing allocation-expiry hygiene, with no message required.
- Every capability in this RFC is idempotent: re-adding, re-committing and re-extending converge on the same state, so the retry story for the whole protocol is "re-invoke".

## Security considerations

- **Attenuation is the containment.** The delegation policy pins space, digest, size and cause; ucantone applies policies conjunctively at every hop and rejects escalation. An agent holding a batch's delegations can produce exactly the allocations sprue decided, nowhere else, and only until expiry.
- **No new agent trust.** The agent gains no authority it could not already exercise through `/blob/add`; sprue's placement discretion is unchanged, exercised at plan time instead of invocation time.
- **No new node exposure.** The node serves the same verbs under a chain rooted in its own registration delegation. It cannot tell, and need not care, whether sprue or the agent carried the invocation.
- **Removing a fake signal.** The digest-derived `/http/put` receipt let anyone with a digest assert an upload happened. This protocol replaces it with the node-signed accept receipt as the only evidence of storage — a signature that actually attests something.
- **Replay.** Delegations expire; invocations carry nonces; allocation is keyed `(digest, space)`, so a replayed allocate reproduces the same row rather than a new liability.
- **Resource bounds.** A batch is an amplification lever: one invocation makes sprue run placement and mint delegations for M blobs. Sprue SHOULD enforce a configurable cap on blobs per batch and account batch adds against the space's quota before minting.

## Limits

The cbor-gen list cap (8,192 elements) bounds every list in these schemas. The per-blob delegation baseline puts two to three signed delegations per blob, a few hundred bytes each, in the batch-add response container — order of a megabyte per thousand blobs. Batches large enough to care SHOULD use the per-provider `or`-compacted delegation form. `MaxBlobSize` (256 MiB) is unchanged.

## Compatibility

- **Storage nodes**: no change. The chain provider → sprue → agent validates under existing subject-plus-policy checking.
- **`/blob/add`**: remains, unchanged, for single-blob callers and for streaming clients that cannot know digests up front. A streaming client MAY still batch per flush window — announce the blobs whose digests it has, upload, commit, repeat.
- **Blob removal (RFC 2026-07)**: receipt-chain routing for accepted blobs is preserved, because commit persists the relayed `/blob/accept` invocation — whose subject is the provider — into the agent store, and registration's `cause` reaches it. Parked blobs differ: sprue never saw the allocate, so `/blob/abort` cannot route for a batch-parked blob. The backstop is node-side allocation expiry, which reconciles parked pieces with no message at all; the optional `/blob/reject` delegation in the placement proofs gives an agent early reclaim directly against the node.
- **Read-on-write**: preserved. The node still publishes its location claim synchronously inside accept, and registration completes inside commit, so a successful commit still implies the blob is resolvable. Index and upload registration (`/index/add`, `/upload/add`) follow commit exactly as they follow the accept receipt today, and are out of scope here.

## Future work

- **Fold the node legs into the PUT.** The end state carries the write authorization in the PUT request header and returns the accept receipt, location claim and PDP promise in the response header, collapsing the per-node containers to zero and using `Expect: 100-continue` to recover the warm-path signal. That change touches the node and deserves its own RFC; nothing here precludes it, and the delegation regime is the same.
- **Deprecate `_put`** on `/blob/accept` once no invoker populates it meaningfully.

## Example flow

An object of M blobs placed across N nodes:

| # | Leg | What moves |
|---|-----|------------|
| 1 | agent → sprue | one `/blob/batch/add`: M digests and sizes |
| 2 | sprue → agent | `BatchAddOK`: M placements + spares, delegations in the container |
| 3 | agent → each node, parallel | one container batching that node's `/blob/allocate` invocations; receipts carry upload addresses |
| 4 | agent → nodes, parallel | M HTTP PUTs; stragglers retried against spares |
| 5 | agent → each node, parallel | one container batching that node's `/blob/accept` invocations; receipts carry location claims and PDP promises |
| 6 | agent → sprue | one `/blob/batch/commit` relaying the accept receipts |
| 7 | sprue → agent | `BatchCommitOK`: per-blob outcomes; the S3 200 follows |

Round trips: two to sprue per batch and two per node, against roughly four to sprue **per blob** (three POSTs plus at least one poll GET) and two sprue→node invocations per blob today. For a 100-blob object on 20 nodes: 42 control exchanges, all parallel across nodes, instead of ~600, serial per blob.
