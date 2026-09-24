# RFC: Forge S3 tenant IAM

Status: Experimental

Extends: [Forge S3 tenant management](./2026-06-forge-s3-tenant-management.md)

## Authors

- [Srdjan](https://github.com/pyropy)

## Introduction

Hilt gains principals, bucket policies, and principal-bound access keys, while Ingot enforces the actions those policies grant. Together, they let the Fil One console grant each member access to a subset of an organization's buckets, as decided in the [bucket access by region](https://github.com/fil-one/fil-one/blob/main/docs/architectural-decisions/2026-08-bucket-policies-m2.md) ADR.

An access key gains an optional principal. A key created with a principal derives its authority from the bucket policies that apply to that principal. A key created without one keeps the [parent RFC's](./2026-06-forge-s3-tenant-management.md) behavior and is authorized from the permissions and buckets supplied at creation. The parent RFC's tenant, bucket, delegation, and access-key route mechanics remain unchanged.

## Motivation

Today, a Hilt access key stores a fixed set of permissions and bucket names at creation. Its authority cannot change afterwards, and there is no object between the tenant and its keys to which a later access change can attach. The console therefore cannot grant a member access to a subset of buckets and later widen or narrow that access without reissuing every key the member holds.

For Forge, the ADR defines the principal-bound authorization path: each member is represented as a principal in the storage system; a bucket's policy is the source of that principal's access to the bucket; and every key bound to the principal carries the authority those policies grant. Service keys remain outside this model and continue to use the permissions and bucket scope they were created with.

## Goals

1. The storage system computes a principal's effective actions on a bucket from that bucket's policy alone: `allow` minus `deny`, with explicit `deny` taking precedence.
2. A policy edit changes the authority of every key bound to an affected principal without reissuing any key.
3. Before Hilt acknowledges a policy change, it publishes to Swarf every revocation required by a narrowing of a principal's effective actions or by a widening on a bucket the key already reaches. Ingot's existing firehose consumer then clears the affected caches within firehose latency.
4. The console signs member traffic, including presigned URLs, with a key bound to that member's principal, while retaining service keys for traffic with no member actor.
5. One policy model governs every principal. Existing keys continue to work, and a tenant can migrate to principals without reissuing keys or introducing a window in which the network rejects requests.

## Concepts

### Roles

The parent RFC's roles apply. Swarf is added:

| Name  | Description                                                                                  |
| ----- | -------------------------------------------------------------------------------------------- |
| Swarf | The Forge revocation service. Hilt publishes revocations to it; Ingot consumes its firehose. |

### Terms

- **Member.** A user in a Fil One organization. Fil One assigns each member one of the roles Owner, Admin, Member, or ReadOnly. Hilt is unaware of these roles.
- **Principal.** Hilt's representation of a member: `(tenant, principalId)` and nothing more. The console creates one principal per member. A principal carries no permissions, role, or key material.
- **Bucket policy.** A document attached to a bucket. Each statement has an effect, a set of principals, and a set of S3 actions. A bucket has at most one policy, which is deleted with the bucket.
- **Effective actions.** The S3 actions a principal may perform on a bucket, computed from that bucket's policy.
- **Service key.** The parent RFC's access key. It has no principal and retains the permissions and buckets supplied at creation. It signs traffic with no member actor, such as tenant setup, bucket creation and deletion, and background work. A tenant may hold several service keys.
- **Principal-bound key.** An S3 access key that authenticates a request and identifies a principal. For each bucket the principal can access, the key holds a tenant delegation for every Forge command implied by that bucket's policy. Hilt updates those delegations when the policy changes. At request time, the key's effective permissions on a bucket are exactly those granted to its principal by the bucket's current policy.

## Hilt - Tenant API

All management API additions and changes defined by this RFC MUST be reflected in the [Service Orchestrator Management API](https://github.com/fil-one/fil-one/blob/main/docs/service-orchestrator-integration/management-openapi.yaml) OpenAPI specification as part of the implementation.

`POST /tenants/{tenantId}/access-keys` creates both key types. The request body has one of two shapes, and that shape determines the key type:

| Body                                           | Kind                | Authority                                                                     |
| ---------------------------------------------- | ------------------- | ----------------------------------------------------------------------------- |
| `{ name, permissions, buckets?, expiresAt? }` | Service key         | The `permissions` and `buckets` in the request, as the parent RFC specifies.  |
| `{ name, principalId, expiresAt? }`            | Principal-bound key | Whatever the bucket policies give the principal, held as delegations that follow the policies. |

Hilt MUST return 422 when a request includes `principalId` together with `permissions` or `buckets`. A principal-bound key has no permissions of its own, so accepting those fields would imply authority that the key does not carry. The management API requirement that `permissions` be present applies only to the service-key shape.

The existing `GET`, `DELETE`, and tenant key-list operations apply to both key types.

### Tenant creation

`PUT /tenants/{tenantId}` is unchanged. Its body currently names the region, and Hilt binds the tenant to the provider that serves it. This RFC does not change Hilt's tenant-to-region model; changes that allow a tenant to span multiple Forge regions are tracked separately in [FIL-1133](https://linear.app/filecoin-foundation/issue/FIL-1133). As today, the console creates the tenant and then creates a service key with no bucket list for bucket provisioning. Nothing in the IAM model below depends on a tenant serving exactly one region.

### Service keys

A service key is the parent RFC's access key, unchanged: an ed25519 `did:key` whose private key is stored at `/tenant/{tenantDID}/access-key/{accessKeyDID}` in the vault. Hilt stores one delegation for each bucket and Forge command derived from the key's permissions. When the key has no bucket list, Hilt instead stores the corresponding powerline delegations with an undefined subject.

A service key:

- Reaches the buckets in its list, or every bucket in the tenant when no list is present. It is never named in a policy statement, and no policy is evaluated for it.
- MAY hold `s3:CreateBucket` and `s3:DeleteBucket`, which no policy grants (see [action vocabulary](#action-vocabulary)).
- Appears in `GET /tenants/{tenantId}/access-keys`, counts toward `accessKeyCount`, and is deleted through the access-key delete route, as today.
- Keeps the parent RFC's optional `expiresAt`. Rotation consists of minting a new key, moving the caller to it, and deleting the old key.

Because its bucket list is fixed at creation, a service key scoped to named buckets cannot reach buckets created later. Tenant setup, bucket creation, and bucket deletion therefore use a service key with no bucket list.

Deleting a service key follows the parent RFC: Hilt publishes a revocation for each delegation, then deletes the delegations, key row, and vault entry.

### Principals

Routes, all under `/tenants/{tenantId}/principals` and authenticated with the partner key:

| Method | Path                                    | Purpose                                                                          |
| ------ | --------------------------------------- | -------------------------------------------------------------------------------- |
| PUT    | `/principals/{principalId}`             | Create the principal: 201; 200 if it exists. Idempotent.                         |
| GET    | `/principals`                           | List principals.                                                                 |
| GET    | `/principals/{principalId}`             | Principal detail.                                                                |
| DELETE | `/principals/{principalId}`             | Remove the principal (see [removal](#principal-removal)). 204 if already gone.   |
| GET    | `/principals/{principalId}/policies`    | Every bucket policy with a statement naming the principal or `*`.                |
| GET    | `/principals/{principalId}/access`      | The principal's effective actions per bucket.                                    |
| GET    | `/principals/{principalId}/access-keys` | List the principal's keys.                                                       |

A principal-bound key is created through the tenant access-key route with `principalId` set (see [principal-bound access keys](#principal-bound-access-keys)).

`principalId` is an opaque caller-supplied string that is unique within the tenant. Hilt returns 422 for an empty ID, an ID longer than 255 characters, or the reserved ID `*`.

A principal is represented as:

```jsonc
{
  "principalId": "8f2c...",
  "createdAt": "2026-09-09T10:00:00Z"
}
```

Creating a principal stores only the principal row. A principal has no DID, vault entry, or delegation and appears in no UCAN. Policy authority reaches the storage system through the principal's keys: each key holds tenant delegations over the buckets its principal may access, and on each request Hilt evaluates the bucket policy for that principal and re-delegates from the key to Ingot. As with a service key, the proof chain runs from the bucket to the tenant to the key to Ingot.

Ingot caches a principal-bound key exactly as it caches a service key: proof chains, the effective actions on the addressed bucket, the derived signing key, and the tenant, all until the next UTC midnight (see [Ingot](#ingot---s3-api)). Ingot consults Hilt on a cache miss and after a revocation. Requests from a warm key proceed from Ingot to Sprue and Piri without a Hilt round trip. Designs that give a principal key material or clear Ingot's cache without a stored grant are covered in [alternatives considered](#alternatives-considered).

The principal's effective actions are read through `GET /principals/{principalId}/access`:

```jsonc
{
  "buckets": [
    { "name": "photos", "actions": ["s3:GetObject", "s3:ListBucket"] },
    { "name": "backups", "actions": ["s3:DeleteObject", "s3:GetObject", "s3:PutObject"] }
  ]
}
```

Buckets with an empty effective set are omitted. Hilt computes the result from its own tables on every call and does not consult another service, so the read reflects Hilt's latest committed write.

A removed principal remains as a tombstone (see [removal](#principal-removal)). It is absent from `GET` and list results, holds no keys, and is named in no statement. A `PUT` for the same ID revives it with no attached state.

### Bucket policies

A bucket has at most one policy, addressed by the bucket's name:

| Method | Path                                              | Purpose                                                        |
| ------ | ------------------------------------------------- | -------------------------------------------------------------- |
| GET    | `/tenants/{tenantId}/buckets/{bucketName}/policy` | Read the policy. Returns an `ETag`.                            |
| PUT    | `/tenants/{tenantId}/buckets/{bucketName}/policy` | Create or replace. Carries `If-None-Match: *` or `If-Match`.   |
| DELETE | `/tenants/{tenantId}/buckets/{bucketName}/policy` | Delete. Carries `If-Match`.                                    |

The document follows the shape of an AWS bucket policy, limited to the fields Forge evaluates:

```jsonc
{
  "statement": [
    {
      "sid": "filone-owners",                       // optional label; Hilt stores and returns it, nothing evaluates it
      "effect": "allow",                            // "allow" | "deny"
      "principal": ["8f2c...", "a91e..."],          // principal IDs, or the string "*" for every principal of the tenant
      "action": ["s3:GetObject", "s3:ListBucket"]   // policy vocabulary, or ["s3:*"] for all of it
    },
    {
      "effect": "deny",
      "principal": "*",
      "action": ["s3:PutObjectRetention", "s3:PutObjectLegalHold"]
    }
  ]
}
```

Fil One writes its default statements under the labels `filone-owners`, `filone-admins` and `filone-creator`, and finds them again by those labels when a member's role changes. To Hilt they are ordinary labels: any statement may carry any `sid`.

Hilt MUST reject with 422:

- an action outside the [policy vocabulary](#action-vocabulary); in particular `s3:CreateBucket`, `s3:DeleteBucket`, and `s3:ListAllMyBuckets`. `s3:*` is accepted and stands for the whole vocabulary,
- a `principal` that is neither the string `"*"` nor a non-empty list of ids of live principals of the tenant. `"*"` inside a list is rejected; the wildcard has one spelling,
- an empty `statement` list or a statement with an empty `action` list. The caller deletes the policy instead,
- a field the schema does not define. There is no `resource` field: the policy governs the bucket it is stored on.

A bucket that does not exist or belongs to another tenant is 404 on all three routes. `GET` and `DELETE` on a bucket that has no policy are 404 as well, with a `PolicyNotFound` body that tells the two cases apart.

Hilt MUST accept a `deny` statement whose `principal` is `"*"`. Hilt does not derive the Fil One role or identity of the member who initiated a management-API request. The console is responsible for deciding whether that member may write such a statement before calling Hilt.

**Evaluation.** The effective actions of principal `p` on bucket `b` are

```
effective(p, b) = union(action of allow statements naming p or "*")
                \ union(action of deny statements naming p or "*")
```

where `s3:*` expands to every action in the policy vocabulary. Hilt returns the set in sorted order. A bucket with no policy has an empty effective set for every principal. Service keys are not principals, so bucket policies are never evaluated for them.

`s3:ListAllMyBuckets` sits outside the per-bucket action set and is granted to every principal. The S3 `ListBuckets` operation therefore requires no policy lookup. This is distinct from listing objects within a bucket, which requires `s3:ListBucket` and is granted through the bucket policy like any other action.

**Compare-and-set.** `GET` returns an `ETag` computed over Hilt's canonical encoding of the stored document. `PUT` and `DELETE` MUST carry exactly one precondition: `If-Match` with that value, or, on a `PUT` that creates the first policy, `If-None-Match: *`. A request with neither, with both, or with another `If-None-Match` value is 400. A mismatch, or `If-None-Match: *` when a policy exists, is 412 and nothing is written. A successful `PUT` answers 201 when it created the policy and 200 when it replaced one, with the new `ETag` in the response header and no body. Callers treat the ETag as opaque.

**Storage.** Hilt stores the document with its bucket and maintains an index from each principal to the buckets whose statements name that principal, making the two principal reads above index lookups. Statements naming `"*"` are indexed at the tenant level. The policy row is deleted with the bucket.

### Bucket creation

Buckets are created and deleted over S3 only with a service key. `CreateBucket` cannot be granted by a bucket policy because the target bucket does not exist yet. This RFC also deliberately excludes `DeleteBucket` from the policy vocabulary so that bucket lifecycle operations remain service-key-only, matching the ADR's decision that member keys cannot create or delete buckets. A principal-bound key therefore receives `AccessDenied` from both operations. `aws s3 mb` with a member's key fails, and members create buckets through the console. The user-facing S3 documentation needs to state this limitation.

A new bucket's policy is carried on the create request in the Forge-specific `x-bucket-policy` header as base64-encoded JSON; AWS does not define this header. Ingot forwards the S3 request to Hilt in `/s3/bucket/create` as it does today, headers included, so the header requires no new Ingot behavior. Hilt decodes it, validates it exactly as `PUT .../policy` would, creates the bucket, writes the policy, and issues every key of each named principal its delegations over the new bucket. If the policy write or the issuance fails, Hilt deletes the bucket with whatever it had written. No bucket outlives a failed write of the policy its create request carried. The header MUST be among the request's `SignedHeaders`; Hilt refuses a create whose policy header is present and unsigned, since otherwise anything on the path could replace the document. A document that fails validation refuses the create with `InvalidBucketPolicy`, which Ingot renders as `InvalidArgument` (400). Ingot caps the request head at 8 KB, leaving roughly 5 KB for the decoded JSON policy under the expected request headers. The console MUST check the encoded request size before sending the create and refuse to submit a request that would exceed Ingot's header limit; an oversized request may otherwise be rejected by Ingot before it reaches Hilt.

The console includes this header on every bucket it creates. The document names the member who created the bucket and the organization's current Owners and Admins, each with the actions their role permits. A bucket created without the header, by a generic S3 client holding a service key, has no policy, and only service keys reach it until a policy is written through the management API. The policy can be added later through the Fil One console, which writes it through that management API.

### Principal-bound access keys

`POST /tenants/{tenantId}/access-keys`

```jsonc
{
  "name": "laptop",                            // unique per principal
  "principalId": "8f2c...",                    // makes the key principal-bound
  "expiresAt": "2026-12-31T00:00:00Z"          // optional
}
```

`principalId` is carried in the request body rather than the path so that the existing `POST /tenants/{tenantId}/access-keys` route can create both key types without breaking the console's current calls. The request body shape determines the key type. A second route, `POST /tenants/{tenantId}/principals/{principalId}/access-keys`, is discussed under [alternatives](#separate-creation-routes-per-kind).

Hilt MUST generate and store the key exactly as the parent RFC specifies, record the principal on the key row, and store `NULL` for both permissions and bucket list. It returns 422 when `principalId` does not identify a live principal of the tenant and 409 when the name is already used by another key of that principal. The response is the parent RFC's `CreatedAccessKey` with a `principal` field carrying the `principalId`, and no `permissions` or `buckets`.

Hilt MUST also issue the key its delegations and store them in the `delegation` table with the tenant's other grants. It reads the principal's effective actions per bucket, the computation behind `GET /principals/{principalId}/access`, and for every bucket with a non-empty set issues one delegation per Forge command those actions map to (see [action vocabulary](#action-vocabulary)):

```jsonc
{
  "iss": "did:plc:tenant",
  "aud": "did:key:accessKey",
  "sub": "did:key:bucket",
  "cmd": "/blob/add",
  // exp: the key's expiresAt, or none
}
```

For example, if a member's policies grant `s3:PutObject` on `photos` and `s3:GetObject` on `backups`, each of the member's two keys receives six delegations over `photos` and one over `backups`. The stored set for a key and a bucket is always `CommandsFor(effective(principal, bucket))`. Every write below keeps that inside its transaction, so a request that passes the action check always resolves a chain, and a key whose principal reaches no bucket holds no delegations. ucantone gives every delegation a random nonce, so a reissued set never carries a CID that was revoked. The shape is the parent RFC's: a service key scoped to named buckets holds the same rows, derived from its permission list.

A service key name remains unique within the tenant, as today. A principal-bound key name is unique only within its principal, so two members may each hold a key named `laptop`.

`GET /tenants/{tenantId}/access-keys` and `GET /tenants/{tenantId}/access-keys/{accessKeyId}` return the `principal` field. A key without it is a service key and carries its own `permissions` and `buckets`. The key list does not carry each key's effective actions; the console reads them once per principal through the principal route.

`DELETE /tenants/{tenantId}/access-keys/{accessKeyId}` deletes either kind in the same sequence: a revocation for each of the key's delegations, published in one Swarf request, then the delegations, the row, and the vault entry. Ingot drops that key's per-key cache and no other.

Both key types count toward `accessKeyCount`. Hilt enforces no key limit today, and this RFC adds none.

### Policy changes and delegation rotation

A policy `PUT` or `DELETE` first determines the affected principals from the union of the old and new documents. If either document contains a statement whose `principal` is `"*"`, the affected set expands to every live principal in the tenant. Hilt computes each affected principal's effective actions on the bucket before and after the change. If the set changed, Hilt rewrites every bound key's delegations over that bucket before acknowledging the write: it publishes a revocation for each existing delegation, then issues and stores the delegations required by the new effective set. A revocation is a `/ucan/revoke` invocation sent to Swarf and signed by the tenant that issued the delegation. All revocations produced by one policy write are sent in a single Swarf request (see [Swarf](#swarf)). Delegations for other buckets are unchanged.

A principal's first grant on a bucket publishes no revocation because the key previously held no delegation for that bucket. Ingot likewise has no cached action set for a bucket it has never seen under that key, so the next request reaches Hilt and is evaluated against the new policy. Widening access on a bucket the key already reaches requires the same rotation as narrowing access: Ingot rejects actions outside its cached set locally, so without rotation the new grant would not become visible until the cache expired at midnight.

Ingot's per-key cache contains the proof chains, effective action sets, derived signing key, and tenant, and every proof chain includes one of the key's stored delegations. When Ingot consumes a revocation for any delegation in those proof chains, it drops all cached state for that access key. As a result, revoking a key's delegations for one bucket also clears that key's cache for every other bucket; the next request returns to Hilt and refills the cache from the current policy. Revocation remains exact at Hilt and Swarf, while Ingot pays the broader cost of a cache refill.

**Locking.** Hilt takes no row locks today; this RFC introduces them on three paths. A policy write runs in a single Postgres transaction. It locks the policy row with `SELECT ... FOR UPDATE`, takes a transaction-scoped advisory lock on the bucket, and locks each affected principal row with `SELECT ... FOR UPDATE` before enumerating that principal's keys. Rotating a key also takes an advisory lock on that key's delegation set. Hilt publishes revocations while these locks are held and commits only afterwards.

`/s3/request/authorize` reads the key, principal, and policy rows with `SELECT ... FOR SHARE`, and reads the key's delegations before reading the policy. Key creation reads the principal row with `SELECT ... FOR SHARE` and materializes delegations from the policies visible to that transaction. An authorize request arriving after revocation publication but before commit therefore waits for the commit and is answered from the new policy with the new delegations. Likewise, a key created while a policy write is in flight is either included in the rotation or created from the committed policy. Without these locks, an authorize request during a write would be answered from the old policy with delegations Ingot has seen revoked, and every request from the key would reach Hilt until the commit (see [error recovery](#error-recovery)).

During a write to a bucket's policy, authorize requests for that bucket may wait for one Swarf round trip because all revocations from the write are sent in one request. The write keeps its transaction open while waiting for Swarf, up to 8 seconds; if it has not completed by then, it fails and rolls back. Readers and writers also fail if they cannot acquire their locks within 10 seconds, ensuring that a slow Swarf releases the locks before waiting readers time out. A timed-out management write returns 409 `ConcurrentChange`. A timed-out authorize returns `TemporarilyUnavailable`, which Ingot renders as `ServiceUnavailable` (503).

The locks do not cover an authorize request that completed before the policy write acquired its locks but whose response reaches Ingot after the revocation. [Error recovery](#error-recovery) covers that response.

**Failure.** A write that fails at any step, before or after publishing its revocations, is covered in [error recovery](#error-recovery).

Deleting a key revokes only that key's delegations. The principal's other keys keep their cached proofs.

The staleness bound for a policy change is the interval between Hilt acknowledging the change and Ingot consuming the corresponding firehose record. It is bounded by firehose latency and is independent of the midnight cache horizon. A revocation clears only Ingot's caches. Sprue and Piri do not consult Swarf, so a per-request delegation already issued by Ingot remains valid there until it expires. That does not extend usable access beyond Ingot: Ingot is the delegation's audience and signs every invocation that carries it. Once revocation clears Ingot's cache, the next request returns to Hilt and is evaluated against the new policy.

### Error recovery

Every write that publishes revocations (a policy change, a key's deletion, a principal's removal, a bucket's deletion, a tenant's removal) runs in one transaction and publishes before it commits. Hilt enforces exactly the state it stores, so a failed write leaves nothing at Hilt to reconcile. What it leaves is the caller's unmet intent. The console owns completing it and SHOULD retry from a durable job rather than from the member's request, so that a member who gives up after one attempt does not leave a removal or a policy change unapplied.

**Swarf rejects the publish, or the publish fails.** Hilt returns 500 and commits nothing. The old state remains in force, so no principal receives authority beyond the committed state.

**Swarf accepts the revocations and the commit fails.** The delegations are revoked at Swarf and still stored in Hilt. Ingot consumes the revocations, drops each affected key's per-key cache, and records each revoked CID in an in-memory set kept until the next UTC midnight plus clock skew, the same horizon as the cache. On the key's next request Ingot misses the cache, calls Hilt, and receives the same stored delegations. Their CIDs are in the set, so Ingot authorizes the request from the response and caches nothing. Every request from that key reaches Hilt until a committed write returns delegations with new CIDs. The member holds exactly the access the committed state grants, at one Hilt round trip per request.

The console's retry republishes the same revoke invocations, which Swarf records as duplicates and does not emit again. The committed write stores fresh delegations, and ucantone's random nonce gives them new CIDs. The key's next request caches them. The change reaches Ingot one request after the commit.

If the console stops retrying, the old state stays in force and Hilt holds nothing partially applied. The key pays a Hilt round trip per request until midnight, when Ingot drops the set and caches the stored delegations again. A retry that commits after that point reaches Ingot at the following midnight.

**A response that outran a write.** An authorize response that left Hilt before a write took its locks can arrive at Ingot after the write's revocations. Its delegations are in the set, so Ingot serves that one request under the old state and caches nothing. Swarf polls its store once per second before emitting a record, so the response must stay in flight longer than that interval for this to occur.

**Restart.** Ingot keeps the set in memory and a restart clears it together with the cache. A restart between a failed commit and its successful retry refills the cache from the stored delegations, and the retry's duplicate revocations emit nothing, so that key sees the change at midnight.

### Principal removal

`DELETE /tenants/{tenantId}/principals/{principalId}` locks the principal row with `SELECT ... FOR UPDATE` and, in one transaction and in order:

1. Publishes a revocation for every delegation of each of the principal's keys, in one Swarf request.
2. Removes the principal from every statement naming it. A statement left with no principal is deleted, and a policy left with no statement is deleted.
3. Deletes the principal's keys: rows, then vault entries.
4. Sets `deleted_at` on the principal row and commits, releasing the lock.

The principal row remains as a tombstone. A removed principal is absent from `GET` and the list, is refused with 422 as the `principalId` of a new key, and is skipped by evaluation should a statement still name it. An authorize request for one of its keys waits on the lock until the removal commits or rolls back, then is refused with `UnknownAccessKey` because the key is gone.

A failure at any step rolls back the transaction. The principal keeps every key and every statement it had, so a failed removal changes nothing at Hilt. The revocations already published stand, and the console retries (see [error recovery](#error-recovery)). The call answers 204 once the principal is removed, and 204 for an id that is removed or never existed.

A `PUT` for a removed principal's id clears `deleted_at`. The principal returns with no keys and named in no statement, so nothing of its earlier access survives, whether the id belongs to the same member re-invited or to a different member given the same id.

### Bucket deletion

`/s3/bucket/delete` deletes the bucket's delegations, policy, and bucket row. Hilt revokes every service-key and principal-bound-key delegation over the bucket in a single Swarf request. Ingot removes the bucket from its local database as part of the same request, and local authorization rejects any bucket that is no longer present, so cached chains for a deleted bucket cannot be used.

### Tenant removal

`DELETE /tenants/{tenantId}` removes a disabled tenant. It MUST publish a revocation for every delegation the tenant's keys hold before it deactivates the tenant's `did:plc`, because the tenant signs the revocations and Swarf verifies them against the tenant's DID. It then deletes the tenant's principals, policies, keys, buckets, and delegations. Ingot drops the per-key cache for every tenant key when the records arrive. Today the deletion publishes nothing, and a warm key keeps serving from Ingot's cache until the next UTC midnight.

## Hilt - UCAN API

The commands and their argument and result types retain their existing shapes. The authorization steps gain a branch on how the permitted actions are derived, and bucket creation gains the policy header.

### `/s3/request/authorize`

For a request signed with a service key, the parent RFC's authorization steps apply unchanged: the action must appear in the key's `permissions`, and the bucket must appear in its `buckets` unless the key has no bucket list.

For a request signed with a principal-bound key, Hilt performs the parent RFC's steps through signature verification, tenant status, issuer, and region, then:

1. Derive the S3 operation and its action from method, path, and query (see [action vocabulary](#action-vocabulary)).
2. Load the key's principal. A key whose principal is removed is refused with `UnknownAccessKey`.
3. If the operation is `ListBuckets`, authorize it: every principal holds `s3:ListAllMyBuckets`.
4. If the operation is `CreateBucket` or `DeleteBucket`, refuse with `OperationNotPermitted`. No policy grants them.
5. Resolve the bucket by name. If it does not exist, or belongs to another tenant, refuse with `UnknownBucket`.
6. Compute `effective(principal, bucket)`. If it is empty, refuse with `UnknownBucket`.
7. If the action is not in the effective set, refuse with `OperationNotPermitted`. A copy evaluates the source bucket against its own policy the same way.
8. Re-delegate from the key to Ingot, one per-request delegation per Forge command mapped from the action, signed with the access key as the parent RFC's path does:

```jsonc
{
  "iss": "did:key:accessKey",
  "aud": "did:web:ingot",       // the invocation issuer
  "sub": "did:key:bucket",
  "cmd": "/content/retrieve",
  // exp: next UTC midnight plus clock skew, capped at the key's expiry
}
```

Expiry follows the parent RFC. The response container carries the per-request delegations, which are not stored, each keyed to itself as the parent RFC's path does; Ingot obtains the chain from the bucket root through the key's stored delegations over `/s3/bucket/info`. The key, principal, and policy rows are read with a share lock (see [policy changes](#policy-changes-and-delegation-rotation)). The path reads the access key's private key from the vault, as the parent RFC's does. A request that passes step 7 always resolves a chain, because the key's stored delegations over the bucket are exactly the commands its effective actions map to.

**Result.** The parent RFC's `AuthorizeOK`, with `tenant` carrying the tenant and `permissions` carrying the key's effective actions on the addressed bucket, sorted. For `ListBuckets` the effective actions are `["s3:ListAllMyBuckets"]`. For a service key `permissions` carries the key's own permission set, as the parent RFC specifies.

### `/s3/bucket/info`

For a principal-bound key, the chain consists of the bucket root, the bucket-to-tenant delegation with `cmd: "/"`, and the key's stored delegations over that bucket, matching a service key scoped to named buckets. The access key is a hop in the chain; the principal itself appears in no UCAN.

`InfoOK` keeps its shape; `permissions` carries the key's effective actions on the bucket. A key whose principal cannot reach the bucket receives `UnknownBucket`.

### `/s3/bucket/create` and `/s3/bucket/delete`

A service key reaches them by holding `s3:CreateBucket` or `s3:DeleteBucket` in its permissions. A principal-bound key receives `OperationNotPermitted` from both.

Create reads the `x-bucket-policy` header from the forwarded request when it is present, checks that the header is signed, validates the decoded document as `PUT .../policy` does, creates the bucket, and stores the policy. A failed policy write deletes the bucket (see [bucket creation](#bucket-creation)). An unsigned or invalid document refuses the create with `InvalidBucketPolicy`. Delete removes the policy with the bucket, as described above.

### `/s3/bucket/list`

Unchanged. The operation returns the tenant's complete bucket set, matching AWS behavior in which a listing may include bucket names the caller cannot open. The console filters its own list against the principal's effective actions.

### Failures

`UnknownBucket` covers a bucket that does not exist, belongs to another tenant, or is out of the principal's reach. `OperationNotPermitted` covers an action outside a non-empty effective set and the two bucket operations no policy can grant. `UnknownAccessKey` covers a principal-bound key whose principal is removed. Ingot's existing mapping renders them as `NoSuchBucket` (404), `AccessDenied` (403), and `InvalidAccessKeyId` (403).

This RFC adds two failures. `TemporarilyUnavailable` is an authorize that could not take its share lock within the lock timeout; Ingot renders it as `ServiceUnavailable` (503) and the client retries. `InvalidBucketPolicy` is a create whose policy header is unsigned or fails validation; Ingot renders it as `InvalidArgument` (400).

## Swarf

The Swarf service requires no changes. A rotation, a key's deletion, a principal's removal, and a bucket's deletion publish `/ucan/revoke` for each delegation they invalidate, a UCAN command Swarf serves on its RPC endpoint, self-signed by the tenant that issued the delegation, with no witness path. That is the form of every revocation Hilt publishes. Swarf validates the delegation's signature and expiry, records the revocation, and emits a `revocation` event on `GET /revocations/:from`.

All revocations produced by one write are sent to Swarf in a single request. A ucantone server executes every invocation addressed to it in a request container, and the client library exposes that as `ExecuteBatch`, which returns one receipt per invocation. The Swarf client gains a batch-publish operation that builds one `/ucan/revoke` invocation per delegation and sends them together. Hilt checks every receipt, so any rejected revocation fails the write. A `"*"` policy edit therefore costs one Swarf round trip regardless of how many keys it rotates, although the request size grows with the fan-out (see [open questions](#open-questions)).

## Ingot - S3 API

Ingot's request path is unchanged: it consults Hilt on a cache miss and otherwise authorizes locally from the cached state.

For each access key, Ingot caches the data carried by Hilt's authorize response: delegations, the effective action set for each bucket Hilt names, the derived SigV4 verification key, and the tenant. The cache lasts until the next UTC midnight plus clock skew. On the fast path, Ingot verifies the signature with the cached key, checks that the requested action is present in the bucket's cached effective set, and requires a chain to Ingot's agent for every Forge command to which the action maps. If a cached entry exists but the action is absent or a required chain does not resolve, Ingot rejects the request locally; it does not consult Hilt.

The cached effective set is what makes `deny` enforceable. Several S3 actions map to the same Forge commands: for example, `s3:GetObject` and `s3:ListBucket` both require `/content/retrieve`, while `s3:PutObject` grants `/blob/remove` alongside `s3:DeleteObject`. A chain lookup alone therefore cannot distinguish between actions that share the same commands or honor a policy that grants one but not the other. The cached action set provides that distinction. A request for a bucket with no cached set goes to Hilt.

Operations that map to no Forge command are authorized at Hilt on every request: `ListBuckets`, `CreateBucket`, and `DeleteBucket`. No per-request delegation carries the key's expiry for them, so a cached set alone would outlive an expired key.

**The firehose consumer records what it revokes.** When Swarf reports a revoked CID, Ingot drops every per-key cache whose proof chains contain a delegation with that CID, including the cached effective action sets, signing key, and tenant, and adds the CID to an in-memory set kept until the next UTC midnight plus clock skew. Every cached proof chain for a principal-bound key contains one of the key's stored delegations, so a rotation always names something the cache contains. A revocation that matches no cache still enters the set. On a cache miss, Ingot checks the delegations in Hilt's response against the set. If any is present, Ingot authorizes the request from the response and caches nothing, so a key whose revocations were published without a commit reaches Hilt on every request until a committed write issues fresh delegations (see [error recovery](#error-recovery)).

Error mapping gains two rows: `TemporarilyUnavailable` renders as `ServiceUnavailable` (503) and `InvalidBucketPolicy` as `InvalidArgument` (400). `UnknownBucket` is already 404 and `OperationNotPermitted` already 403.

## Fil One console

The console maintains two key types per tenant.

The service key is the tenant-wide credential the console uses today. It is created without a bucket list and holds `s3:CreateBucket` and `s3:DeleteBucket`. It signs every request that has no member actor and every request no policy can authorize: tenant setup, bucket creation with the policy header, bucket deletion, and background workers. For that traffic the console's own role check is the whole of the enforcement, as today.

For each member, the console stores one principal-bound key for the tenant. It creates the key through the tenant access-key route on that member's first request in the region, persists the returned credential, and reuses it for subsequent traffic from that member rather than minting a new key per request. The console uses that key for all member traffic: listing buckets and objects, reading, writing and deleting objects, and creating presigned URLs. Hilt and Ingot authorize this traffic against the member's bucket policies, so bucket access is enforced by the storage system while the console's role checks remain an additional front-end guard.

Consequences:

- A member's key cannot create or delete a bucket, whether the console or the member created the key. The console signs those two operations with its service key after its own role check, so the tenant-wide credential stays in service.
- Hilt does not distinguish the console's key for a member from a key the member created themselves. Both are bound to the same principal, are authorized against the same policies, and are deleted with the principal.
- A presigned URL is authorized against the member's policies when it is redeemed, under the parent RFC's signature time bounds and the key's own expiry.
- A policy change reaches the member's console traffic the way it reaches every key of theirs: through the rotation of the key's delegations.

## Action vocabulary

The policy vocabulary is Hilt's S3 permission set excluding the three bucket-level actions, plus a wildcard:

| S3 action                             | Forge commands                                                                                 | Note                                   |
| ------------------------------------- | ---------------------------------------------------------------------------------------------- | -------------------------------------- |
| `s3:GetObject`                        | `/content/retrieve`                                                                            |                                        |
| `s3:GetObjectVersion`                 | `/content/retrieve`                                                                            |                                        |
| `s3:GetObjectRetention`               | `/content/retrieve`                                                                            |                                        |
| `s3:GetObjectLegalHold`               | `/content/retrieve`                                                                            |                                        |
| `s3:PutObject`                        | `/blob/add`, `/index/add`, `/upload/add`, `/content/retrieve`, `/blob/abort`, `/blob/remove`   |                                        |
| `s3:PutObjectRetention`               | as `s3:PutObject`                                                                              |                                        |
| `s3:PutObjectLegalHold`               | as `s3:PutObject`                                                                              |                                        |
| `s3:DeleteObject`                     | `/blob/remove`, `/upload/remove`                                                               |                                        |
| `s3:DeleteObjectVersion`              | `/blob/remove`, `/upload/remove`                                                               |                                        |
| `s3:ListBucket`                       | `/content/retrieve`                                                                            |                                        |
| `s3:ListBucketVersions`               | `/content/retrieve`                                                                            |                                        |
| `s3:ListBucketMultipartUploads`       | `/content/retrieve`                                                                            |                                        |
| `s3:ListMultipartUploadParts`         | `/content/retrieve`                                                                            |                                        |
| `s3:AbortMultipartUpload`             | `/blob/abort`, `/blob/remove`                                                                  |                                        |
| `s3:*`                                | every row above                                                                                | policy documents only                  |

Excluded from policies: `s3:CreateBucket` and `s3:DeleteBucket`, because a principal holding them would act outside the policy that granted it, and `s3:ListAllMyBuckets`, which every principal holds. `s3:*` never expands to any of the three. A service key still holds the two bucket actions through its own permissions, which is how the console creates and deletes buckets.

Bucket-configuration reads (`GET /{bucket}?versioning`, `GET /{bucket}?object-lock`) classify as `ListBucket`, as they do today, so `s3:ListBucket` covers them and Ingot's fast path applies. The management API's action enum, now used by policy statements, gains `s3:AbortMultipartUpload` and `s3:ListMultipartUploadParts`, which Hilt already accepts.

Retention and legal-hold writes pass through the API like any other action. The rule that only an Owner may grant them is the console's, and Hilt does not know it.

## Migration

Every existing key becomes a service key because a key without a principal is exactly the key defined by the parent RFC. Existing keys are preserved because they are embedded in customer workflows. They continue to work unchanged and can be rotated by customers on their own schedule.

**Once per Forge network.**

1. Deploy Ingot with the two new error mappings and the revoked set. Ingot already enforces the cached effective action set. Swarf's service needs no deployment; Hilt takes the client library with the batch publish.
2. Deploy Hilt. Its schema migration adds the principal and policy tables and the `principal_id` column on keys. Tenants, buckets, keys, vault entries, and delegations are untouched, and every key keeps authorizing through the parent RFC's path. Principal-bound keys are new, so every one is created with its delegations and nothing is backfilled.

**Once per organization.** The console creates one principal per member with `PUT /tenants/{tenantId}/principals/{principalId}`, then writes a policy for each bucket naming the organization's Owners and Admins. This step does not yet change member behavior: the console continues to sign member traffic with its service key, and service keys are not evaluated against bucket policies.

**Once per region.** After every organization in the region has principals and policies, the console switches that region's access model from `scoped-keys` to `iam`. The access model is configured per region; policy routes, policy fan-out, and per-member keys are active only where the model is `iam`. From that point forward, the console includes `x-bucket-policy` on every bucket it creates.

**Once per member.** On the member's first request after the regional switch, the console creates a key bound to that member's principal, persists the returned credential, and reuses it for subsequent member traffic. Until that happens, the console continues to sign the member's traffic with the service key, which reaches every bucket in the tenant and is not subject to bucket policies. Policy changes therefore affect a member's console traffic only after the member has a principal-bound key.

No migration step takes an existing key out of service.

## Alternatives considered

### A principal invalidation record

Swarf could add a `/principal/invalidate` command and a `principal` firehose event carrying a tenant and `principalId`, while Ingot's consumer could drop every per-key cache whose authorize result carried that pair. A policy change would publish one record per changed principal rather than one revocation per key, and Hilt would store nothing per key. Nothing in the record is a delegation, so Swarf could check no proof and would have to trust a configured publisher list. Ingot would index its per-key caches by tenant and principal, and one key's deletion would clear every key the principal holds. Policy-derived delegations cost more rows and revocations per change and change neither Swarf nor Ingot's consumer, and what they revoke is a grant.

### A marker delegation

Each principal-bound key could hold one tenant-issued delegation for a command no service handles. Every authorize response would carry that delegation so Ingot's per-key cache always contains it, and policy changes would rotate it. It costs one row per key and makes every change one revocation per key. The revoked delegation grants nothing, so Ingot dropping the per-key cache on its revocation rests on an agreement between Hilt and Ingot recorded only here, and Sprue and Piri never see it. Policy-derived delegations revoke the grant that changed.

### Recorded per-request delegations revoked on narrowing

Hilt could persist every per-request delegation it issues and revoke the outstanding delegations for a principal and bucket when access narrows. The revocation would be exact to the bucket, and downstream services would never see a wider grant than the policy. It adds a table that churns once per key per bucket per day, and the stored delegations already name the bucket and command, so it adds no precision. Revoking them would also cut in-flight work at Sprue and Piri.

### A did:key principal in the chain

Each principal could instead be an ed25519 key stored by Hilt, with one tenant delegation included in every proof chain and the principal delegating onward to its access keys. A policy change would then rotate one delegation per principal rather than one set per key, and a did:key principal could sign its own invocations one day. It costs a vault entry and a stored delegation per member, a hop in every proof chain, and a chain that names the tenant's whole authority rather than the buckets the policies grant. Delegating to the key keeps principals free of key material and revokes exactly the grant a policy change removes.

### Policies evaluated at Ingot

Ingot embeds versitygw, whose policy engine is currently disabled by assigning the admin role to every authenticated request. Storing policies at Ingot and enabling that engine would evaluate them at the edge with no Hilt round trip. It puts the rule outside the system that owns it, requires policy persistence and replication at every Ingot instance, and leaves Hilt unable to issue delegations that match the decision.

### Separate creation routes per kind

`POST /tenants/{tenantId}/service-credentials` and `POST /tenants/{tenantId}/principals/{principalId}/access-keys` would make a key's kind a property of the route it was created at, so each body would hold only the fields its kind uses and there would be no field combination to reject. A service credential would reach every bucket of the tenant rather than carrying a permission list, and the scoped-key path would leave Hilt entirely. The cost is a second set of create, list, and delete routes, a second vault path, a change to the management API contract the console already calls, and a migration that revokes and deletes every existing key before any member can be served. One route whose body takes one of two shapes leaves the console's existing calls working and keeps the parent RFC's authorization path in place for traffic that has no member actor.

### An access model flag per tenant

A tenant could carry an access-model flag, `scoped-keys` or `iam`, and Hilt could apply that model to every key the tenant holds. Each request would then read one model from the tenant row rather than branching on the key. A tenant cannot hold a worker key and a member key at once under that rule, and the console holds both from the moment it creates its first principal.

### A permission list on a principal-bound key

A principal-bound key could carry its own permission list, similar to an AWS session policy, and Hilt could authorize against the intersection of that list and the bucket-policy result. This would let a member hold, for example, a read-only key and a full-access key for the same buckets without modeling them as separate principals. The trade-off is a second authorization document on every request: a 403 could come from either the key or the bucket policy, and Ingot's cached set would no longer represent the principal's effective actions directly. Rejecting key-level permission fields with 422 preserves a single source of truth for what a member may do.

### A default policy template per tenant

Hilt could store one policy template per tenant and copy it onto each bucket at creation, eliminating the policy from the create request. The console would then have to keep that template synchronized with the organization's roster, creating a second copy of the same statements already written to each bucket. The template also could not naturally include the member who initiated a specific bucket creation. Carrying the policy in the request header sends the document once, at the point where it is needed.

### The policy written after the create

The console could create the bucket over S3 and then write its policy through the management API, responding to the member only after both operations complete. A console failure between those operations, however, could leave a bucket visible in listings but inaccessible to every member, with no event to trigger another policy write. Carrying the policy on the create request makes bucket creation and initial policy installation one operation.

### Filtering the bucket listing

Hilt PR #48 proposed scoping `/s3/bucket/list` to the buckets reachable by a principal. The proposal was closed because it diverged from AWS behavior, where a listing may include bucket names the caller cannot open. The console filters its own list, and Ingot's behavior is unchanged.

## Open questions

1. What latency target does `GET /principals/{principalId}/access` carry? The console resolves it per request for its bucket list and activity feed. It is an index lookup at Hilt; the number is unmeasured.
2. A policy edit naming `"*"` rewrites the delegations of every key of every live principal of the tenant over that bucket: rows and tenant-key signatures inside the write, and one Swarf request whose size grows with the fan-out. A key's rows number its buckets times the commands its actions map to, up to seven per bucket. There is no figure for principals per tenant or keys per principal on Forge. If the product is large, that write amplification is the cost to watch.
3. Limits on statements per policy and principals per statement. None are specified here beyond the 8 KB request head on a create.

## Evaluation criteria

- The time from a change's acknowledgement to Ingot's first refusal, measured against a warm key. This is the staleness bound the ADR asks Hilt to publish.
- Authorization latency for a service key is unchanged from today's key.
- No existing key stops working at any point in the rollout.
- The console's in-memory IAM fake and Hilt pass the same contract tests.
- A policy `PUT` costs one Swarf revocation and one replacement stored delegation for each affected Forge-command delegation on each key whose principal's effective actions changed.
- Swarf's service needs no changes. Ingot's firehose consumer adds only the revoked set.

## References

- [Forge S3 tenant management](./2026-06-forge-s3-tenant-management.md), the parent RFC.
- [Bucket access by region: scoped keys on Aurora and FTH, IAM on Forge](https://github.com/fil-one/fil-one/blob/main/docs/architectural-decisions/2026-08-bucket-policies-m2.md), the ADR implemented here.
- [Organizations, membership, and roles](https://github.com/fil-one/fil-one/blob/main/docs/architectural-decisions/2026-08-organizations-roles-m1.md), for roles and the permission registry the console applies before calling Hilt.
- [Service Orchestrator Management API](https://github.com/fil-one/fil-one/blob/main/docs/service-orchestrator-integration/management-openapi.yaml), the contract the HTTP additions land in.
- [UCAN revocation](https://github.com/ucan-wg/revocation), for path witnesses.
- [FIL-1133](https://linear.app/filecoin-foundation/issue/FIL-1133), multiple Forge regions per tenant.
- Hilt PR #48, the closed proposal for a scoped bucket listing.

## Appendix

### Schema

The following schema changes extend the parent RFC. Types and constraints follow the existing migrations.

```sql
CREATE TABLE principal (
    tenant_id   TEXT        NOT NULL REFERENCES tenant(id) ON DELETE RESTRICT,
    external_id TEXT        NOT NULL,                     -- principalId
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at  TIMESTAMPTZ,                              -- set by removal; a PUT clears it
    PRIMARY KEY (tenant_id, external_id)
);

-- A key with no principal is the parent RFC's key, authorized from its own
-- permissions and buckets. A key with one is authorized from the bucket policies,
-- which its stored delegations materialize, so both are NULL. Existing rows have
-- no principal and need no backfill.
ALTER TABLE access_key
    ALTER COLUMN permissions DROP NOT NULL,
    ADD COLUMN principal_id TEXT,                         -- principalId
    ADD FOREIGN KEY (tenant_id, principal_id) REFERENCES principal(tenant_id, external_id) ON DELETE RESTRICT,
    ADD CONSTRAINT access_key_principal_unscoped
        CHECK ((principal_id IS NULL AND permissions IS NOT NULL)
            OR (principal_id IS NOT NULL AND permissions IS NULL AND buckets IS NULL));

-- A service key's name stays unique within the tenant; a principal-bound key's
-- name is unique within its principal. access_key_tenant_id_name_key is the name
-- Postgres generated for the parent RFC's UNIQUE (tenant_id, name).
ALTER TABLE access_key DROP CONSTRAINT access_key_tenant_id_name_key;
CREATE UNIQUE INDEX access_key_service_name_idx ON access_key (tenant_id, name)
    WHERE principal_id IS NULL;
CREATE UNIQUE INDEX access_key_principal_name_idx ON access_key (tenant_id, principal_id, name)
    WHERE principal_id IS NOT NULL;

CREATE TABLE bucket_policy (
    bucket_id  TEXT        PRIMARY KEY REFERENCES bucket(id) ON DELETE CASCADE,
    document   JSONB       NOT NULL,
    etag       TEXT        NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Which principals a policy names; "*" statements are indexed with a NULL principal_id.
CREATE TABLE bucket_policy_principal (
    bucket_id  TEXT NOT NULL REFERENCES bucket(id) ON DELETE CASCADE,
    tenant_id  TEXT NOT NULL,
    principal_id TEXT,
    FOREIGN KEY (tenant_id, principal_id) REFERENCES principal(tenant_id, external_id) ON DELETE CASCADE
);
-- Postgres treats NULLs as distinct in a UNIQUE constraint, so the wildcard row gets its own index.
CREATE UNIQUE INDEX bucket_policy_principal_named_idx ON bucket_policy_principal (bucket_id, principal_id) WHERE principal_id IS NOT NULL;
CREATE UNIQUE INDEX bucket_policy_principal_wildcard_idx ON bucket_policy_principal (bucket_id) WHERE principal_id IS NULL;
CREATE INDEX bucket_policy_principal_idx ON bucket_policy_principal (tenant_id, principal_id, bucket_id);
```

The `delegation` table is unchanged. For a principal-bound key, its rows represent policy-derived grants, one per bucket and Forge command. The vault keeps the parent RFC's layout: every key at `/tenant/{tenantDID}/access-key/{accessKeyDID}`, and no entry for a principal.

### Example flows

#### Migrate an existing tenant

1. The tenant's keys keep working as service keys. Fil One changes nothing about them and keeps using the service key it holds.
2. Fil One calls `PUT /tenants/{tenantId}/principals/{principalId}` for every member.
3. Hilt stores each principal.
4. Fil One writes each bucket's policy.
5. On a member's next request, Fil One calls `POST /tenants/{tenantId}/access-keys` with that member's `principalId`, persists the returned credential, and signs that member's subsequent traffic with it.

#### Create a bucket

1. A member creates `photos` in the console. Fil One builds the policy naming the member and the org's Owners and Admins, and sends `PUT /photos` to Ingot signed with its service key, with the document base64-encoded in the signed `x-bucket-policy` header.
2. Ingot invokes `/s3/bucket/create` with the request.
3. Hilt verifies the signature, checks that the header is signed, decodes and validates the document, and creates the bucket and its policy in one transaction.
4. Ingot adds the bucket to its local database and answers the member. Their next request on `photos` reaches Hilt and is authorized against the policy.

#### Grant a member a bucket

1. Fil One reads `GET /tenants/{t}/buckets/photos/policy` and its `ETag`.
2. Fil One writes the document with an added `allow` statement, sending `If-Match`.
3. Hilt compares effective sets before and after for every named principal. The added member gained actions on a bucket their keys held nothing over, so Hilt locks the policy row and the member's principal row, stores the new delegations for each of the member's keys, publishes nothing, and commits the document. The response carries the new `ETag`.
4. Ingot holds no cached set for `photos` under the member's keys, so their next request on it reaches Hilt and is authorized against the new policy.

#### Put an object with a principal-bound key

1. A client sends `PUT /photos/cat.jpg` to Ingot, signed with the member's key.
2. Ingot finds no cached effective set for that key and bucket and invokes `/s3/request/authorize`.
3. Hilt verifies the signature, loads the key's principal, resolves `photos`, computes `effective(principal, photos)`, finds `s3:PutObject`, and signs key-to-Ingot delegations for the six write commands with the access key, expiring at the next UTC midnight. It returns them with their chains through the key's stored delegations.
4. Ingot invokes `/s3/bucket/info` for the bucket root, caches the chains and the effective set under the key, and serves the request.
5. Later requests from the key on `photos` are authorized locally: the action is in the cached set and a chain resolves for each command.

#### Remove a member from a bucket

1. Fil One writes the policy without the member's statement, with `If-Match`.
2. Hilt finds the member's effective set on the bucket shrank to nothing, locks the policy row and the member's principal row, publishes a revocation for each delegation the member's keys hold over `photos` in one Swarf request, deletes those rows, and commits the document. An authorize request for `photos` that arrives meanwhile waits for the commit.
3. Swarf's firehose delivers the revocations. Ingot drops the per-key cache for every key whose cached proof chains contained one of the revoked delegations.
4. On the member's next request, Ingot no longer has cached authorization for the key and calls Hilt's `/s3/request/authorize`. On the removed bucket Hilt returns `UnknownBucket`, which Ingot renders as `NoSuchBucket`.

#### Remove a member

1. Fil One calls `DELETE /tenants/{t}/principals/{principalId}`.
2. Hilt locks the principal row, publishes a revocation for every delegation of the member's keys in one Swarf request, strips the principal from every statement, deletes its keys, sets `deleted_at`, and commits.
3. Ingot drops each key's per-key cache when it consumes the revocation. The keys no longer resolve at Hilt.
4. If the member is re-invited, Fil One calls `PUT` for the same `principalId`. The tombstone clears and the principal starts with no keys and no statements.
