# RFC: Adopt bounded POSIX access for Fil One

**Status:** Proposal
**Author:** [James Kurz](https://github.com/jameskurz-filecoin)
**Date:** 2026-08-18

## TL;DR

I propose that Fil One stage a deliberately small filesystem-access product:

- Linux amd64 and arm64;
- direct `filfs` mounts over an existing Fil One bucket;
- static Kubernetes CSI volumes over an existing bucket; and
- the documented `core` operation set: read, list, sequential create, and
  delete when explicitly enabled.

This is an access method over the existing S3 product, not a new storage,
tenant, provisioning, entitlement, usage, or billing system. Fil One and the
Service Orchestrator retain product authority; Forge and Ingot retain
enforcement; Clockwork can map a commercial line to a versioned Fil One
capability without becoming a mount runtime dependency.

Merging this RFC would approve a Forge staging integration and the ownership
boundary below. It would not approve production publication, dynamic CSI,
desktop clients, or broader POSIX claims.

## Why

Some useful storage workloads still arrive through a filesystem interface:

- backup and restore tools that can target a mounted directory but not an S3
  endpoint directly;
- Linux data-processing and media workflows built around ordinary file paths;
  and
- Kubernetes workloads that need a static volume backed by an existing bucket.

S3 remains the native interface and the honest semantic model. The mount gives
these workloads a migration and interoperability path without pretending that
object storage has acquired atomic rename, random in-place write, append,
persistent POSIX metadata, or distributed file locking.

The first staging workload should reproduce the long-running, multipart,
Proxmox-like backup failure in FIL-281 and complete a full restore and integrity
check. That is more useful than a synthetic throughput headline: it tests the
failure mode that motivated this work.

## Proposal

### Direct mount

`filfs` receives a region, existing bucket, embedded region-profile ID, and an
owner-only AWS credential file containing an existing bucket-scoped SigV4 key.
It validates the exact endpoint origin, region, operation set, and limits from
the release-bound profile, then performs an authenticated S3 readiness probe
before reporting the mount ready.

Production artifacts accept only profiles embedded into the signed release.
An operator-supplied profile is a development facility behind an explicit
unsafe flag. Bucket-prefix isolation is unsupported until FIL-917; the first
release fails closed instead of claiming it.

### Static CSI

A static PersistentVolume names the same existing bucket, region, endpoint, and
profile. Kubernetes supplies a namespaced `nodePublishSecretRef`; the node
plugin materializes an owner-only per-volume credential file and removes it on
unpublish.

The release deployment has no dynamic provisioner, no controller-side Secret
creation, and no cluster-wide Secret or ServiceAccount permissions. Controller
create, delete, snapshot, and expansion operations are unimplemented and no
controller capability is advertised.

### Semantics

The supported `core` operations are read, list, sequential create, and delete
when explicitly allowed. Empty directories use marker objects. Unsupported
rename, append, random write, persistent metadata, locks, and deeper POSIX
operations return an unsupported-operation error.

Multipart completion is the durability boundary. Ambiguous completion is an
error, never guessed success. The `core` runtime has no replay journal, and its
status, diagnostics, recovery, and fsck surfaces say so. Object Lock refusals,
expired or revoked keys, tenant disablement, and write-locks remain
authoritative at the S3 gateway.

## Ownership boundary

| Area | Authority | Posixmount responsibility |
| --- | --- | --- |
| User identity | Auth0/BFF | None |
| Product organization, entitlement, region, resource lifecycle, and raw usage | Fil One and the Service Orchestrator | Consume already-resolved mount inputs |
| Tenant and access-key enforcement | Forge/Hilt | Observe the resulting S3 response |
| Object data plane | Ingot S3 gateway | Issue the bounded SigV4 `core` operations |
| Commercial configuration and workflow | Clockwork after the applicable domain cutover; the existing Fil One path before it | No runtime dependency or commercial model |
| Stripe and billing effects | The single writer named by the applicable account/domain cutover | No billing calls; client OTLP is operational telemetry only |

The mount never receives the partner-scoped Service Orchestrator Management API
credential and never calls Clockwork. It has no account, order, SKU, price,
subscription, tenant, or billing state.

If Clockwork is adopted, an accepted commercial line maps to a versioned Fil
One filesystem-access capability. Clockwork sends the existing signed,
idempotent commercial command; Fil One validates it and owns the resulting
entitlement and resource state. Suspension or offboarding reaches a running
mount through existing product enforcement such as write-lock, disablement, or
key revocation—not through a second POSIX control plane.

## Candidate and evidence

The reviewed upstream base is
[`f708ebd5`](https://github.com/fil-one/posixmount/tree/f708ebd547c90af4fcebcffb434bad34e5cbe38d).
The remediation candidate is a four-PR review stack:

1. [Static CSI without credential authority](https://github.com/fil-one/posixmount/pull/18)
2. [`filfs` signed profile and credential binding](https://github.com/fil-one/posixmount/pull/19)
3. [Fail-closed evidence-v2 and release gates](https://github.com/fil-one/posixmount/pull/20)
4. [Fil One, Forge, and Clockwork boundary](https://github.com/fil-one/posixmount/pull/21)

The exact top revision is
[`cf6253f1`](https://github.com/fil-one/posixmount/tree/cf6253f16c1ab5d332801b0368833eaf3c0f78d3).
Portable formatting, static analysis, Rust and Go tests, dependency audits,
contract validation, release-feature inspection, secret scanning, privacy
checks, and adversarial harnesses pass for this source content. The repository
records zero release evidence and zero certifications, so no region or artifact
is currently described as qualified. No region is certified.

GitHub-hosted jobs currently terminate before their first step because of the
organization billing/spend gate. On native Linux arm64, the optimized `filfs`
build, 76 mount-library tests, four credential/security tests, Go vet,
staticcheck, Go tests, and the exact-revision OCI build pass. Linux amd64
`filfs` and CSI binaries cross-build successfully but were not executed
locally. Helm and Kustomize output passes strict schema validation. Real FUSE,
Kubernetes, Forge, signing, install/upgrade, and soak evidence remains required
before the candidate is treated as staging-ready.

## Staging and rollback

Deploy the exact signed candidate to Forge staging `eu-central-3` using a bucket
and bucket-scoped key created through the existing Fil One flow. Establish an
AWS CLI or boto3 S3 baseline, then test the same bucket through `filfs` and
static CSI. The run covers byte identity in both directions, multipart
completion, 10,000-key listings, supported key characters, out-of-band changes,
throttling, retry ambiguity, credential rotation and revocation, Object Lock
refusal, multi-pod reads, node restart, cleanup, and cross-volume/RBAC attacks.

The pilot then runs the FIL-281-style long backup, followed by a complete
restore and integrity verification. Raw evidence records the environment,
candidate SHA, artifact digests, commands, results, and explicit skips.

Rollback is intentionally simple: stop new mounts, unpublish CSI volumes,
unmount `filfs`, and revoke the pilot key if needed. The bucket remains an S3
bucket and the workload returns to the existing S3 path; no POSIX-owned tenant,
metadata service, or provisioned bucket must be migrated or unwound.

Production publication additionally requires the Linux and Kubernetes
architecture matrix, signed deb/rpm/tar/OCI/Helm artifacts with SBOM and
provenance, install/upgrade/rollback tests, a two-week crash/restart and restore
soak, support and operational ownership, Finance approval, and release-signing
authority. These remain release gates, not promises made by this RFC.

Execution stays in the existing team work: FIL-839 and FIL-524 for staging and
operations; FIL-521 and FIL-536 for usage ownership; FIL-971 and FIL-892 for
backup/restore evidence; and FIL-931 and FIL-980 for SLA and observability
claims. This RFC does not introduce a parallel delivery taxonomy. If accepted,
at most one Posixmount adoption issue should coordinate those existing owners.

## Deferred work

Dynamic CSI, desktop clients, shared cache, native rename and append, persistent
POSIX metadata, writeback replay, prefix-scoped products, and competitive
performance claims are separate proposals with separate evidence gates. Their
source may remain as explicitly non-release design material, but none can enter
the first-release artifacts or claims.

## Alternatives

### Keep S3 as the only interface

This is the lowest-complexity option and remains the recommended path for
S3-native workloads. It does not help tools that require file paths, so those
workloads either need a migration adapter or cannot use Fil One directly.

### Lightly configure upstream Mountpoint

This minimizes maintained code. It does not by itself bind Fil One region
evidence, credential-file policy, static CSI Secret handling, gateway state,
or release claims to the existing Fil One and Forge seams. We retain imported
Mountpoint components and keep the Fil One-specific layer narrow.

### Use a third-party filesystem client

This transfers some implementation work but not the support, security,
credential, qualification, or semantic-claims responsibility. A third-party
client remains a valid comparison during staging, especially if it can meet the
same bounded contract with less owned code.

## Decisions needed

1. Do we accept the bounded Linux `core` plus static-CSI candidate for Forge
   staging qualification?
2. Do we accept the stated boundary: Posixmount is an S3 data-plane client and
   does not duplicate Fil One, Forge, Ingot, or Clockwork authority?

## References

- [Candidate readiness assessment](https://github.com/fil-one/posixmount/blob/cf6253f16c1ab5d332801b0368833eaf3c0f78d3/RELEASE_READINESS.md)
- [First-release semantics](https://github.com/fil-one/posixmount/blob/cf6253f16c1ab5d332801b0368833eaf3c0f78d3/SEMANTICS.md)
- [Commerce and product authority ADR](https://github.com/fil-one/posixmount/blob/cf6253f16c1ab5d332801b0368833eaf3c0f78d3/docs/adr/0007-commerce-and-product-authority-boundary.md)
- [Large technical design](https://github.com/fil-one/posixmount/blob/cf6253f16c1ab5d332801b0368833eaf3c0f78d3/docs/rfc/2026-08-posix-mount.md)
- [Clockwork adjacent-service boundary](https://github.com/fil-one/clockwork/blob/55b4082d380e086b29da7e76f4e060d19cbb49a6/docs/adjacent-service-integration.md)
- [Service Orchestrator Management API ADR](https://github.com/fil-one/fil-one/blob/main/docs/architectural-decisions/2026-04-service-orchestrator-management-api.md)
