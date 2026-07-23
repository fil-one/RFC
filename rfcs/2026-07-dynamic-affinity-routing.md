# RFC: Dynamic affinity routing

Status: Experimental

## Authors

- [Alan Shaw](https://github.com/alanshaw)

## Motivation

[Ingot](2026-06-forge-s3-tenant-management.md#ingot---s3-api) is an S3 facade
that is normally co-located with a Forge Piri storage node. The default Sprue
routing policy selects an eligible storage provider using weighted random
selection among other routing factors. That policy can send an upload accepted
by Ingot to a _remote_ Piri, adding an avoidable data-plane hop and potentially
cross-region transfer.

Ingot needs a way to express its affinity for its local Piri, that allows for
dynamic updates if needed, and that does not impose significant API changes or
state additions to Sprue.

## Hypothesis / Goals

Ingot should be able to dynamically choose a Piri node to write to, providing
opportunity to horizontally scale, manage downtime/maintenance and load balance
writes.

Non-dynamic space-level affinity (referred herein as _static affinity_), where
ALL writes to a given space are routed to a specific storage node, imposes a
state management burden on Sprue, adds to routing complexity and necessitates an
additional UCAN command for affinity management. It will also disallow
multi-region buckets to exist in the future.

Adding an optional affinity list to `/blob/add` will allow Ingot to constrain
Sprue's storage-node selection to its co-located Piri(s) without imposing
additional state management on Sprue, all via a minimal backward compatible
protocol change.

The design has the following goals:

1. Preserve existing `/blob/add` routing behavior when no affinity is given.
2. Allow an invoker to limit a request to one or more registered Piri nodes.
3. Keep the final choice among the allowed nodes with Sprue.
4. Ensure that an affinity-constrained request is never routed to a node outside
   its affinity list.

## Design

### Terminology

**Affinity** is a set of storage-node DIDs supplied by an invoker of
`/blob/add`. It is a routing constraint, not a request that any particular
node be selected, unless the set contains just a single DID.

**Storage node** is a Piri node registered with Sprue and eligible to receive
blob allocations.

### `/blob/add` arguments

`/blob/add` gains an optional `affinity` field:

```ipldsch
type BlobAddArguments struct {
  # Existing /blob/add arguments are omitted.
  affinity optional [DID]
}
```

An invocation without `affinity` has the same routing semantics as it did
before this change.

When present, `affinity` MUST contain one or more DIDs. Every DID in the list
MUST identify a storage node registered with Sprue. An invocation with an
empty list or a DID that does not identify a registered storage node is
invalid and Sprue MUST reject it.

The list is a candidate set, not an ordered preference list. Clients MAY
include one DID to require a particular storage node or multiple DIDs to allow
Sprue to select between them.

### Routing

For an invocation with `affinity`, Sprue MUST select a storage node from the
provided list. Sprue MUST NOT route that invocation to a storage node whose DID
is not in `affinity`.

Sprue MAY use any of its normal routing considerations to choose among the
provided nodes, including availability, capacity, and weights. It MAY choose
any node in the affinity list at its discretion; the order of the list has no
meaning.

If none of the affinity nodes can serve the request, Sprue MUST fail the
invocation. It MUST NOT fall back to a storage node outside the affinity list.

An affinity list constrains only the storage node selected for the current
`/blob/add` invocation. It does not guarantee that the selected node remains
available, establish a durable placement policy, or request replication.

### Ingot use

An Ingot instance SHOULD include the DID or DIDs of its co-located Piri nodes
as `affinity` on each `/blob/add` invocation it originates. Ingot MAY omit
`affinity` when it intentionally accepts Sprue's ordinary routing policy.

This preserves Sprue as the authority that selects the node while ensuring that
an Ingot deployment does not cause a blob to be allocated to an unrelated
storage node.

## Alternatives Considered

### Static affinity routing

AKA assigning a "home" node to each space.

Sprue could store affinity as an attribute of a space as part of Ingot's space
creation procedure. This is additional state that _Sprue_ would have to store
as well as provide additional UCAN command for management (to initially set and
subsequently change). Affinity would also have to be retrieved and considered
when routing, adding an additional complexity to an otherwise relatively simple
decision.

Conversely, dynamic affinity routing may be useful for use cases outside of an
S3 facade. A client may have a trusted set of providers and wish to manually
route blobs to particular nodes. A space-level policy would make this routing
choice unavailable to them or require additional per-client state in Sprue.

An additional benefit of dynamic affinity is that Ingot can add, remove, or
replace its local Piri nodes _immediately_ by changing the affinity included
with each write. A static space-level assignment instead requires a separate
update before the change takes effect, and creates a window in which Sprue may
route to an outdated node.
