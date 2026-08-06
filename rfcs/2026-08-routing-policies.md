# RFC: Routing policies

Status: Proposed

## Authors

- [Alan Shaw](https://github.com/alanshaw)

## Motivation

[Dynamic routing candidates](2026-07-dynamic-routing-candidates.md) allows an
invoker of `/blob/add` to constrain Sprue's storage-node selection per
invocation. The constraint travels with each write, so it binds only the
invocations that carry it: any agent authorized to write to a space that omits
`candidates` — a misconfigured Ingot, a direct client, a future integration —
falls back to Sprue's default weighted random selection, and the space's data
lands on an arbitrary storage node.

The [deployment proposal](2026-05-filone-forge-deployment-proposal.md#the-upload-path)
calls for a stronger property: every space has a home provider, and
allocations for that space go to the home provider's Piri nodes — regardless
of which agent invoked the write or from where. This keeps the data plane
in-region and future-proofs the case where a customer runs their own S3
gateway on-prem and invokes `/blob/add` directly: the customer's gateway need
not know (or be trusted to supply) storage node DIDs.

That property requires the constraint to be an attribute of the space,
enforced by Sprue for every write. Storing a candidate list per space would be
repetitive — every space in a region carries the same list, and a change to
the region's nodes becomes an update to every one of its spaces. This RFC
instead makes the candidate list a shared, first-class entity — a _routing
policy_ — that spaces reference by DID.

## Hypothesis / Goals

Routing policies give Sprue-enforced placement for all writes to a space,
while keeping the state Sprue stores minimal: one candidate list per policy
and one DID per space. Changing a region's storage nodes is a single policy
update that takes immediate effect for every space referencing it.

The design has the following goals:

1. Preserve existing routing behavior for spaces that reference no policy.
2. Allow all allocations for a space to be constrained to one or more
   registered Piri nodes, without cooperation from writers.
3. Share one candidate list across many spaces, so node changes are a single
   update.
4. Keep the final choice among the allowed nodes with Sprue.

## Design

### Terminology

**Routing policy** is an entity identified by a DID that owns a set of
storage-node DIDs — its _candidates_. It is a routing constraint for the
spaces that reference it, not a request that any particular node be selected,
unless the set contains just a single DID.

**Policy reference** is the association from a space to a routing policy,
stored by Sprue as an attribute of the space.

**Storage node** is a Piri node registered with Sprue and eligible to receive
blob allocations.

### Policy identity

A routing policy MUST be a cryptographic key pair identified by a [`did:key`]
URI — the same pattern as a [space]. Authority over the policy is rooted in
its key: after creation, the policy key SHOULD issue a non-expiring delegation
of `/` to the principal that will manage it, after which the private key MAY
be discarded — mirroring the
[bucket key pattern](2026-06-forge-s3-tenant-management.md#bucket-creation).

A policy requires no registration step: it exists in Sprue once its first
[`/routing/set`](#routingset) invocation succeeds.

### `/routing/set`

An authorized agent MAY invoke the `/routing/set` capability on a **routing
policy** subject to replace the policy's candidate set.

```ipldsch
type RoutingSetArguments struct {
  candidates [DID]
}
```

`candidates` MUST contain one or more DIDs. Every DID in the list MUST
identify a storage node registered with Sprue. An invocation with an empty
list or a DID that does not identify a registered storage node is invalid and
Sprue MUST reject it.

The list is a candidate set, not an ordered preference list.

```jsonc
{
  "iss": "did:web:hilt.example.com",
  "aud": "did:web:sprue.example.com",
  "sub": "did:key:zPolicy",
  "cmd": "/routing/set",
  "args": {
    "candidates": ["did:key:zPiri1", "did:key:zPiri2"]
  },
  "prf": [{ "/": "bafy..dlgPolicy" }]
  // ...
}
```

The success value is an empty object.

A change to a policy's candidates applies to invocations routed after the
change takes effect. In-flight writes MAY be routed using the previous set.

### `/routing/use`

An authorized agent MAY invoke the `/routing/use` capability on a **space**
subject to set or clear the space's policy reference.

```ipldsch
type RoutingUseArguments struct {
  policy optional DID
}
```

When present, `policy` MUST identify a routing policy known to Sprue — one
with a stored candidate set. An invocation referencing an unknown policy is
invalid and Sprue MUST reject it. When absent, the space's policy reference is
cleared and the space returns to Sprue's default routing.

The invocation MUST fail if the subject space is not provisioned with a
provider.

```jsonc
{
  "iss": "did:plc:tenant",
  "aud": "did:web:sprue.example.com",
  "sub": "did:key:zBucket",
  "cmd": "/routing/use",
  "args": {
    "policy": "did:key:zPolicy"
  },
  "prf": [{ "/": "bafy..dlgBucketTenant" }]
  // ...
}
```

The success value is an empty object.

### `/routing/get`

An authorized agent MAY invoke the `/routing/get` capability to inspect stored
routing state. It takes no arguments. On a **routing policy** subject the
success value is the policy's candidate set:

```ipldsch
type RoutingGetPolicyOK struct {
  candidates [DID]
}
```

On a **space** subject the success value is the space's policy reference,
where `policy` is absent when the space references no policy:

```ipldsch
type RoutingGetSpaceOK struct {
  policy optional DID
}
```

### Routing

For a `/blob/add` invocation on a space with a policy reference, Sprue MUST
resolve the referenced policy and select a storage node from its candidates.
Sprue MUST NOT route the invocation to a storage node outside the candidate
set, and MUST fail the invocation rather than fall back if no candidate can
serve it _(error name `CandidateUnavailable`)_.

Sprue MAY use any of its normal routing considerations to choose among the
candidates, including availability, capacity, and weights. The order of the
list has no meaning.

A `/blob/add` invocation on a space with no policy reference has the same
routing semantics as it did before this change.

#### Replication

The policy constrains replica placement as well: every replica node selected
for the space under the [replication protocol] MUST be a candidate of the
referenced policy. Since a replica node is never the node that received the
initial allocation, replication at level _n_ requires the policy to contain at
least _n + 1_ candidates. Policy managers MUST size candidate sets
accordingly.

### S3 deployment use

[Hilt](2026-06-forge-s3-tenant-management.md) creates one routing policy per
region at setup and holds the policy's root delegation. It is configured with
the storage-node DIDs serving each region and maintains each policy's
candidates with `/routing/set`.

On bucket creation, after provisioning the bucket (space) with Sprue, Hilt
invokes `/routing/use` — signed with the tenant key, whose authority flows
from the bucket's root delegation — pointing the bucket at its region's
policy.

A provider adding, removing, or replacing a Piri node is then a single
`/routing/set` on the region's policy: no space is touched, and every bucket
in the region follows immediately. Ingot requires no knowledge of storage-node
DIDs at all.

Policy management authority MAY in future be delegated to the regional
provider itself — the policy root delegation makes this a delegation, not a
protocol change — allowing providers to rotate their own nodes without
involving the network operator.

## Trade-offs

This design deliberately accepts costs identified in
[dynamic routing candidates](2026-07-dynamic-routing-candidates.md#static-affinity-routing):

- Sprue stores state — a candidate set per policy and a reference per space —
  and MUST resolve the reference on every routing decision for a constrained
  space.
- Changes require a `/routing/set` or `/routing/use` invocation before taking
  effect, creating a window in which Sprue may route using outdated state.
- A space is pinned to its policy's candidates; a future multi-region bucket
  would need a policy spanning regions, weakening the locality property the
  policy was created for.

It buys server-side enforcement — no writer can, deliberately or by
misconfiguration, place a space's data outside its home provider — and the
policy indirection reduces the state and update costs to one list per region.

## Alternatives Considered

### Dynamic per-invocation candidates

Specified in [dynamic routing candidates](2026-07-dynamic-routing-candidates.md),
to which this RFC is a mutually exclusive alternative. It requires no Sprue
state and changes take effect immediately, but the constraint binds only
invocations that carry it: enforcement depends on every writer supplying the
correct candidates on every write.

### Per-space candidate list

Store the candidate set directly as an attribute of each space (e.g. via a
`/routing/set` on the space subject). Rejected: every space in a region
duplicates the same list, and a change to the region's nodes becomes an update
to every one of its spaces — with a long window in which spaces route to
outdated nodes.

### Extending `/provider/add`

Provisioning is the moment a space is bound to a provider, so an optional
candidate list there would ride an existing flow. Rejected: it overloads
provisioning with policy management, requires awkward update semantics
(distinguishing an absent field from an empty list on re-invocation).

[space]: https://github.com/fil-forge/ucan-protocol-specs/blob/main/blob.md#space
[replication protocol]: https://github.com/fil-forge/ucan-protocol-specs/blob/main/replication.md
[`did:key`]: https://w3c-ccg.github.io/did-key-spec/
