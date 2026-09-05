# RFC: Forge multi-blob add

Status: Experimental

## Authors

- [Hannah Howard](https://github.com/hannahhoward)

## Motivation

Uploading one blob today costs the client two POSTs to the upload service, an HTTP PUT of the bytes to the storage node, and a receipt poll, and costs the upload service two invocations against the storage node, all serial: `/blob/add`, the PUT, `/ucan/conclude` carrying a self-issued `/http/put` receipt, then `GET /receipt/{cid}` until the `/blob/accept` receipt appears — at least one more round trip per blob, and a second of added latency whenever the receipt is not ready on the first try. An object of M blobs pays that chain M times. Client-side parallelism hides some of the latency but none of the message count.

The clients that drive volume know every digest before the first network call. Ingot spools a PutObject to disk and shards it before uploading; any client that pre-shards or erasure-codes an object holds its full digest list up front. Nothing in the protocol lets them say so. And the protocol has no vocabulary for racing an upload: one node is selected per blob, and if it stalls, the whole blob waits.

This RFC proposes a batch upload protocol:

- `/blob/batch/add` — announce M blobs in one invocation. The upload service plans placement for the whole batch in one pass and returns, per blob, a set of candidate nodes plus delegations that let the agent invoke `/blob/allocate` on those nodes directly.
- The agent drives allocation and upload itself: one container of `/blob/allocate` invocations per node, then PUTs in parallel to as many candidates as it chooses, cancelling the rest once enough land. Over-allocation and long-tail cancellation become placement policy rather than failure handling.
- `/ucan/conclude`, generalized to many receipts. For each blob that landed, the client concludes with the node-signed `/blob/allocate` receipt and the `/http/put` receipt. Sprue verifies the pair, drives `/blob/accept` at each winning node (one container per node), registers the blobs, and returns the accept receipts in the conclude response.
- `/blob/batch/extend` — replacement candidates when a blob exhausts its set.

Cost for an M-blob object: two client↔sprue round trips per batch, one agent↔node round trip per node plus the PUTs, and one sprue↔node round trip per winning node. No receipt poll: the conclude response carries the accept receipts.

## Goals

- Client↔upload-service round trips are independent of blob count: two per batch, plus one `/blob/batch/extend` exchange per repair round.
- No storage-node changes. Nodes keep serving `/blob/allocate` and `/blob/accept` exactly as today; only the allocate invoker changes. The protocol deploys against existing piri.
- Accept authority stays with the upload service. Accept is the liability-creating verb — the node mints and publishes a location claim, enqueues PDP, and the blob leaves the parked state that expiry can reclaim — so it remains invoked by the service, adjacent to the registration that accounts for it.
- The upload service drops out of the per-blob upload sequence. It mints capabilities without contacting a node, and touches nodes only at accept, after the client reports what landed.
- Every node-side action remains traceable to a client-signed intent: the allocate's `cause` names the client's batch invocation, and the delegation policy binds it.
- Partial success is first-class. Blobs succeed and fail independently; conclusion is idempotent and resumable, so a straggler concludes later under the same batch.
- The receipt chain keeps today's shape — `add → allocate → put → accept` — so everything that walks receipts (replay, removal routing) is unchanged.

## Concepts

### Roles

| Name  | Description                                                                  |
| ----- | ---------------------------------------------------------------------------- |
| Agent | A client of the Forge network holding delegations from a space (e.g. Ingot). |
| Sprue | The Forge network upload service.                                            |
| Piri  | A Forge network storage node (a.k.a. storage provider).                      |
| Space | A namespace (cryptographic key pair) that owns content. A bucket is a space. |
| Ingot | An S3 facade typically co-located with a Forge Piri node.                    |

### The third delegation regime: re-delegated allocate

The existing upload protocol has two delegation regimes: space-rooted (the space delegates upload capabilities to the agent, which invokes on sprue) and provider-rooted (the node delegates `/blob/allocate` and `/blob/accept` to sprue at registration, and sprue invokes on the node). This RFC adds a third, composed from the second: sprue re-delegates **`/blob/allocate` only** to the agent — issuer sprue, audience the agent, subject the **provider DID** — with a policy that pins the invocation arguments to exactly the placement sprue decided:

```jsonc
// Policy on the /blob/allocate delegation (dag-json for readability)
[
  ["==", ".space", "did:key:zSpace"],
  ["==", ".blob.digest", { "/": { "bytes": "EiCcvAfD..." } }],
  ["==", ".blob.size", 1048576],
  ["==", ".cause", { "/": "bafy...batchaddtask" }]
]
```

The agent invokes allocate on the node under the chain provider → sprue → agent. Validation needs nothing new: ucantone's validator checks the subject and applies delegation policies conjunctively at every hop, so the agent can allocate only the announced digest, at the announced size, for the named space and cause, at the nodes sprue selected, until the delegation expires. Escalation is rejected by the existing validator; there is no issuer allow-list on these handlers to bypass.

Allocate is the safe verb to hand over: it is idempotent, keyed `(digest, space)`, and an allocation that never receives bytes is parked state the node's existing expiry reclaims. The worst an agent can do with an allocate delegation is create allocations sprue already decided to permit, which then expire. `/blob/accept` is deliberately **not** re-delegated: acceptance creates obligations that do not expire, so it stays a service-invoked verb, issued only after the client reports success and in the same step that registers the blob.

The baseline is one allocate delegation per blob per candidate node. For large batches sprue MAY compact to one delegation per provider, binding the digest set with the policy language's `or` operator. Sprue MAY additionally include a policy-bound `/blob/reject` delegation per placement, letting the agent retire its own parked pieces early instead of waiting for allocation expiry; this is safe because reject refuses blobs the invoking space has accepted (`BlobAccepted`, per the [blob-removal RFC](./2026-07-forge-blob-removal.md)).

### Over-allocation and the winner report

Placement returns more candidates than a blob needs. The agent allocates at its candidates, PUTs to as many as it likes, and stops when enough land — the race is client policy, from "primary first, spare on failure" to "start all, cancel at first." A candidate whose allocate receipt carries no upload address already holds the bytes (the warm path, unchanged) and is an instant winner.

The network learns the outcome from the client, in receipt form. For each blob that landed, the client holds two artifacts:

- the **node-signed `/blob/allocate` receipt** — real evidence that the winning node precommitted to the piece, and the routing for everything that follows: the allocate invocation names the provider (its subject), the space, the digest and the cause, and its proof chain terminates in the delegation sprue itself minted;
- the **`/http/put` receipt** — the completion event, in the protocol's existing conclusion idiom. The put invocation is signed with a key derived from the blob digest, so it attests nothing cryptographically; it is the client's assertion that the bytes landed, and the protocol treats it as exactly that. The real check is the accept that follows: a false conclusion just produces a failed accept at the node.

Losing candidates need no message at all: their parked allocations, and any bytes from PUTs cancelled after completion, expire node-side.

### The client-minted `/http/put`

Today sprue builds the `/http/put` invocation during `/blob/add`, deriving an ed25519 key from the blob digest and embedding it in the invocation metadata so the client can sign the receipt. In this protocol the agent performs the allocate, so the agent builds the put invocation itself — same derivation, same shape:

- key: the multihash's 32-byte digest as an ed25519 seed; issuer = audience = subject = the derived `did:key`;
- arguments: the blob and a `destination` await linking the agent's own `/blob/allocate` task;
- receipt: self-signed with the derived key on PUT success — immediately, with no PUT, when the allocate receipt carried no upload address.

Nothing about the artifact's strength changes — anyone holding the digest could always mint it, which is the deliberate design — only where it is minted. The chain `batch add → allocate → put → accept` is exactly today's chain shape, and `/blob/accept`'s `_put` field awaits a real put task, as today.

### Batch lifecycle

1. **Plan** — `/blob/batch/add`. Sprue selects candidate nodes per blob over its provider directory in one pass, mints the allocate delegations, and returns placements. Sprue contacts no node; blobs already registered in the space are returned as such, with no placement.
2. **Allocate and race** — one container of `/blob/allocate` invocations per candidate node; PUTs in parallel; cancel at enough. The agent mints a put receipt per winner.
3. **Conclude** — one `/ucan/conclude` carrying every winner's receipt pair. Sprue verifies the pairs, coalesces winners by provider, sends one `/blob/accept` container per winning node in parallel, registers each accepted blob with the batch-add task as its cause, and returns the accept receipts and per-blob outcomes in the response.
4. **Repair the tail** — a blob whose candidates all failed gets fresh ones from `/blob/batch/extend` and concludes later under the same batch.

### Transport

Batching travels as ucantone container contents: a single HTTP exchange carries a container holding any number of invocations, and receipts ride container metadata the same way. This stack has no ucanto-style effects field, and none is needed.

## Capabilities

### `/blob/batch/add`

* Issuer: Agent
* Audience: Sprue
* Subject: The space

Announces a batch of blobs for upload. Sprue verifies the caller's space authority and quota exactly as `/blob/add` does, plans placement for the whole batch in one selection pass, and returns candidate placements per blob with the minted delegations carried in the response container. The result names one primary candidate per blob in `placements` and additional candidates in `spares`; how many spares to race and how many to hold in reserve is client policy. Digests already registered in the space are listed in `registered` and receive no placement.

The upload address is not part of the result: it comes from the node itself, in the `/blob/allocate` receipt, as today.

Idempotent: re-announcing a digest is safe — allocation is keyed `(digest, space)` on the node, and registration at conclusion tolerates the existing entry.

#### Arguments

**IPLD schema**

```ipldsch
type Blob struct {
  digest Bytes # multihash of the blob
  size   Int   # size in bytes
}

type BatchAddArguments struct {
  blobs [Blob]
}
```

<details>
<summary>Go syntax</summary>

```go
type BatchAddArguments struct {
  Blobs []blob.Blob `cborgen:"blobs"` // reuses libforge commands/blob Blob{Digest, Size}
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
  provider String # DID of the candidate storage node
  proofs   [Link] # delegations in the response container: /blob/allocate, optionally /blob/reject
}

type BatchAddOK struct {
  placements [Placement] # primary candidate per blob
  spares     [Placement] # additional candidates; race or hold in reserve, client's choice
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

### `/ucan/conclude`, generalized

* Issuer: Agent
* Audience: Sprue
* Subject: The space

The existing conclusion command, extended to carry many receipts. The arguments gain an optional plural field; a server implementing this RFC MUST accept both forms, and the legacy single-receipt flow is untouched.

The multi-receipt form also tightens authorization, deliberately. Today's conclude is effectively self-authorized — the client issues it with itself as subject, and the handler checks only that the named receipt corresponds to a stored invocation. The multi form MUST carry the space as subject, proven by the agent's space-rooted chain, and registration MUST bind to the space the placement delegation pins, which MUST equal the invocation subject. This closes conclusion of another space's uploads by anyone who learns a task CID.

For each concluded `/http/put` receipt, the request container MUST make the winner's pair resolvable: the put invocation, and — via its `destination` await — the `/blob/allocate` invocation and its node-signed receipt. The handler resolves the pair from the request container; resolution from sprue's agent store remains for the legacy single form, whose artifacts sprue itself wrote.

Sprue dispatches each receipt whose `Ran` command has a registered conclusion handler; other receipts in the container are context. The `/http/put` handler, per conclusion:

1. follows `destination` to the allocate invocation and receipt;
2. verifies the allocate invocation's proof chain terminates in a **placement delegation sprue itself signed**, whose audience is the concluding agent and whose policy pins the provider, space, digest, size and cause — and that the pinned space equals the conclude subject. Sprue verifying its own signature is the placement check, and it requires no stored state. Verification checks signature and policy bindings, **not** freshness: the node-signed allocate receipt attests that the full chain, expiry included, validated at execution time. Provider registration and liveness are checked at accept time;
3. verifies the node's signature on the allocate receipt;
4. enforces the placement bound: at most the placement policy's replica count of providers — one, unless placement says otherwise — is accepted per `(digest, space)`. The first verified pair wins; surplus pairs return a `SurplusWinner` outcome with no accept, and registration records the accepted provider. This bound, not delegation revocation, is what caps winners — an extend's exclusions revoke nothing;
5. invokes `/blob/accept` at the provider — the verification the protocol actually relies on: the node confirms it holds the bytes. Within one conclude request sprue SHOULD coalesce accepts by provider into one container per node, sent in parallel, each under a mandatory per-node deadline so no stalled node delays the response;
6. on accept success, registers the blob against the space with the batch-add task as `cause`, and persists the relayed invocations and receipts to its agent store.

The multi-receipt form returns per-conclusion outcomes, with the accept receipts in the response container — this is what retires the receipt poll. A conclusion failing any check lands in the outcome list with a named reason and blocks nothing else.

A batch never has to conclude whole. Conclusion registers whatever verifies; a later conclude under the same batch picks up stragglers. Registration and acceptance tolerate repeats, so overlapping or repeated conclusions converge. Clients SHOULD conclude within the delegation lifetime: node-side allocation expiry can reap unconcluded bytes, and the node's deadline is not visible to sprue.

Idempotent: concluding an already-accepted blob MUST succeed.

Implementation note: the container accessor for receipts is a linear scan. Matching M conclusions to their container artifacts naively is O(M²); index the container's contents by link and by `Ran()` once before the loop.

#### Arguments

**IPLD schema**

```ipldsch
type ConcludeArguments struct {
  receipt  optional Link   # legacy single form
  receipts optional [Link] # the /http/put receipts being concluded
}
```

<details>
<summary>Go syntax</summary>

```go
type ConcludeArguments struct {
  Receipt  *cid.Cid  `cborgen:"receipt,omitempty"`
  Receipts []cid.Cid `cborgen:"receipts,omitempty"`
}
```
</details>

e.g.

```jsonc
// Encoded as dag-json for readability; the referenced put receipts,
// put invocations, allocate invocations and allocate receipts travel
// in the same container
{
  "iss": "did:key:agent",
  "aud": "did:web:up.forge.example.com",
  "sub": "did:key:space",
  "cmd": "/ucan/conclude",
  "args": {
    "receipts": [
      { "/": "bafyreia...putrcpt1" },
      { "/": "bafyreib...putrcpt2" }
    ]
  }
}
```

#### Result

The legacy single form keeps its current result. The multi form returns:

**IPLD schema**

```ipldsch
type ConcludeOutcome struct {
  receipt    Link            # the concluded /http/put receipt
  registered Bool
  accept     optional Link   # the node-signed /blob/accept receipt, in the response container
  reason     optional String # named error, e.g. InvalidAllocateReceipt, UnknownPlacement,
                             # SurplusWinner, AcceptRefused, AcceptTimeout
}

type BatchConcludeOK struct {
  outcomes [ConcludeOutcome]
}
```

<details>
<summary>Go syntax</summary>

```go
type ConcludeOutcome struct {
  Receipt    cid.Cid  `cborgen:"receipt"`
  Registered bool     `cborgen:"registered"`
  Accept     *cid.Cid `cborgen:"accept,omitempty"`
  Reason     string   `cborgen:"reason,omitempty"`
}

type BatchConcludeOK struct {
  Outcomes []ConcludeOutcome `cborgen:"outcomes"`
}
```
</details>

### `/blob/batch/extend`

* Issuer: Agent
* Audience: Sprue
* Subject: The space

Requests replacement candidates for blobs whose existing candidates have failed. Each entry MAY exclude providers the client has already failed against; sprue MUST NOT place an entry on an excluded provider. The result is a `BatchAddOK` — fresh candidates with fresh delegations — and the returned delegations bind `cause` to the **original** batch-add task, so the whole batch concludes under one cause.

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

The storage-node verbs, their argument types and their results are exactly today's. What changes:

- **`/blob/allocate`** is invoked by the agent under the re-delegation, with `cause` = the `/blob/batch/add` task CID (bound by the delegation policy), so every allocation row a node writes names the client invocation that motivated it.
- **`/blob/accept`** is invoked by sprue under its registration delegation, as today, with `_put` awaiting the client-minted `/http/put` task from the concluded pair.

## Registration durability

Accept at the node and registration at sprue are two steps, and the gap between them is where accepted-but-unregistered state — a node holding non-expiring, PDP-enqueued bytes no space is registered for — would otherwise leak: an accept that times out after the node committed, a sprue crash between accept and registration, a client that disconnects mid-conclude. The protocol closes the gap by making completion sprue's obligation rather than the request's:

- Sprue MUST journal each verified conclusion durably **before** invoking accept, and MUST drive the journaled pair through accept and registration to completion regardless of the request — the handler runs to completion on client disconnect, and a reconciliation sweep re-drives journaled pairs orphaned by a crash or timeout.
- The outcome vocabulary keeps the two failure meanings apart. `AcceptRefused` means the node rejected the accept: the blob is not stored there, and re-placing via `/blob/batch/extend` is the right response. `AcceptTimeout` means the outcome is unknown and sprue owns reconciliation: the client MUST NOT re-place the blob on this signal — re-concluding later is safe and returns the settled outcome.

With the journal, a verified conclusion either completes or is retried by sprue itself; no client behavior, node stall, or crash strands an accept unregistered beyond the sweep interval.

## Failure handling and idempotency

- A failed or stalled PUT races another candidate; no sprue call. An exhausted candidate set costs one `/blob/batch/extend`.
- An `AcceptRefused` outcome sends the blob back through extend and a later conclude; an `AcceptTimeout` outcome is sprue's to reconcile (see [Registration durability](#registration-durability)).
- Allocations that never receive bytes, and bytes from cancelled or losing PUTs, expire on the node's existing allocation-expiry hygiene, with no message required.
- Every capability in this RFC is idempotent: re-adding, re-concluding and re-extending converge on the same state, so the retry story for the whole protocol is "re-invoke".

## Security considerations

- **Attenuation.** The allocate delegation's policy pins space, digest, size and cause; ucantone applies policies conjunctively at every hop and rejects escalation. An agent holding a batch's delegations can produce exactly the parked allocations sprue decided to permit, nowhere else, and only until expiry.
- **Accept authority never moves.** Accept — the step that mints a published location claim, enqueues PDP, and exits the expiry-reclaimable state — is invoked only by sprue, only after a client conclusion, and its completion is journaled (see [Registration durability](#registration-durability)), so no client can manufacture an accepted-but-unregistered blob.
- **No new agent trust.** The agent gains no authority it could not already exercise through `/blob/add`; sprue's placement discretion is unchanged, exercised at plan time instead of invocation time.
- **Evidence strength.** The put receipt is a client assertion — its digest-derived key was always mintable by any holder of the digest — and the protocol treats it as one. The node-signed allocate receipt is the signed artifact, and accept succeeding at the node is the verification the protocol relies on.
- **Replay.** Delegations expire; invocations carry nonces; allocation is keyed `(digest, space)`, so a replayed allocate reproduces the same row rather than a new liability; re-conclusion and re-registration converge, and the conclude subject binding stops cross-space conclusion of a leaked pair.
- **Resource bounds.** A batch amplifies work: one invocation makes sprue run placement and mint delegations for M blobs, and one conclude makes it verify and fan out accepts for M winners. Sprue SHOULD enforce configurable caps on blobs per batch and conclusions per conclude. Quota SHOULD be taken as a hold with a TTL equal to the delegation expiry, released on expiry and reconciled at registration — an outright debit leaks on abandoned batches, and a bare check over-commits across parallel batches.

## Limits

The cbor-gen list cap (8,192 elements) bounds every list in these schemas. The per-candidate delegation baseline puts one or two signed delegations per candidate, a few hundred bytes each, in the batch-add response container — order of a megabyte per thousand candidates. Batches large enough to care SHOULD use the per-provider `or`-compacted delegation form. `MaxBlobSize` (256 MiB) is unchanged, enforced node-side at allocate as today.

## Compatibility

- **Storage nodes**: no change. The provider → sprue → agent allocate chain validates under existing subject-plus-policy checking.
- **`/blob/add`**: remains, unchanged, for single-blob callers and for streaming clients that cannot know digests up front. A streaming client MAY still batch per flush window — announce the blobs whose digests it has, race, conclude, repeat.
- **`/ucan/conclude`**: the plural field is additive; existing single-receipt clients and the legacy flow are untouched.
- **Receipt-chain consumers**: the persisted chain — `batch add → allocate → put → accept` — has today's shape, so the replay path and the [blob-removal RFC](./2026-07-forge-blob-removal.md#receipt-chain-routing)'s receipt-chain routing work unchanged for concluded blobs. Never-concluded blobs differ: sprue never saw their allocates, so `/blob/abort` cannot route for them. The backstop is node-side allocation expiry, which reconciles parked pieces with no message at all; the optional `/blob/reject` delegation in the placement proofs gives an agent early reclaim directly against the node.
- **Read-on-write**: preserved. The node still publishes its location claim synchronously inside accept, and registration completes inside the conclude request, so a successful conclusion still implies the blob is resolvable. Index and upload registration (`/index/add`, `/upload/add`) follow conclusion exactly as they follow the accept receipt today, and are out of scope here.

## Future work

- **Fold the node legs into the PUT.** The end state carries the write authorization in the PUT request header and returns the accept receipt, location claim and PDP promise in the response header, collapsing the per-node exchanges to zero and using `Expect: 100-continue` to recover the warm-path signal. That change touches the node and deserves its own RFC; nothing here precludes it, and the delegation regime is the same.

## Example flow

An object of M blobs raced across N candidate nodes:

| # | Leg | What moves |
|---|-----|------------|
| 1 | agent → sprue | one `/blob/batch/add`: M digests and sizes |
| 2 | sprue → agent | `BatchAddOK`: candidates per blob, allocate delegations in the container |
| 3 | agent → each node, parallel | one container batching that node's `/blob/allocate` invocations; receipts carry upload addresses (or none — warm, instant winner) |
| 4 | agent → nodes, parallel | HTTP PUTs to chosen candidates; cancel at enough; mint a put receipt per winner |
| 5 | agent → sprue | one `/ucan/conclude` carrying every winner's allocate + put receipt pair |
| 6 | sprue → each winning node, parallel | one container batching that node's `/blob/accept` invocations |
| 7 | sprue → agent | `BatchConcludeOK`: per-blob outcomes, accept receipts in the container; the S3 200 follows |

Round trips: two to sprue per batch, one per node from the agent, one per winning node from sprue — against three exchanges with sprue per blob (`/blob/add`, `/ucan/conclude`, at least one poll GET) and two sprue→node invocations per blob today. For a 100-blob object on 20 nodes: at most 42 control exchanges, all parallel across nodes, instead of ~500, serial per blob. Byte PUTs are excluded from both counts.

## Open questions

- Default and maximum sizes for the batch, conclude and candidate-set caps.
- Whether `/blob/reject` delegations ship in placement proofs by default or on request.
- The per-node accept deadline and the reconciliation sweep interval.
- Whether the per-provider `or`-compacted delegation form ships in v1 or waits for a batch size that needs it.

## Evaluation criteria

- Control exchanges for an M-blob object drop from ~5M to 2 + 2N, measured on ingot's sharded-object upload.
- End-to-end upload latency for a multi-blob object improves by at least the removed poll and serialization overhead, measured against the current guppy/ingot path.
- No growth in stranded state: accepted-but-unregistered blobs stay at zero under fault injection (accept timeout, sprue restart mid-conclude, client disconnect mid-conclude).
- The legacy single-blob flow is byte-identical on the wire before and after deployment.

## References

- [Forge blob removal RFC](./2026-07-forge-blob-removal.md) — blob lifecycle, `/blob/release`, `/blob/reject`, receipt-chain routing.
- [UCAN specification](https://github.com/ucan-wg/spec) — delegation, invocation, policy language.
