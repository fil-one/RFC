# RFC: Forge multi-blob add

Status: Experimental

## Authors

- [Hannah Howard](https://github.com/hannahhoward)

## Motivation

Uploading one blob today costs the client two POSTs to the upload service, an HTTP PUT of the bytes to the storage node, and a receipt poll, and costs the upload service two invocations against the storage node, all serial: `/blob/add`, the PUT, `/ucan/conclude` carrying a self-issued `/http/put` receipt, then `GET /receipt/{cid}` until the `/blob/accept` receipt appears — at least one more round trip per blob, and a second of added latency whenever the receipt is not ready on the first try. An object of M blobs pays that chain M times. Client-side parallelism hides some of the latency but none of the message count.

The clients that drive volume know every digest before the first network call. Ingot spools a PutObject to disk and shards it before uploading; any client that pre-shards or erasure-codes an object holds its full digest list up front. Nothing in the protocol lets them say so. And the protocol has no vocabulary for racing an upload: one node is selected per blob, and if it stalls, the whole blob waits.

This RFC proposes a batch upload protocol:

- `/blob/batch/add` — announce M blobs in one invocation. The upload service plans placement for the whole batch in one pass and returns, per blob, a set of candidate placements, each carrying the upload's tasks as pre-minted, promise-chained invocations — `/blob/allocate`, `/http/put`, `/blob/accept`, `/blob/register` — exactly as today's `/blob/add` response carries them for one blob and one node.
- The agent drives the node exchange itself: it relays each candidate's allocate invocations in one container per node, PUTs in parallel to as many candidates as it chooses, cancelling the rest once enough land, then relays the accept invocations at the winners. The accept receipts — location claim and PDP promise included — are in the agent's hands when the accept round completes. Nothing is polled and nothing is awaited from the upload service.
- Registration travels as receipt delivery over `/ucan/conclude`, which stays what it is today: an open conclusion mechanism that anyone holding a receipt may invoke. The storage node SHOULD conclude each accept it executes; the agent SHOULD deliver its collected receipts as well, batched. The upload service executes the pre-minted `/blob/register` task idempotently from whichever delivery arrives first.
- `/blob/batch/extend` — replacement candidates when a blob exhausts its set; `/blob/batch/abort` — cancel a batch's outstanding placements.

Cost for an M-blob object: one synchronous round trip to the upload service, one allocate and one accept round trip per node plus the PUTs, and asynchronous receipt deliveries that never sit on the completion path.

## Goals

- One synchronous client↔upload-service exchange per batch, independent of blob count, plus one `/blob/batch/extend` exchange per repair round.
- No polling and no conclusion wait: the agent holds every receipt it needs — accept, location claim, PDP promise — directly from the nodes, at the end of the node exchange.
- The response container carries the upload as executable tasks: each candidate placement is a complete, sprue-signed program, every argument fixed, nothing for a holder to vary. The one right outside the program, early unwind via `/blob/reject`, travels as a key-bound delegation.
- `/ucan/conclude` remains an intentionally open delivery mechanism; the upload service trusts what the delivered chain proves and nothing about who delivered it.
- Reporting duty sits with the party that bears the loss: the node concludes its own accepts, and the agent's delivery is the second, independent path.
- Every acceptance has a protocol-visible registration horizon, read off the expiry of the delivered task itself, so acceptance, registration and reclaim can never disagree about whether a blob is live.
- The upload service drops off the upload path entirely after planning. It mints the tasks without contacting a node and never fans out to nodes at all.
- Every node-side action is linked to the client's intent: the allocate's `cause` names the client's batch invocation.
- Partial success is first-class. Blobs succeed and fail independently; delivery is idempotent and repeatable, so stragglers converge under the same batch.
- The receipt chain keeps today's shape — `add → allocate → put → accept` — and gains a sprue-signed terminal link: registration itself becomes a chain-visible `/blob/register` receipt.

## Concepts

### Roles

| Name  | Description                                                                  |
| ----- | ---------------------------------------------------------------------------- |
| Agent | A client of the Forge network holding delegations from a space (e.g. Ingot). |
| Sprue | The Forge network upload service.                                            |
| Piri  | A Forge network storage node (a.k.a. storage provider).                      |
| Space | A namespace (cryptographic key pair) that owns content. A bucket is a space. |
| Ingot | An S3 facade typically co-located with a Forge Piri node.                    |

### The placement program

Today's `/blob/add` response already carries the upload as a program: the `/blob/allocate`, `/http/put` and `/blob/accept` invocations, promise-chained, each executed by a different party. This RFC keeps that contract and widens it. For each candidate placement of each blob, sprue pre-mints the full task chain:

| Task | Executor | Chaining |
| --- | --- | --- |
| `/blob/allocate` | the node (the agent relays the invocation) | `cause` = the `/blob/batch/add` task |
| `/http/put` | the client (signs the receipt with the digest-derived key sprue embeds, as today) | `destination` awaits the allocate task |
| `/blob/accept` | the node (the agent relays the invocation) | `_put` awaits the put task |
| `/blob/register` | sprue, at delivery | `accept` links the accept task |

The allocate and accept invocations are issued and signed by sprue with the provider as subject, under the provider's registration delegation — the same chain nodes validate today; sprue likewise authors the put task under the digest-derived key, and the sprue-subject register task. The agent adds no authority and can vary nothing, only execute or not execute. All four are built without nonces, so their task CIDs are deterministic and known to every party from the moment the response lands, including the register task, whose receipt the client can await from t=0. The allocate, put and accept tasks carry an explicit expiry equal to the batch's upload window (the 30-second invocation default would be useless here). The register task expires later, at the registration horizon: the window plus a delivery allowance for receipts to land.

Because allocate's subject is the provider, the chain is minted per candidate: a blob with three candidates gets three parallel programs, of which the client executes one (or more, up to the replica count). Unexecuted programs cost nothing and expire.

One artifact in the placement is not an invocation: a **`/blob/reject` delegation**, audience the agent, its policy pinning the space and the digest so the agent can unwind only its own placement's parked state — content-addressed dedup means another tenant can hold a parked allocation for the same digest, and an unpinned space field would reach it. Reject unwinds the upload rather than advancing it, and that difference is the security line: the program's tasks are idempotent and produce only what the space announced, so they are safe to execute on delivery no matter who carries them, while reject carries destructive discretion over in-flight state and is therefore key-bound to the agent. It is safe against anything accepted: reject refuses blobs the invoking space has accepted (`BlobAccepted`, per the [blob-removal RFC](./2026-07-forge-blob-removal.md)).

### Over-allocation and the race

Placement returns more candidates than a blob needs. The agent allocates at its candidates, PUTs to as many as it likes, and stops when enough land — the race is client policy, from "primary first, spare on failure" to "start all, cancel at first." A candidate whose allocate receipt carries no upload address already holds the bytes (the warm path, unchanged) and is an instant winner.

The agent then relays `/blob/accept` at winners, and MUST accept at no more than the blob's replica count of them: acceptance is the durable state, and registration will not pay for more. A retried accept that lands late anyway can still leave one extra acceptance; that converges rather than corrupts — the surplus node learns its status from its own delivery outcome and reclaims, readers who hit its stale location claim retry the registered node's claim, and the agent reconciles its index from the delivery outcomes. When relaying an accept, the agent MUST include the candidate's register invocation, put invocation and put receipt in the same container: those artifacts are what the node's own delivery needs, and the node refuses the accept without them.

Losing candidates need no message at all: their parked allocations, and any bytes from PUTs cancelled after completion, expire node-side.

### Batch lifecycle

1. **Plan** — `/blob/batch/add`. Sprue selects candidate nodes per blob over its provider directory in one pass, mints the placement programs, and returns them. Sprue contacts no node; blobs already registered in the space are returned as such, with no placement.
2. **Allocate, race, accept** — one container of allocate invocations per candidate node; PUTs in parallel; cancel at enough; one container of accept invocations per winning node. The agent ends the node exchange holding node-signed accept receipts, location claims and PDP promises for every blob that landed. This is the client's completion point.
3. **Deliver** — the node SHOULD invoke `/ucan/conclude` on sprue for each accept it executes; the agent SHOULD deliver its collected accept receipts in one batched `/ucan/conclude`. Sprue verifies each delivered chain, executes the pre-minted `/blob/register` task idempotently, and answers with per-receipt outcomes linking its receipt.
4. **Repair the tail** — a blob whose candidates all failed gets fresh ones from `/blob/batch/extend` and its receipts are delivered later under the same batch. A batch the client walks away from is cancelled with `/blob/batch/abort`.

### Transport

Batching travels as ucantone container contents: a single HTTP exchange carries a container holding any number of invocations, and receipts ride container metadata the same way. This stack has no ucanto-style effects field, and none is needed.

## Capabilities

### `/blob/batch/add`

* Issuer: Agent
* Audience: Sprue
* Subject: The space

Announces a batch of blobs for upload. Sprue verifies the caller's space authority and quota exactly as `/blob/add` does, plans placement for the whole batch in one selection pass, and returns candidate placements per blob, with the placement programs' invocations and the reject delegations carried in the response container alongside the provider registration delegations the node-facing invocations need as proofs. Candidates come only from nodes that advertised batch support at registration (see [the node legs](#the-node-legs-bloballocate-and-blobaccept-unchanged-verbs)); sprue MUST NOT hand batch placements to a node that has not. The result names one primary candidate per blob in `placements` and additional candidates in `spares`; how many spares to race and how many to hold in reserve is client policy. Digests already registered in the space are listed in `registered` and receive no placement.

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
  allocate Link   # the /blob/allocate invocation, in the response container
  put      Link   # the /http/put invocation
  accept   Link   # the /blob/accept invocation
  register Link   # the /blob/register invocation, executed by sprue at delivery
  reject   Link   # the /blob/reject delegation, for early unwind
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
  Allocate cid.Cid             `cborgen:"allocate"`
  Put      cid.Cid             `cborgen:"put"`
  Accept   cid.Cid             `cborgen:"accept"`
  Register cid.Cid             `cborgen:"register"`
  Reject   cid.Cid             `cborgen:"reject"`
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

For a delivered `/blob/accept` receipt to register a blob, the request container MUST make the placement program resolvable: the accept invocation, its node-signed receipt, the put invocation its `_put` field awaits, the `/blob/allocate` invocation that put's `destination` awaits, and the placement's `/blob/register` invocation. Deliverers SHOULD include the allocate receipt as well, so sprue persists a complete chain for receipt-chain consumers.

Sprue dispatches each delivered receipt whose `Ran` command has a registered conclusion handler; other artifacts in the container are context. The new `/blob/accept` conclusion handler, per delivery:

1. verifies the accept receipt is signed by the provider DID that is the accept invocation's subject, with `Ran` matching and a success result — else `InvalidReceipt`;
2. verifies the accept and register invocations are **issued and signed by sprue itself**, and that the register invocation's `accept` field names the delivered accept task. Sprue verifying its own signatures is the placement check, and it requires no stored state — the register task exists only for placements sprue minted, which also keeps legacy sprue-signed accepts out of this path — failures return `UnknownPlacement`;
3. enforces the registration horizon: a register task that expired more than the protocol grace period ago MUST be refused with an `Expired` outcome. Within the horizon, freshness is not otherwise rechecked — the node-signed receipt attests that the accept validated, expiry included, at execution time;
4. enforces the registration bound: at most the replica count the register invocation names is registered per `(digest, space)`. The earliest delivered acceptances win, up to the bound; further acceptances of the same blob at other providers return a `Surplus` outcome and are not registered;
5. checks the tombstones (below): a chain matching one returns a `Removed` outcome instead of registering;
6. registers the blob by executing the register task — issuing its receipt — debits the space's quota, and persists the delivered chain and the registration alongside it.

Registration and removal MUST serialize per `(digest, space)`, and two tombstones make their races come out right in both directions. Removing a **registered** blob retires its `/blob/register` tasks and tombstones its `(digest, space, cause)`: a redelivered chain for that cause returns `Removed`, while a fresh batch's chain — new cause — registers. Removing a digest with **no registration yet** writes a pending tombstone stamped with the removal time, held for the length of the registration horizon: it blocks chains whose placement was minted before the stamp (the batch the client walked away from) and passes chains minted after it (a re-upload). `/blob/batch/abort` writes the same pending tombstones for a whole batch at once. Both checks compare the delivered artifacts — the register task's `cause` and issuance time — against the tombstone record alone, so sprue keeps no journal of in-flight uploads.

Delivery is idempotent and unordered: the register task's CID is fixed at minting, so the node's conclusion and the agent's conclusion of the same accept converge on the identical task and one receipt, and a repeat delivery returns the same registration link. A delivery failing any check lands in the outcome list with a named reason and blocks nothing else, and a `Surplus` outcome still carries the blob's registration links, so the deliverer learns the registered providers directly. The legacy `/http/put` conclusion handler is unchanged and continues to serve the single-blob flow.

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
// Encoded as dag-json for readability; the referenced accept receipts and
// their placement-program invocations travel in the same container
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

The terminal task of the placement program, and a virtual task in the tradition of `/http/put`: no node serves it. Sprue mints it at `/blob/batch/add` alongside the other three tasks and executes it — issues its receipt — inside the `/blob/accept` conclusion handler when a delivery verifies. The receipt is the canonical, sprue-signed record that a blob is registered — the artifact billing, index reconciliation and disputes start from — and the registration time is the receipt's issue time. Sprue sets `replicas` from the space's placement policy at planning time; extend-minted register tasks carry the original batch's bound. Because the task CID is fixed at minting, the client can await this receipt from the moment the batch-add response lands.

Exactly one receipt exists per registered acceptance, no matter who delivered first or how many times. Removing a registered blob retires its register tasks; the cause-keyed tombstone records them.

Idempotent: a delivery that finds the registration already executed returns the existing receipt's link in its outcome.

#### Arguments

**IPLD schema**

```ipldsch
type RegisterArguments struct {
  space    String # DID of the space the blob is registered to
  blob     Blob   # digest and size
  provider String # DID of the storage node holding the registered acceptance
  cause    Link   # the /blob/batch/add task
  accept   Link   # the /blob/accept task whose delivery triggers registration
  replicas Int    # registration bound for this blob, per (digest, space)
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
  Replicas uint64              `cborgen:"replicas"`
}
```
</details>

#### Result

Successful registration returns a unit result (`{}`). The receipt's substance is its signature and issue time.

### `/blob/batch/extend`

* Issuer: Agent
* Audience: Sprue
* Subject: The space

Requests replacement candidates for blobs whose existing candidates have failed. Each entry MAY exclude providers the client has already failed against; sprue MUST NOT place an entry on an excluded provider. The result is a `BatchAddOK`: fresh placement programs with fresh expiries and a fresh registration horizon for the re-placed blobs, whose quota holds replace the superseded candidates' outstanding ones, and whose tasks bind `cause` to the **original** batch-add task, so the whole batch delivers under one cause. Extend MUST NOT issue a placement for a digest carrying a live cause-keyed tombstone under this cause, and extending an aborted batch MUST fail with the named error `BatchAborted`.

Idempotent: extending a digest that has since registered simply returns no placement for it, listed in `registered`.

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

### `/blob/batch/abort`

* Issuer: Agent
* Audience: Sprue
* Subject: The space

Cancels a batch's outstanding placements: the client renounces every placement of the batch that has not registered. Sprue releases the batch's remaining quota holds and writes the pending tombstones immediately, so chains from the abandoned placements return `Removed` instead of registering — the abort is, in effect, revocation of the outstanding placement programs. Blobs already registered are untouched and listed in the result; they exit via `/blob/remove`.

Abort is the upload-service half of cancellation. For node-side cleanup the client invokes `/blob/reject` under the placements' reject delegations, one container per node it touched, and SHOULD abort first so the tombstones are in place before any in-flight delivery races the cleanup. Acceptances that already executed cannot be rejected (`BlobAccepted`); they resolve through the delivery loop — the node concludes, receives `Removed`, and reclaims.

Abort terminates the cause: a later `/blob/batch/extend` under it fails with `BatchAborted`, so a cancelled batch cannot mint post-tombstone placements that slip past its own cancellation.

Idempotent: aborting an unknown, already-aborted, or fully-registered batch MUST succeed.

#### Arguments

**IPLD schema**

```ipldsch
type BatchAbortArguments struct {
  cause Link # the /blob/batch/add task being cancelled
}
```

<details>
<summary>Go syntax</summary>

```go
type BatchAbortArguments struct {
  Cause cid.Cid `cborgen:"cause"`
}
```
</details>

#### Result

**IPLD schema**

```ipldsch
type BatchAbortOK struct {
  cancelled  [Bytes] # digests whose placements are tombstoned
  registered [Bytes] # digests already registered; use /blob/remove
}
```

<details>
<summary>Go syntax</summary>

```go
type BatchAbortOK struct {
  Cancelled  []multihash.Multihash `cborgen:"cancelled"`
  Registered []multihash.Multihash `cborgen:"registered"`
}
```
</details>

### The node legs: `/blob/allocate` and `/blob/accept`, unchanged verbs

* Issuer: Sprue
* Audience: Piri
* Subject: The provider

The storage-node verbs, their argument types, their results, and their issuer are exactly today's — the invocations are sprue-signed under the provider's registration delegation, and only the courier changes: the agent relays them, with the proofs, in its own containers. `cause` on allocate names the `/blob/batch/add` task, so every allocation and acceptance a node writes names the client invocation that motivated it, and `_put` on accept awaits the placement's `/http/put` task, preserving the chain shape.

A node that serves batch placements advertises **batch support** at registration, which commits it to two behaviors:

- It MUST refuse to execute an accept whose container lacks the placement's register invocation, put invocation and put receipt — without them its acceptance could never deliver a registrable chain — and SHOULD deliver each accept it executes to the upload service via `/ucan/conclude` (see [Registration by receipt delivery](#registration-by-receipt-delivery)), retrying until acknowledged. It SHOULD defer enqueueing the acceptance into PDP aggregation until a registration signal — its delivery outcome, or the register receipt — so a forced or surplus acceptance costs storage until reclaim rather than on-chain proof obligations.
- It reclaims acceptances that remain unregistered past its reclaim deadline — learned from its delivery outcomes (`Surplus`, `Removed`, `Expired`, `UnknownPlacement`) or from sustained delivery failure — where the deadline MUST be at least the register task's expiry plus the protocol grace period, so reclaim can never race a chain that could still register. Beyond that floor the deadline is node policy. Reclaim has the semantics of a self-invoked `/blob/release` under the [blob-removal RFC](./2026-07-forge-blob-removal.md): it drops that space's acceptance and location claim, and physical deletion remains gated on zero claims across spaces and on-chain proof retirement. This is a deliberate amendment to that RFC's lifecycle, which otherwise lets an acceptance exit only via `/blob/remove`; the exit exists only for acceptances that never registered.

## Registration by receipt delivery

Acceptance happens at the node; registration happens at sprue; and the bridge between them is receipt delivery. Delivery removes the coordination state the upload service would otherwise need — no journal, no accept fan-out, no timeout vocabulary — because the evidence sits with the parties who can act on it:

- **The node holds the receipt it minted and bears the loss if registration never happens** — an accepted blob nobody registers is bytes it stores and proves unpaid. So the node SHOULD conclude every accept it executes, retrying until acknowledged.
- **The agent holds the same receipts** and SHOULD deliver them batched — typically immediately, as its confirmation that registration is underway. Either delivery alone suffices; both together make loss require two independent failures.
- **Sprue does only local work per delivery**: verify its own signatures on the program, apply the horizon, the bound and the tombstones, execute the register task, persist. Registration is idempotent, so redundant and repeated deliveries converge on one `/blob/register` receipt, and a delivery can be replayed at any time by anyone still holding the receipts.

The **registration horizon** is what keeps the three parties' clocks consistent, and it is read off the delivered artifact itself: a chain registers only until its register task's expiry plus a protocol grace period, and the node's reclaim deadline starts at that same boundary. So a chain that can still register names bytes the node still holds, a reclaimed acceptance names a chain that can no longer register, and a months-old receipt delivered by anyone is refused rather than resurrected as a billed ghost. A delivery that misses the horizon returns `Expired`, the node reclaims, and the client re-uploads under a fresh batch; sprue never registers bytes no node holds.

The residual window is bounded: a blob is accepted-but-unregistered until the first delivery lands, and if every holder crashes before delivering, until one recovers or the horizon closes it out.

The client's completion does not wait for any of this. Resolvability comes from accept itself (the node publishes its location claim synchronously, as today), so the S3 200 returns on accept receipts; registration — billing, retain seeding, repair and removal routing — converges with delivery, typically within the agent's own immediate conclude. A client that wants a stronger signal than the 200 holds one artifact: the `/blob/register` receipt, addressable by task CID from the batch-add response, verifiable offline and presentable to anyone.

## Failure handling and idempotency

- A failed or stalled PUT races another candidate; no sprue call. An exhausted candidate set costs one `/blob/batch/extend`.
- A refused accept is visible to the agent directly in the node's response; the blob goes back through the race or through extend.
- An `Expired` outcome means the batch's registration horizon lapsed undelivered; the blob re-uploads under a fresh batch.
- The agent SHOULD reconcile its index against the `/blob/register` receipts linked from its delivery outcomes: they name the registered providers, whose location claims are the durable ones.
- Allocations that never receive bytes, and bytes from cancelled or losing PUTs, expire on the node's existing allocation-expiry hygiene, with no message required. An abandoned batch is cancelled sooner with `/blob/batch/abort` plus reject.
- Every capability in this RFC is idempotent: re-adding, re-delivering, re-extending and re-aborting converge on the same state, so the retry story for the whole protocol is "re-invoke".

## Example flow

An object of M blobs raced across N candidate nodes:

| # | Leg | What moves |
|---|-----|------------|
| 1 | agent → sprue | one `/blob/batch/add`: M digests and sizes |
| 2 | sprue → agent | `BatchAddOK`: candidate placement programs in the container |
| 3 | agent → each node, parallel | one container relaying that node's `/blob/allocate` invocations; receipts carry upload addresses (or none — warm, instant winner) |
| 4 | agent → nodes, parallel | HTTP PUTs to chosen candidates; cancel at enough; sign a put receipt per winner |
| 5 | agent → each winning node, parallel | one container relaying that node's `/blob/accept` invocations (register and put invocations and put receipts included); receipts carry location claims and PDP promises — **the client is done** |
| 6 | node → sprue, async | each node SHOULD `/ucan/conclude` the accepts it executed |
| 7 | agent → sprue, async | the agent SHOULD deliver its collected receipts in one batched `/ucan/conclude`; the response links a sprue-signed `/blob/register` receipt per registered acceptance |

A repair round inserts between legs 5 and 7: the agent calls `/blob/batch/extend` for the blobs that failed everywhere, repeats legs 3–5 against the fresh candidates, and the late receipts ride the same delivery path under the original cause. A batch the client abandons is cancelled with `/blob/batch/abort` followed by `/blob/reject` at the touched nodes.

Synchronous round trips: one to sprue per batch, two per node — against three exchanges with sprue per blob (`/blob/add`, `/ucan/conclude`, at least one poll GET) and two sprue→node invocations per blob today. For a 100-blob object on 20 nodes: at most 41 synchronous control exchanges, all parallel across nodes, plus at most 21 asynchronous receipt deliveries off the completion path — instead of ~500, serial per blob. Byte PUTs are excluded from both counts.

## Security considerations

- **The program is fixed.** Every node-facing artifact is a complete sprue-signed invocation: all arguments set, nothing for the holder to vary. The agent's entire latitude is which pre-authorized programs to execute.
- **Bearer execution is bounded by the announcement.** The invocations execute on delivery, so a holder of a leaked response container can advance the program — at worst force-completing an announced upload of content it possesses, billing the space for blobs and sizes the space itself named, inside the expiry window. It cannot divert the program to other digests, sizes, spaces, or non-candidate nodes; within a blob's candidate set a faster delivery can pick a different winner than the agent chose, which converges through `Surplus` and reclaim like any late acceptance. `/blob/batch/abort`'s tombstones close even the forced-completion sliver for a batch the client cancels. The one destructive right, reject, is key-bound to the agent precisely because a bearer form of it would let an interceptor destroy in-flight uploads.
- **Registration requires evidence.** No delivered artifact moves state unless the chain proves it: a node-signed accept receipt for an accept invocation sprue itself signed, named by a register task sprue itself signed, inside the horizon. The conclude issuer is irrelevant by design. Registration produces evidence in turn: the sprue-signed `/blob/register` receipt is the record billing and disputes start from.
- **Surplus acceptance is bounded twice.** The registration bound caps what sprue registers per `(digest, space)`; the reclaim deadline — a commitment of every batch-advertising node — returns unregistered acceptances to reclaimable state. An agent that over-accepts gains nothing durable and costs the nodes storage until reclaim — and, on nodes that defer PDP enqueue to the registration signal, no proof obligations.
- **Evidence strength.** The put receipt is a client assertion — its digest-derived key was always mintable by any holder of the digest — and nothing rests on it; the node-signed accept receipt is the artifact registration relies on.
- **Replay.** Invocation task CIDs are deterministic and their effects idempotent, keyed `(digest, space)`; delivery, registration and reclaim all converge under repeats; the horizon refuses stale chains outright, and the tombstones stop a replayed delivery from resurrecting a removed registration in either race direction.
- **Resource bounds.** A batch amplifies work: one invocation makes sprue run placement and mint programs for M blobs, and one conclude makes it verify M chains. Sprue SHOULD enforce configurable caps on blobs per batch and receipts per conclude. Quota SHOULD be taken as a hold with a TTL equal to the registration horizon, released on expiry or abort and reconciled at registration; registration inside the horizon debits unconditionally, so the over-commit exposure is bounded to chains delivered in the grace period after their hold released — an outright debit at add would leak on abandoned batches, and a bare check would over-commit across parallel batches.

## Limits

The cbor-gen list cap (8,192 elements) bounds every list in these schemas. A placement is five signed artifacts — four invocations and a delegation — a few hundred bytes each, so the batch-add response runs to a couple of megabytes per thousand candidates; batch and candidate caps size accordingly. `MaxBlobSize` (256 MiB) is unchanged, enforced node-side at allocate as today.

## Compatibility

- **Storage nodes**: the verbs, argument types, results and issuer are unchanged — nodes validate today's exact chain; only the courier differs. Outbound conclusion has an existing precedent: piri already delivers replica-transfer receipts via `/ucan/conclude`. What is new is opt-in: a node advertises batch support at registration, taking on the delivery and reclaim behaviors, and sprue places batches only on nodes that have. Nodes that never advertise keep serving the legacy flow exactly as today and see no batch traffic, so the delivery model's guarantees never depend on a node that hasn't signed up for them.
- **`/blob/add` and the [blob protocol spec](https://github.com/fil-forge/ucan-protocol-specs/blob/main/blob.md)**: the single-blob flow remains, unchanged, for callers that cannot know digests up front, and the spec's response-container contract generalizes rather than changes — the container still carries the upload's promise-chained task invocations; there are simply more of them, plus the explicit terminal register task. A streaming client MAY batch per flush window — announce the blobs whose digests it has, race, accept, deliver, repeat.
- **`/ucan/conclude`**: the plural field is additive, and the `/blob/accept` conclusion handler is a new entry in the existing dispatch map; existing single-receipt clients and the legacy `/http/put` handler are untouched.
- **The [blob-removal RFC](./2026-07-forge-blob-removal.md)** is amended in three stated places. Its receipt-chain routing walk (the registration's cause, then the add receipt's `site`, then the accept task) does not exist for batch registrations — `BatchAddOK` has no `site` — so for batch-registered blobs routing MUST start from the `/blob/register` receipts, which name the providers and link the accept tasks directly, and `/blob/remove` fans `/blob/release` to every registered acceptance's provider. Its remove handler gains duties: the pending tombstone on removal of an unregistered digest, and per-`(digest, space)` serialization with registration. And its lifecycle gains the reclaim exit for never-registered acceptances, as specified in [the node legs](#the-node-legs-bloballocate-and-blobaccept-unchanged-verbs). `/blob/abort` cannot route for batch blobs (sprue never learns which candidates the client executed until delivery); `/blob/batch/abort` plus the reject delegations replace it at batch granularity, with node-side expiry as the passive backstop.
- **Read-on-write**: resolvability is preserved at its source — the node publishes its location claim synchronously inside accept, before the agent ever holds the receipt. Registration-dependent behavior converges with delivery. Index and upload registration (`/index/add`, `/upload/add`) follow the accept receipts exactly as today, and are out of scope here.

## Alternatives considered

### Policy-bound delegations instead of pre-minted invocations

The batch-add response could carry authority instead of tasks: sprue re-delegates `/blob/allocate` and `/blob/accept` to the agent — subject the provider, policy pinning space, digest, size and cause — and the agent authors the node-facing invocations itself. The two shapes are close: the artifact count per candidate is similar, the delivery, horizon, tombstone and advertisement machinery is identical, and nodes validate a provider-rooted chain either way.

What the delegation shape buys: the response container is key-bound, so an intercepted copy is inert without the agent's key, where pre-minted invocations execute on delivery (bounded to force-completing the announced upload, as the security section states); the agent's own signature appears on the node-facing chain, rooting every node action in the customer's intent chain rather than in sprue's authorship; and very large batches can compact to one delegation pair per provider with an `or` policy over the digest set, which invocations cannot imitate.

What it costs, and why this RFC chose invocations: a policy pins only the fields it names, so policy completeness is a standing obligation — today's argument schemas already contain an unpinned field (`_put`), and every future argument added to a node verb silently becomes agent latitude until someone pins it — where an invocation is complete automatically; conclusion verification becomes a proof-chain walk with a policy-binding checklist instead of a self-signature check; and the response-container contract diverges from the deployed protocol's instead of generalizing it. A deployment that wants the agent-signed intent chain — a fleet whose constraint is that the validated chain be the customer's chain — takes this alternative wholesale; nothing else in the RFC changes.

## Future work

- **Fold the node legs into the PUT.** The end state carries the placement program in the PUT request header and returns the accept receipt, location claim and PDP promise in the response header, collapsing the per-node exchanges to zero and using `Expect: 100-continue` to recover the warm-path signal. That change touches the node and deserves its own RFC; nothing here precludes it — the program and the delivery model are the same.

## Open questions

- The values of the upload window (the upload tasks' expiry), the delivery allowance that sets the registration horizon (the register task's expiry), and the grace period beyond it; the window wants to be generous, on the order of hours.
- The mechanics of the batch-support advertisement at registration: a capability proven in the registration container, alongside the four commands required today, or a flag on the provider record.
- Whether the agent awaits its conclude response — and with it the `/blob/register` receipt — before the S3 200, or fires it after; this RFC leans to the latter since resolvability does not depend on it.
- Default and maximum sizes for the batch, conclude and candidate-set caps.

## Evaluation criteria

- Synchronous control exchanges for an M-blob object drop from ~5M to 1 + 2N, measured on ingot's sharded-object upload, with zero polling.
- End-to-end upload latency for a multi-blob object improves by at least the removed poll, conclusion wait and serialization overhead, measured against the current guppy/ingot path.
- No growth in stranded state: accepted-but-unregistered blobs converge to zero — registered, or reclaimed with a visible `Expired`/`Surplus`/`Removed` outcome — under fault injection: node crash before conclude, agent crash before conclude, sprue restart, `/blob/remove` racing a late delivery in both directions, abort racing an in-flight delivery, and delivery after the horizon.
- Exactly one `/blob/register` receipt exists per registered acceptance under concurrent node and agent delivery.
- The legacy single-blob flow is byte-identical on the wire before and after deployment.

## References

- [Forge blob removal RFC](./2026-07-forge-blob-removal.md) — blob lifecycle, `/blob/release`, `/blob/reject`, receipt-chain routing.
- [Blob protocol specification](https://github.com/fil-forge/ucan-protocol-specs/blob/main/blob.md) — the current single-blob add protocol this RFC batches.
- [UCAN specification](https://github.com/ucan-wg/spec) — delegation, invocation, policy language.
