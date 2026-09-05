# RFC: Forge multi-blob add

Status: Experimental

## Authors

- [Hannah Howard](https://github.com/hannahhoward)

## Motivation

Uploading one blob today costs the client two POSTs to the upload service, an HTTP PUT of the bytes to the storage node, and a receipt poll, and costs the upload service two invocations against the storage node, all serial: `/blob/add`, the PUT, `/ucan/conclude` carrying a self-issued `/http/put` receipt, then `GET /receipt/{cid}` until the `/blob/accept` receipt appears — at least one more round trip per blob, and a second of added latency whenever the receipt is not ready on the first try. An object of M blobs pays that chain M times. Client-side parallelism hides some of the latency but none of the message count.

The clients that drive volume know every digest before the first network call. Ingot spools a PutObject to disk and shards it before uploading; any client that pre-shards or erasure-codes an object holds its full digest list up front. Nothing in the protocol lets them say so. And the protocol has no vocabulary for racing an upload: one node is selected per blob, and if it stalls, the whole blob waits.

This RFC proposes a batch upload protocol:

- `/blob/batch/add` — announce M blobs in one invocation. The upload service plans placement for the whole batch in one pass and returns, per blob, a set of candidate nodes plus delegations that let the agent invoke `/blob/allocate` and `/blob/accept` on those nodes directly.
- The agent drives the entire node exchange itself: one container of `/blob/allocate` invocations per node, PUTs in parallel to as many candidates as it chooses, cancelling the rest once enough land, then one container of `/blob/accept` invocations per winning node. The accept receipts — location claim and PDP promise included — are in the agent's hands when the accept round completes. Nothing is polled and nothing is awaited from the upload service.
- Registration travels as receipt delivery over `/ucan/conclude`, which stays what it is today: an open conclusion mechanism that anyone holding a receipt may invoke. The storage node SHOULD conclude each accept it executes; the agent SHOULD deliver its collected receipts as well, batched. The upload service registers idempotently from whichever delivery arrives first.
- `/blob/batch/extend` — replacement candidates when a blob exhausts its set.

Cost for an M-blob object: one synchronous round trip to the upload service, one allocate and one accept round trip per node plus the PUTs, and asynchronous receipt deliveries that never sit on the completion path.

## Goals

- One synchronous client↔upload-service exchange per batch, independent of blob count, plus one `/blob/batch/extend` exchange per repair round.
- No polling and no conclusion wait: the agent holds every receipt it needs — accept, location claim, PDP promise — directly from the nodes, at the end of the node exchange.
- `/ucan/conclude` remains an intentionally open delivery mechanism; the upload service trusts what the delivered chain proves and nothing about who delivered it.
- Reporting duty sits with the party that bears the loss. An accepted blob nobody registers is unpaid bytes on the node, so the node concludes its own accepts; the agent's delivery is the second, independent path.
- Every acceptance has a protocol-visible registration horizon, derived from the delegation that authorized it, so acceptance, registration and reclaim can never disagree about whether a blob is live.
- The upload service drops off the upload path entirely after planning. It mints capabilities without contacting a node and never fans out to nodes at all.
- Every node-side action remains traceable to a client-signed intent: the allocate's `cause` names the client's batch invocation, and the delegation policy binds it.
- Partial success is first-class. Blobs succeed and fail independently; delivery is idempotent and repeatable, so stragglers converge under the same batch.
- The receipt chain keeps today's shape — `add → allocate → put → accept` — so everything that walks receipts (replay, removal routing) is unchanged, and gains a sprue-signed terminal link: registration itself becomes a chain-visible `/blob/register` receipt.

## Concepts

### Roles

| Name  | Description                                                                  |
| ----- | ---------------------------------------------------------------------------- |
| Agent | A client of the Forge network holding delegations from a space (e.g. Ingot). |
| Sprue | The Forge network upload service.                                            |
| Piri  | A Forge network storage node (a.k.a. storage provider).                      |
| Space | A namespace (cryptographic key pair) that owns content. A bucket is a space. |
| Ingot | An S3 facade typically co-located with a Forge Piri node.                    |

### The third delegation regime: re-delegated placement

The existing upload protocol has two delegation regimes: space-rooted (the space delegates upload capabilities to the agent, which invokes on sprue) and provider-rooted (the node delegates `/blob/allocate` and `/blob/accept` to sprue at registration, and sprue invokes on the node). This RFC adds a third, composed from the second: sprue re-delegates `/blob/allocate` and `/blob/accept` to the agent — issuer sprue, audience the agent, subject the **provider DID** — with a policy that pins the invocation arguments to exactly the placement sprue decided:

```jsonc
// Policy on each placement delegation (dag-json for readability)
[
  ["==", ".space", "did:key:zSpace"],
  ["==", ".blob.digest", { "/": { "bytes": "EiCcvAfD..." } }],
  ["==", ".blob.size", 1048576],
  ["==", ".cause", { "/": "bafy...batchaddtask" }]
]
```

The agent invokes the node verbs under the chain provider → sprue → agent. Validation needs nothing new: ucantone's validator checks the subject and applies delegation policies conjunctively at every hop, so the agent can act only on the announced digest, at the announced size, for the named space and cause, at the nodes sprue selected, until the delegation expires. Escalation is rejected by the existing validator; there is no issuer allow-list on these handlers to bypass.

Two facts of each placement delegation do double duty later. Its metadata carries the blob's **replica count** (one, unless placement says otherwise), which bounds registration. Its **expiry** defines the registration horizon: the chain it roots can register only until that expiry plus a protocol grace period (see [Registration by receipt delivery](#registration-by-receipt-delivery)).

Handing over allocate is safe: it is idempotent, keyed `(digest, space)`, and an allocation that never receives bytes is parked state the node's existing expiry reclaims. Handing over accept is safe because registration follows the receipt rather than the invoker: acceptance creates the durable obligations — a published location claim, a PDP enqueue — and what keeps those from dangling is not who invoked accept but that the accept receipt reaches the upload service inside the horizon, which the node itself has the strongest incentive to ensure.

The baseline is one delegation pair per blob per candidate node. For large batches sprue MAY compact to one pair per provider, binding the digest set with the policy language's `or` operator. Sprue MAY additionally include a policy-bound `/blob/reject` delegation per placement, letting the agent retire its own parked pieces early instead of waiting for allocation expiry; this is safe because reject refuses blobs the invoking space has accepted (`BlobAccepted`, per the [blob-removal RFC](./2026-07-forge-blob-removal.md)).

### Over-allocation and the race

Placement returns more candidates than a blob needs. The agent allocates at its candidates, PUTs to as many as it likes, and stops when enough land — the race is client policy, from "primary first, spare on failure" to "start all, cancel at first." A candidate whose allocate receipt carries no upload address already holds the bytes (the warm path, unchanged) and is an instant winner.

The agent then invokes `/blob/accept` at winners, and MUST accept at no more than the blob's replica count of them: acceptance is the durable state, and registration will not pay for more. A retried accept that lands late anyway can still leave one extra acceptance; that converges rather than corrupts — the surplus node learns its status from its own delivery outcome and reclaims, readers who hit its stale location claim retry the registered node's claim, and the agent reconciles its index from the delivery outcomes.

Losing candidates need no message at all: their parked allocations, and any bytes from PUTs cancelled after completion, expire node-side.

### The client-minted `/http/put`

Today sprue builds the `/http/put` invocation during `/blob/add`, deriving an ed25519 key from the blob digest and embedding it in the invocation metadata so the client can sign the receipt. In this protocol the agent drives the node exchange, so the agent builds the put invocation itself — same derivation, same shape:

- key: the multihash's 32-byte digest as an ed25519 seed; issuer = audience = subject = the derived `did:key`;
- arguments: the blob and a `destination` await linking the agent's own `/blob/allocate` task;
- receipt: self-signed with the derived key on PUT success — immediately, with no PUT, when the allocate receipt carried no upload address.

Nothing about the artifact's strength changes — anyone holding the digest could always mint it, which is the deliberate design — only where it is minted. The agent's `/blob/accept` invocation links the put task in `_put`, so the chain `batch add → allocate → put → accept` is exactly today's chain shape.

### Batch lifecycle

1. **Plan** — `/blob/batch/add`. Sprue selects candidate nodes per blob over its provider directory in one pass, mints the placement delegations, and returns placements. Sprue contacts no node; blobs already registered in the space are returned as such, with no placement.
2. **Allocate, race, accept** — one container of `/blob/allocate` invocations per candidate node; PUTs in parallel; cancel at enough; one container of `/blob/accept` invocations per winning node. The agent ends the node exchange holding node-signed accept receipts, location claims and PDP promises for every blob that landed. This is the client's completion point.
3. **Deliver** — the node SHOULD invoke `/ucan/conclude` on sprue for each accept it executes; the agent SHOULD deliver its collected accept receipts in one batched `/ucan/conclude`. Sprue verifies each delivered chain, mints the `/blob/register` receipt idempotently, and answers with per-receipt outcomes linking it.
4. **Repair the tail** — a blob whose candidates all failed gets fresh ones from `/blob/batch/extend` and its receipts are delivered later under the same batch.

### Transport

Batching travels as ucantone container contents: a single HTTP exchange carries a container holding any number of invocations, and receipts ride container metadata the same way. This stack has no ucanto-style effects field, and none is needed.

## Capabilities

### `/blob/batch/add`

* Issuer: Agent
* Audience: Sprue
* Subject: The space

Announces a batch of blobs for upload. Sprue verifies the caller's space authority and quota exactly as `/blob/add` does, plans placement for the whole batch in one selection pass, and returns candidate placements per blob with the minted delegations carried in the response container. Candidates come only from nodes that advertised batch support at registration (see [the node legs](#the-node-legs-bloballocate-and-blobaccept-unchanged-verbs)); sprue MUST NOT hand batch placements to a node that has not. The result names one primary candidate per blob in `placements` and additional candidates in `spares`; how many spares to race and how many to hold in reserve is client policy. Digests already registered in the space are listed in `registered` and receive no placement.

The upload address is not part of the result: it comes from the node itself, in the `/blob/allocate` receipt, as today.

Idempotent: re-announcing a digest is safe — allocation is keyed `(digest, space)` on the node, and registration tolerates the existing entry.

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
  proofs   [Link] # delegations in the response container: /blob/allocate, /blob/accept, optionally /blob/reject
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

* Issuer: any principal holding the receipts — typically the storage node or the agent
* Audience: Sprue
* Subject: the issuer (self-issued, as today)

The existing conclusion command, extended to carry many receipts. Conclude is deliberately an **open receipt-delivery mechanism**: it carries no authority of its own, anyone may invoke it, and the upload service acts only on what the delivered artifacts prove. Piri already uses it outbound this way, delivering its replica-transfer receipts to the upload service. The arguments gain an optional plural field — exactly one of `receipt` or `receipts` MUST be present — and a server implementing this RFC MUST accept both forms; the legacy single-receipt flow is untouched.

For a delivered `/blob/accept` receipt to register a blob, the request container MUST make the full chain resolvable: the accept invocation, its node-signed receipt, the put invocation its `_put` field awaits, and the `/blob/allocate` invocation that put's `destination` awaits. Deliverers SHOULD include the allocate receipt as well, so sprue persists a complete chain for receipt-chain consumers.

Sprue dispatches each delivered receipt whose `Ran` command has a registered conclusion handler; other artifacts in the container are context. The new `/blob/accept` conclusion handler, per delivery:

1. verifies the accept receipt is signed by the provider DID that is the accept invocation's subject, with `Ran` matching and a success result;
2. verifies the accept invocation's proof chain terminates in a **placement delegation sprue itself signed**, whose subject is the provider and whose policy pins the space, digest, size and cause. Sprue verifying its own signature is the placement check, and it requires no stored state;
3. enforces the registration horizon: a chain whose placement delegation expired more than the protocol grace period ago MUST be refused with an `Expired` outcome. Within the horizon, freshness is not otherwise rechecked — the node-signed receipt attests that the chain, expiry included, validated at execution time;
4. enforces the registration bound: at most the replica count the placement delegation's metadata names — one, absent the field — is registered per `(digest, space)`. The earliest delivered acceptances win, up to the bound; further acceptances of the same blob at other providers return a `Surplus` outcome and are not registered;
5. checks the tombstones (below): a chain matching one returns a `Removed` outcome instead of registering;
6. registers the blob by minting one [`/blob/register`](#blobregister) task per winning acceptance and issuing its receipt, debits the space's quota, and persists the delivered chain and the registration alongside it.

Registration and removal MUST serialize per `(digest, space)`, and two tombstones make their races come out right in both directions. Removing a **registered** blob retires its `/blob/register` tasks and tombstones its `(digest, space, cause)`: a redelivered chain for that cause returns `Removed`, while a fresh batch's chain — new cause — registers. Removing a digest with **no registration yet** writes a pending tombstone stamped with the removal time, held for the length of the registration horizon: it blocks chains whose placement delegation was issued before the stamp (the batch the client walked away from) and passes chains minted after it (a re-upload). Both checks read only the delivered delegation, so verification stays stateless.

Delivery is idempotent and unordered: the `/blob/register` task is built with no nonce, so the node's conclusion and the agent's conclusion of the same accept converge on the identical task CID and one receipt, and a repeat delivery returns the same registration link. A delivery failing any check lands in the outcome list with a named reason and blocks nothing else, and a `Surplus` outcome still carries the blob's registration links, so the deliverer learns the registered providers directly. The legacy `/http/put` conclusion handler is unchanged and continues to serve the single-blob flow.

Implementation note: the container accessor for receipts is a linear scan. Matching M deliveries to their container artifacts naively is O(M²); index the container's contents by link and by `Ran()` once before the loop.

#### Arguments

**IPLD schema**

```ipldsch
type ConcludeArguments struct {
  receipt  optional Link   # legacy single form
  receipts optional [Link] # the receipts being delivered
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
// Encoded as dag-json for readability; the referenced accept receipts,
// accept invocations, put invocations and allocate invocations travel
// in the same container
{
  "iss": "did:key:zPiriNode",
  "aud": "did:web:up.forge.example.com",
  "sub": "did:key:zPiriNode",
  "cmd": "/ucan/conclude",
  "args": {
    "receipts": [
      { "/": "bafyreia...acceptrcpt1" },
      { "/": "bafyreib...acceptrcpt2" }
    ]
  }
}
```

#### Result

The legacy single form keeps its current result. The multi form returns:

**IPLD schema**

```ipldsch
type ConcludeOutcome struct {
  receipt      Link            # the delivered receipt
  registration optional Link   # the /blob/register receipt, in the response container
  reason       optional String # named outcome when this delivery registered nothing,
                               # e.g. Surplus, Removed, Expired, InvalidReceipt, UnknownPlacement
}

type BatchConcludeOK struct {
  outcomes [ConcludeOutcome]
}
```

<details>
<summary>Go syntax</summary>

```go
type ConcludeOutcome struct {
  Receipt      cid.Cid  `cborgen:"receipt"`
  Registration *cid.Cid `cborgen:"registration,omitempty"`
  Reason       string   `cborgen:"reason,omitempty"`
}

type BatchConcludeOK struct {
  Outcomes []ConcludeOutcome `cborgen:"outcomes"`
}
```
</details>

### `/blob/register`

* Issuer: Sprue
* Audience: Sprue
* Subject: Sprue

The terminal link of the upload chain, and a virtual task in the tradition of `/http/put`: nothing serves it. Sprue mints it inside the `/blob/accept` conclusion handler when a delivery verifies, issues the receipt itself, and persists both with the delivered chain. The receipt is the canonical, sprue-signed record that a blob is registered — the artifact billing, index reconciliation and disputes start from — and the registration time is the receipt's issue time.

The task is built with no nonce, so its CID is deterministic over its arguments: every delivery of the same acceptance names the identical task, and exactly one receipt exists per registered acceptance no matter who delivered first or how many times. Removing a registered blob retires its register task per acceptance; the cause-keyed tombstone records them.

Idempotent: a delivery that finds the registration already minted returns the existing receipt's link in its outcome.

#### Arguments

**IPLD schema**

```ipldsch
type RegisterArguments struct {
  space    String # DID of the space the blob is registered to
  blob     Blob   # digest and size
  provider String # DID of the storage node holding the registered acceptance
  cause    Link   # the /blob/batch/add task
  accept   Link   # the /blob/accept task whose delivery triggered registration
}
```

<details>
<summary>Go syntax</summary>

```go
type RegisterArguments struct {
  Space    did.DID             `cborgen:"space"`
  Blob     blob.Blob           `cborgen:"blob"`
  Provider did.DID             `cborgen:"provider"`
  Cause    cid.Cid             `cborgen:"cause"`
  Accept   cid.Cid             `cborgen:"accept"`
}
```
</details>

#### Result

Successful registration returns a unit result (`{}`). The receipt's substance is its signature and issue time.

### `/blob/batch/extend`

* Issuer: Agent
* Audience: Sprue
* Subject: The space

Requests replacement candidates for blobs whose existing candidates have failed. Each entry MAY exclude providers the client has already failed against; sprue MUST NOT place an entry on an excluded provider. The result is a `BatchAddOK` — fresh candidates with fresh delegations, opening fresh quota holds and a fresh registration horizon for the re-placed blobs — and the returned delegations bind `cause` to the **original** batch-add task, so the whole batch delivers under one cause.

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

e.g.

```jsonc
// Encoded as dag-json for readability
{
  "iss": "did:key:agent",
  "aud": "did:web:up.forge.example.com",
  "sub": "did:key:space",
  "cmd": "/blob/batch/extend",
  "args": {
    "cause": { "/": "bafy...batchaddtask" },
    "blobs": [
      {
        "digest": { "/": { "bytes": "EiCcvAfD..." } },
        "exclude": ["did:key:zFailedNode"]
      }
    ]
  }
}
```

#### Result

`BatchAddOK`, as for [`/blob/batch/add`](#blobbatchadd).

### The node legs: `/blob/allocate` and `/blob/accept`, unchanged verbs

* Issuer: Agent
* Audience: Piri
* Subject: The provider

The storage-node verbs, their argument types and their results are exactly today's. What changes:

- Both are invoked by the agent under the placement delegations, with `cause` = the `/blob/batch/add` task CID (bound by policy), so every allocation and acceptance a node writes names the client invocation that motivated it.
- `/blob/accept`'s `_put` field awaits the client-minted `/http/put` task, preserving the chain shape.
- A node that serves batch placements advertises **batch support** at registration, which commits it to two behaviors. It SHOULD deliver each accept it executes to the upload service via `/ucan/conclude` (see [Registration by receipt delivery](#registration-by-receipt-delivery)), retrying until acknowledged. And it reclaims acceptances that remain unregistered past its reclaim deadline — learned from its delivery outcomes (`Surplus`, `Removed`, `Expired`) or from sustained delivery failure — where the deadline MUST be at least the placement delegation's expiry plus the protocol grace period, so reclaim can never race a chain that could still register. Beyond that floor the deadline is node policy.

## Registration by receipt delivery

Acceptance happens at the node; registration happens at sprue; and the bridge between them is receipt delivery. Delivery removes the coordination state the upload service would otherwise need — no journal, no accept fan-out, no timeout vocabulary — because the evidence sits with the parties who can act on it:

- **The node holds the receipt it minted and bears the loss if registration never happens** — an accepted blob nobody registers is bytes it stores and proves unpaid. So the node SHOULD conclude every accept it executes, retrying until acknowledged.
- **The agent holds the same receipts** and SHOULD deliver them batched — typically immediately, as its confirmation that registration is underway. Either delivery alone suffices; both together make loss require two independent failures.
- **Sprue does only local work per delivery**: verify the chain, apply the horizon, the bound and the tombstones, mint the registration, persist. Registration is idempotent, so redundant and repeated deliveries converge on one `/blob/register` receipt, and a delivery can be replayed at any time by anyone still holding the receipts.

The **registration horizon** is what keeps the three parties' clocks consistent. A chain registers only until its placement delegation's expiry plus a protocol grace period; the node's reclaim deadline starts at that same boundary. So a chain that can still register names bytes the node still holds, a reclaimed acceptance names a chain that can no longer register, and a months-old receipt delivered by anyone is refused rather than resurrected as a billed ghost. The failure this makes visible instead of silent: a delivery that misses the horizon returns `Expired`, the node reclaims, and the client re-uploads under a fresh batch — rather than sprue registering bytes no node holds.

The residual window is bounded: a blob is accepted-but-unregistered until the first delivery lands, and if every holder crashes before delivering, until one recovers or the horizon closes it out.

The client's completion does not wait for any of this. Resolvability comes from accept itself (the node publishes its location claim synchronously, as today), so the S3 200 returns on accept receipts; registration — billing, retain seeding, repair and removal routing — converges with delivery, typically within the agent's own immediate conclude. A client that wants a stronger signal than the 200 holds one artifact: the `/blob/register` receipt linked from its delivery outcome, verifiable offline and presentable to anyone.

## Failure handling and idempotency

- A failed or stalled PUT races another candidate; no sprue call. An exhausted candidate set costs one `/blob/batch/extend`.
- A refused accept is visible to the agent directly in the node's response; the blob goes back through the race or through extend.
- An `Expired` outcome means the batch's registration window lapsed undelivered; the blob re-uploads under a fresh batch.
- The agent SHOULD reconcile its index against the `/blob/register` receipt linked from its delivery outcomes: it names the registered provider, whose location claim is the durable one.
- Allocations that never receive bytes, and bytes from cancelled or losing PUTs, expire on the node's existing allocation-expiry hygiene, with no message required.
- Every capability in this RFC is idempotent: re-adding, re-delivering and re-extending converge on the same state, so the retry story for the whole protocol is "re-invoke".

## Security considerations

- **Attenuation.** The placement delegations' policy pins space, digest, size and cause; ucantone applies policies conjunctively at every hop and rejects escalation. An agent holding a batch's delegations can act only on the placements sprue decided to permit, nowhere else, and only until expiry.
- **Registration requires evidence.** No delivered artifact moves state unless the chain proves it: a node-signed accept receipt whose invocation chain terminates in sprue's own placement delegation, inside the horizon. The conclude issuer is irrelevant by design — a leaked receipt chain delivered by a stranger causes at most a correct, in-horizon registration of the space the chain names. Registration produces evidence in turn: the sprue-signed `/blob/register` receipt is the record billing and disputes start from.
- **No new agent trust.** The agent gains no authority it could not already exercise through `/blob/add`; sprue's placement discretion is unchanged, exercised at plan time instead of invocation time.
- **Surplus acceptance is bounded twice.** The registration bound caps what sprue registers per `(digest, space)`; the reclaim deadline — a commitment of every batch-advertising node — returns unregistered acceptances to reclaimable state. An agent that over-accepts gains nothing durable and costs the nodes only until reclaim.
- **Evidence strength.** The put receipt is a client assertion — its digest-derived key was always mintable by any holder of the digest — and nothing rests on it; the node-signed accept receipt is the artifact registration relies on.
- **Replay.** Delegations expire; invocations carry nonces; allocation and acceptance are keyed `(digest, space)`; delivery, registration and reclaim all converge under repeats; the horizon refuses stale chains outright, and the tombstones stop a replayed delivery from resurrecting a removed registration in either race direction.
- **Resource bounds.** A batch amplifies work: one invocation makes sprue run placement and mint delegations for M blobs, and one conclude makes it verify M chains. Sprue SHOULD enforce configurable caps on blobs per batch and receipts per conclude. Quota SHOULD be taken as a hold with a TTL equal to the delegation expiry, released on expiry and reconciled at registration; registration inside the horizon debits unconditionally, so the over-commit exposure is bounded to chains delivered in the grace period after their hold released — an outright debit at add would leak on abandoned batches, and a bare check would over-commit across parallel batches.

## Limits

The cbor-gen list cap (8,192 elements) bounds every list in these schemas. The per-candidate delegation baseline puts two or three signed delegations per candidate, a few hundred bytes each, in the batch-add response container — order of a megabyte per thousand candidates. Batches large enough to care SHOULD use the per-provider `or`-compacted delegation form. `MaxBlobSize` (256 MiB) is unchanged, enforced node-side at allocate as today.

## Compatibility

- **Storage nodes**: the verbs, argument types and results are unchanged, and the provider → sprue → agent chain validates under existing subject-plus-policy checking. Outbound conclusion has an existing precedent — piri already delivers replica-transfer receipts via `/ucan/conclude`. What is new is opt-in: a node advertises batch support at registration, taking on the delivery and reclaim behaviors, and sprue places batches only on nodes that have. Nodes that never advertise keep serving the legacy flow exactly as today and see no batch traffic, so the delivery model's guarantees never depend on a node that hasn't signed up for them.
- **`/blob/add`**: remains, unchanged, for single-blob callers and for streaming clients that cannot know digests up front. A streaming client MAY still batch per flush window — announce the blobs whose digests it has, race, accept, deliver, repeat.
- **`/ucan/conclude`**: the plural field is additive, and the `/blob/accept` conclusion handler is a new entry in the existing dispatch map; existing single-receipt clients and the legacy `/http/put` handler are untouched.
- **Receipt-chain consumers**: the persisted chain — `batch add → allocate → put → accept`, closed by a sprue-signed `register` — has today's shape in its prefix, so the replay path and the [blob-removal RFC](./2026-07-forge-blob-removal.md)'s receipt-chain routing work unchanged for delivered blobs; consumers MAY also start from the registration receipt, which links the accept task directly. Undelivered blobs differ: sprue has not seen their chains, so `/blob/abort` cannot route for them. The backstop is node-side allocation expiry and the reclaim deadline; the optional `/blob/reject` delegation gives an agent early reclaim directly against the node.
- **Read-on-write**: resolvability is preserved at its source — the node publishes its location claim synchronously inside accept, before the agent ever holds the receipt. Registration-dependent behavior converges with delivery. Index and upload registration (`/index/add`, `/upload/add`) follow the accept receipts exactly as today, and are out of scope here.

## Future work

- **Fold the node legs into the PUT.** The end state carries the write authorization in the PUT request header and returns the accept receipt, location claim and PDP promise in the response header, collapsing the per-node exchanges to zero and using `Expect: 100-continue` to recover the warm-path signal. That change touches the node and deserves its own RFC; nothing here precludes it — the delegation regime and the delivery model are the same.

## Example flow

An object of M blobs raced across N candidate nodes:

| # | Leg | What moves |
|---|-----|------------|
| 1 | agent → sprue | one `/blob/batch/add`: M digests and sizes |
| 2 | sprue → agent | `BatchAddOK`: candidates per blob, placement delegations in the container |
| 3 | agent → each node, parallel | one container batching that node's `/blob/allocate` invocations; receipts carry upload addresses (or none — warm, instant winner) |
| 4 | agent → nodes, parallel | HTTP PUTs to chosen candidates; cancel at enough; mint a put receipt per winner |
| 5 | agent → each winning node, parallel | one container batching that node's `/blob/accept` invocations; receipts carry location claims and PDP promises — **the client is done** |
| 6 | node → sprue, async | each node SHOULD `/ucan/conclude` the accepts it executed |
| 7 | agent → sprue, async | the agent SHOULD deliver its collected receipts in one batched `/ucan/conclude`; the response links a sprue-signed `/blob/register` receipt per registered blob |

A repair round inserts between legs 5 and 7: the agent calls `/blob/batch/extend` for the blobs that failed everywhere, repeats legs 3–5 against the fresh candidates, and the late receipts ride the same delivery path under the original cause.

Synchronous round trips: one to sprue per batch, two per node — against three exchanges with sprue per blob (`/blob/add`, `/ucan/conclude`, at least one poll GET) and two sprue→node invocations per blob today. For a 100-blob object on 20 nodes: at most 41 synchronous control exchanges, all parallel across nodes, plus at most 21 asynchronous receipt deliveries off the completion path — instead of ~500, serial per blob. Byte PUTs are excluded from both counts.

## Open questions

- The value of the registration grace period, and of the delegation expiry it extends (the delegation lifetime is also the upload window, so it wants to be generous — hours, not seconds).
- The mechanics of the batch-support advertisement at registration: a capability proven in the registration container, alongside the four commands required today, or a flag on the provider record.
- Whether the agent awaits its conclude response — and with it the `/blob/register` receipt — before the S3 200, or fires it after; this RFC leans to the latter since resolvability does not depend on it.
- Default and maximum sizes for the batch, conclude and candidate-set caps.
- Whether `/blob/reject` delegations ship in placement proofs by default or on request.
- Whether the per-provider `or`-compacted delegation form ships in v1 or waits for a batch size that needs it.

## Evaluation criteria

- Synchronous control exchanges for an M-blob object drop from ~5M to 1 + 2N, measured on ingot's sharded-object upload, with zero polling.
- End-to-end upload latency for a multi-blob object improves by at least the removed poll, conclusion wait and serialization overhead, measured against the current guppy/ingot path.
- No growth in stranded state: accepted-but-unregistered blobs converge to zero — registered, or reclaimed with a visible `Expired`/`Surplus`/`Removed` outcome — under fault injection: node crash before conclude, agent crash before conclude, sprue restart, `/blob/remove` racing a late delivery in both directions, and delivery after the horizon.
- Exactly one `/blob/register` receipt exists per registered acceptance under concurrent node and agent delivery.
- The legacy single-blob flow is byte-identical on the wire before and after deployment.

## References

- [Forge blob removal RFC](./2026-07-forge-blob-removal.md) — blob lifecycle, `/blob/release`, `/blob/reject`, receipt-chain routing.
- [Blob protocol specification](https://github.com/fil-forge/ucan-protocol-specs/blob/main/blob.md) — the current single-blob add protocol this RFC batches.
- [UCAN specification](https://github.com/ucan-wg/spec) — delegation, invocation, policy language.
