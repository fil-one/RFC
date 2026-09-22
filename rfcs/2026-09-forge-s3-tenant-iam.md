# RFC: Forge S3 tenant IAM

Status: Experimental

Extends: [Forge S3 tenant management](./2026-06-forge-s3-tenant-management.md)

## Authors

- [Srdjan](https://github.com/pyropy)

## Introduction

Hilt gains principals, bucket policies, and access keys bound to principals, and Ingot enforces the actions those policies grant. Together they give the Fil One console what it needs to grant a member a subset of an organization's buckets, as decided in the [bucket access by region](https://github.com/fil-one/fil-one/blob/main/docs/architectural-decisions/2026-08-bucket-policies-m2.md) ADR.

An access key gains an optional principal. A key created with one carries whatever the bucket policies give that principal. A key created without one keeps the [parent RFC's](./2026-06-forge-s3-tenant-management.md) behavior, authorized from the permissions and buckets it was created with. The parent RFC's tenant, bucket, and delegation mechanics stay in force, and its access-key routes are unchanged.

## Motivation

An access key on Hilt is a row of permissions and bucket names fixed at creation. Nothing can change what a key carries afterwards, and there is no object between the tenant and its keys that a change could attach to. The console therefore cannot give a member a subset of buckets today, and cannot widen or narrow that subset later without reissuing every key the member holds.

On Forge the ADR settles three things: each member is a principal at the storage system, a bucket's policy is what grants a principal access to that bucket, and a key belongs to a principal and carries whatever the policies give that principal.

## Goals

1. The storage system computes what a principal may do on a bucket from that bucket's policy alone, `allow` minus `deny`, with an explicit `deny` winning.
2. A policy edit changes what every key bound to a named principal may do, with no key reissued.
3. Every narrowing of a principal's effective actions, and every widening on a bucket a key already reaches, is published to Swarf as a revocation before Hilt acknowledges it, and Ingot's existing firehose consumer clears Ingot's caches within firehose latency.
4. The console signs a member's traffic, presigned URLs included, with a key bound to that member's principal, and keeps service keys for traffic that has no member actor.
5. One policy model governs every principal. Existing keys keep working, and a tenant moves to principals with no key reissued and no window in which the network refuses requests.

## Concepts

### Roles

The parent RFC's roles apply. Swarf is added:

| Name  | Description                                                                                  |
| ----- | -------------------------------------------------------------------------------------------- |
| Swarf | The Forge revocation service. Hilt publishes revocations to it; Ingot consumes its firehose. |

### Terms

- **Member.** A user of a Fil One organization. Fil One gives each member one of the roles Owner, Admin, Member, or ReadOnly. Hilt knows none of them.
- **Principal.** A member as Hilt knows them: `(tenant, principalId)` and nothing more. The console creates one principal per member. A principal carries no permissions, no role, and no key material.
- **Bucket policy.** A bucket's own document of statements. Each statement has an effect, a list of principals, and a list of S3 actions. A bucket has at most one policy, and the policy is deleted with the bucket.
- **Effective actions.** The S3 actions a principal may perform on a bucket, computed from that bucket's policy.
- **Service key.** The parent RFC's access key: it has no principal and carries the permissions and buckets it was created with. It signs traffic that has no member actor, such as tenant setup, bucket create and delete, and background workers. A tenant may hold several.
* **Principal-bound key.** An S3 access key that authenticates a request and identifies a principal. For each bucket the principal can access, the key holds a delegation from the tenant for every Forge command granted by that bucket’s policy. Hilt updates these delegations whenever the policy changes. At request time, the key’s effective permissions on a bucket are exactly those granted to its principal by the bucket’s current policy.

## Hilt - Tenant API

`POST /tenants/{tenantId}/access-keys` creates both kinds of key. The body takes one of two shapes, and the shape decides the kind:

| Body                                           | Kind                | Authority                                                                     |
| ---------------------------------------------- | ------------------- | ----------------------------------------------------------------------------- |
| `{ name, permissions, buckets?, expiresAt? }` | Service key         | The `permissions` and `buckets` in the request, as the parent RFC specifies.  |
| `{ name, principalId, expiresAt? }`            | Principal-bound key | Whatever the bucket policies give the principal, held as delegations that follow the policies. |

Hilt MUST reject with 422 a body carrying `principalId` together with `permissions` or `buckets`. A principal-bound key has no permissions of its own, and accepting the fields would suggest otherwise. The management API's rule that `permissions` is required applies to the service-key shape only.

`GET`, `DELETE`, and the tenant's key list serve both kinds.

### Tenant creation

`PUT /tenants/{tenantId}` is unchanged. Its body names the region, and Hilt binds the tenant to the provider that serves it. One Hilt serves several regions; a tenant serves one. The console creates the tenant and then creates a service key with no bucket list for provisioning buckets, as it does today.

Fil One uses the organization's id as the tenant id. An organization served in two regions of the same Hilt would need two tenants, and this RFC gives them no id. Multi-region tenants are [FIL-1133](https://linear.app/filecoin-foundation/issue/FIL-1133). Nothing below depends on a tenant serving one region.

### Service keys

A service key is the parent RFC's access key, unchanged: an ed25519 `did:key` whose private key is stored at the vault path `/tenant/{tenantDID}/access-key/{accessKeyDID}`. It holds one delegation from the tenant per bucket and Forge command mapped from its permissions, or a powerline delegation with an undefined subject when it was created with no bucket list.

A service key:

- Reaches the buckets in its list, or every bucket of the tenant when it has none. It is named in no statement and no policy is evaluated for it.
- MAY hold `s3:CreateBucket` and `s3:DeleteBucket`, which no policy grants (see [action vocabulary](#action-vocabulary)).
- Appears in `GET /tenants/{tenantId}/access-keys`, counts toward `accessKeyCount`, and is deleted through the access-key delete route, as today.
- Keeps the parent RFC's optional `expiresAt`. Rotation is minting a new key, moving the caller to it, and deleting the old one.

Its bucket list is fixed at creation, so a service key scoped to named buckets cannot reach a bucket created afterwards. Tenant setup, bucket create, and bucket delete therefore use a key with no bucket list.

Deleting a service key publishes a revocation for each of its delegations, then deletes the delegations, the row, and the vault entry, as the parent RFC's key delete does.

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

A key bound to a principal is created at the tenant's access-key route with `principalId` set (see [principal-bound access keys](#principal-bound-access-keys)).

`principalId` is an opaque string supplied by the caller, unique within the tenant. Hilt rejects with 422 an empty id, an id longer than 255 characters, and the id `*`.

A principal is represented as:

```jsonc
{
  "principalId": "8f2c...",
  "createdAt": "2026-09-09T10:00:00Z"
}
```

Creating a principal stores the row and nothing else. A principal has no DID, no vault entry, and no delegation, and it appears in no UCAN. The policies reach the storage system through the principal's keys: each key holds delegations from the tenant over the buckets the policies grant its principal, and on each request Hilt evaluates the bucket's policy for the principal and re-delegates from the key to Ingot. The proof chain runs from the bucket to the tenant to the key to Ingot, as it does for a service key.

Ingot caches what Hilt answers for a principal-bound key exactly as it does for a service key: the proof chains, the effective actions on the addressed bucket, the derived signing key, and the tenant, until the next UTC midnight (see [Ingot](#ingot---s3-api)). Hilt is consulted on a cache miss and after a revocation. A warm key's requests go from Ingot to Sprue and Piri with no Hilt round trip. The designs that give a principal key material, or that clear Ingot's cache without a stored grant, are in [alternatives considered](#alternatives-considered).

The principal's effective actions are read through `GET /principals/{principalId}/access`:

```jsonc
{
  "buckets": [
    { "name": "photos", "actions": ["s3:GetObject", "s3:ListBucket"] },
    { "name": "backups", "actions": ["s3:DeleteObject", "s3:GetObject", "s3:PutObject"] }
  ]
}
```

Buckets with an empty effective set are omitted. Hilt computes the answer from its own tables on every call and makes no call to another service, so the read is consistent with Hilt's last write.

A removed principal is kept as a tombstone (see [removal](#principal-removal)). It is absent from `GET` and the list, holds no keys, is named in no statement, and a `PUT` for its id revives it with nothing attached.

### Bucket policies

A bucket has at most one policy, addressed by the bucket's name:

| Method | Path                                              | Purpose                                                        |
| ------ | ------------------------------------------------- | -------------------------------------------------------------- |
| GET    | `/tenants/{tenantId}/buckets/{bucketName}/policy` | Read the policy. Returns an `ETag`.                            |
| PUT    | `/tenants/{tenantId}/buckets/{bucketName}/policy` | Create or replace. Carries `If-None-Match: *` or `If-Match`.   |
| DELETE | `/tenants/{tenantId}/buckets/{bucketName}/policy` | Delete. Carries `If-Match`.                                    |

The document follows the shape of an AWS bucket policy, reduced to what Forge evaluates:

```jsonc
{
  "statement": [
    {
      "sid": "owners",                              // optional label; Hilt stores and returns it, nothing evaluates it
      "effect": "allow",                            // "allow" | "deny"
      "principal": ["8f2c...", "a91e..."],          // principal ids, or the string "*" for every principal of the tenant
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

Hilt MUST reject with 422:

- an action outside the [policy vocabulary](#action-vocabulary); in particular `s3:CreateBucket`, `s3:DeleteBucket`, and `s3:ListAllMyBuckets`. `s3:*` is accepted and stands for the whole vocabulary,
- a `principal` that is neither the string `"*"` nor a non-empty list of ids of live principals of the tenant. `"*"` inside a list is rejected; the wildcard has one spelling,
- an empty `statement` list or a statement with an empty `action` list. The caller deletes the policy instead,
- a field the schema does not define. There is no `resource` field: the policy governs the bucket it is stored on.

A bucket that does not exist or belongs to another tenant is 404 on all three routes. `GET` and `DELETE` on a bucket that has no policy are 404 as well, with a `PolicyNotFound` body that tells the two cases apart.

Hilt MUST accept a `deny` statement whose `principal` is `"*"`. The management API carries no actor; the console decides whether to write one.

**Evaluation.** The effective actions of principal `p` on bucket `b` are

```
effective(p, b) = union(action of allow statements naming p or "*")
                \ union(action of deny statements naming p or "*")
```

where `s3:*` stands for every action in the policy vocabulary. Hilt returns the set sorted. A bucket with no policy has an empty effective set for every principal. A service key is not a principal and no policy is evaluated for it.

`s3:ListAllMyBuckets` is outside the per-bucket set: every principal holds it. The S3 `ListBuckets` operation, which lists the tenant's bucket names and needs that action, is therefore authorized without reading any policy. It is a different operation from listing the objects in one bucket, which needs `s3:ListBucket` and is granted like any other action.

**Compare-and-set.** `GET` returns an `ETag` computed over Hilt's canonical encoding of the stored document. `PUT` and `DELETE` MUST carry exactly one precondition: `If-Match` with that value, or, on a `PUT` that creates the first policy, `If-None-Match: *`. A request with neither, with both, or with another `If-None-Match` value is 400. A mismatch, or `If-None-Match: *` when a policy exists, is 412 and nothing is written. A successful `PUT` answers 201 when it created the policy and 200 when it replaced one, with the new `ETag` in the response header and no body. Callers treat the ETag as opaque.

**Storage.** Hilt stores the document with its bucket and maintains an index from principal to the buckets whose statements name it, so the two principal reads above cost an index lookup. Statements naming `"*"` are indexed under the tenant. The policy row is deleted with the bucket.

### Bucket creation

Buckets are created and deleted over S3 with a service key, and only with a service key. A bucket policy grants actions on the bucket it is stored on; `CreateBucket` acts on a bucket that does not exist yet and `DeleteBucket` on the bucket whose policy would have to grant it, so no policy can carry either. A principal-bound key receives `AccessDenied` from both. `aws s3 mb` with a member's key therefore fails, and members create buckets through the console. This is a limit of the policy model, and the ADR accepts it: `CreateBucket` and `DeleteBucket` come off member keys in every region. The user-facing S3 documentation needs to state it.

A new bucket's policy travels on the create request, in the `x-bucket-policy` header: the policy document as JSON, base64-encoded. Ingot forwards the S3 request to Hilt in `/s3/bucket/create` as it does today, headers included, so the header needs nothing from Ingot. Hilt decodes it, validates it exactly as `PUT .../policy` would, creates the bucket, writes the policy, and issues every key of each named principal its delegations over the new bucket, all in one transaction. If the policy write fails, Hilt deletes the bucket. No bucket outlives a failed write of the policy its create request carried. The header MUST be among the request's `SignedHeaders`; Hilt refuses a create whose policy header is present and unsigned, since otherwise anything on the path could replace the document. A document that fails validation refuses the create with `InvalidBucketPolicy`, which Ingot renders as `InvalidArgument` (400). Ingot caps a request head at 8 KB, which leaves roughly 5 KB of JSON for the document.

The console sends the header on every bucket it creates. The document names the member who created the bucket and the organization's current Owners and Admins, each with the actions their role permits. A bucket created without the header, by a generic S3 client holding a service key, has no policy, and only service keys reach it until a policy is written through the management API.

### Principal-bound access keys

`POST /tenants/{tenantId}/access-keys`

```jsonc
{
  "name": "laptop",                            // unique per principal
  "principalId": "8f2c...",                    // makes the key principal-bound
  "expiresAt": "2026-12-31T00:00:00Z"          // optional
}
```

`principalId` travels in the body rather than the path so that one route, `POST /tenants/{tenantId}/access-keys`, creates both kinds of key and the console's existing calls keep working. The shape of the body decides the kind. A second route, `POST /tenants/{tenantId}/principals/{principalId}/access-keys`, is discussed under [alternatives](#separate-creation-routes-per-kind).

Hilt MUST generate and store the key exactly as the parent RFC specifies, MUST record the principal on the key row, and MUST store `NULL` for its permissions and its bucket list. 422 on a `principalId` that is not a live principal of the tenant, and 409 on a duplicate name for that principal. The response is the parent RFC's `CreatedAccessKey` with a `principal` field carrying the `principalId`, and no `permissions` or `buckets`.

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

A member whose policies grant `s3:PutObject` on `photos` and `s3:GetObject` on `backups`, holding two keys, gets six delegations over `photos` and one over `backups` on each key. The stored set for a key and a bucket is always `CommandsFor(effective(principal, bucket))`. Every write below keeps that inside its transaction, so a request that passes the action check always resolves a chain, and a key whose principal reaches no bucket holds no delegations. ucantone gives every delegation a random nonce, so a reissued set never carries a CID that was revoked. The shape is the parent RFC's: a service key scoped to named buckets holds the same rows, derived from its permission list.

A service key's name stays unique within the tenant, as today. A principal-bound key's name is unique within its principal, so two members may each hold a key named `laptop`.

`GET /tenants/{tenantId}/access-keys` and `GET /tenants/{tenantId}/access-keys/{accessKeyId}` return the `principal` field. A key without it is a service key and carries its own `permissions` and `buckets`. The key list does not carry each key's effective actions; the console reads them once per principal through the principal route.

`DELETE /tenants/{tenantId}/access-keys/{accessKeyId}` deletes either kind in the same sequence: a revocation for each of the key's delegations, published in one Swarf request, then the delegations, the row, and the vault entry. Ingot drops that key's store and no other.

Both kinds count toward `accessKeyCount`. Hilt enforces no key limit today, and none is added.

### Policy changes and delegation rotation

A policy `PUT` or `DELETE` changes the effective actions of the principals it names on that bucket, with `"*"` standing for every live principal of the tenant. For each principal named in the old or the new document, Hilt compares the effective set before and after. For every key bound to a principal whose set changed, Hilt rewrites the key's delegations over the bucket before acknowledging the write: it publishes a revocation for each delegation the key holds over the bucket, then issues and stores the set the new policy gives. To publish a revocation is to send Swarf a `/ucan/revoke` invocation for the delegation, signed by the tenant that issued it, and all revocations of one write travel in one Swarf request (see [Swarf](#swarf)). Delegations over other buckets are untouched.

A first grant of a bucket publishes nothing. The key held no delegation over it, and Ingot holds no cached set for a bucket it has not seen under that key, so the next request goes to Hilt and is authorized against the new policy. A widening of a bucket the key already reaches rotates as a narrowing does, because Ingot refuses an action outside a cached set on its own and the grant would otherwise wait for the cache to expire at midnight.

Ingot's store for a key holds the proof chains and the effective actions, and every chain contains one of the key's stored delegations. Ingot drops the whole store when any delegation in it is revoked, so revoking a key's delegations over one bucket clears its cache for every bucket, and the key's next request on any of them refills from Hilt. Revocation is exact at Hilt and Swarf. The refill is the cost at Ingot.

**Locking.** Hilt takes no row locks today; this RFC adds them on three paths. A policy write runs in one Postgres transaction that locks the policy row with `SELECT ... FOR UPDATE`, takes a transaction-scoped advisory lock on the bucket, and locks each affected principal's row with `SELECT ... FOR UPDATE` before it enumerates that principal's keys. Each key's rotation holds an advisory lock on that key's delegation set. Hilt publishes the revocations while those locks are held and commits afterwards. `/s3/request/authorize` reads the key row, the principal row, and the policy row with `SELECT ... FOR SHARE`, and reads the key's delegations before the policy. Key creation reads the principal row with `SELECT ... FOR SHARE` and materializes from the policies it can see. An authorize request that arrives between the publish and the commit therefore waits for the commit and is answered from the new policy with the new delegations, and a key created while a write is in flight is either seen by the rotation or created from the committed policy. Without the locks, an Ingot that had consumed the revocation could refill its cache from the old policy, holding delegations already revoked, and keep them until midnight.

Authorize requests for a bucket wait one Swarf round trip during a write to that bucket's policy, since the write's revocations travel in one request. A write holds its transaction open while it waits on Swarf, for at most 8 seconds, and a write that has not finished by then fails and rolls back. A reader or writer that cannot take its lock within 10 seconds fails as well, so a slow Swarf releases the locks before any reader times out. A management write that times out is 409 `ConcurrentChange`; an authorize that times out is `TemporarilyUnavailable`, which Ingot renders as `ServiceUnavailable` (503). The locks do not cover an authorize answered before the write took its locks whose response reaches Ingot after the revocation does. Swarf polls its store once a second before it emits a record, so that response would have to be in flight for longer than that.

**Failure.** If Swarf refuses or fails the publish, Hilt answers 500 and commits nothing. The old policy stays in force, and the principal never holds more than it did. If the publish succeeded and the commit then failed, the key's delegations are revoked at Swarf and still stored at Hilt: Ingot drops the key's store once and, on the key's next request, refills it from the old policy with those same delegations. The member's access is what the unchanged policy says. The console retries the write. The retry publishes the same revocations again, which Swarf records as the same invocations and ignores, so Ingot receives no second event and keeps the refilled store. The change reaches Ingot through a later write that revokes a different set, or when the cache expires at midnight. A console that stops retrying leaves the old policy in force and nothing half-applied.

**Recovery.** Hilt runs no reconciliation, because a failed write leaves nothing to reconcile: every write that publishes runs in one transaction, and the state Hilt stores is always the state it enforces. What a failed call leaves behind is the caller's unmet intent. The console owns completing it and SHOULD retry from a durable job rather than from the member's request, so that a member who gives up after one attempt does not leave a removal or a policy change unapplied.

Deleting a key revokes only that key's delegations. The principal's other keys keep their cached proofs.

The staleness bound for a change is the time from Hilt's acknowledgement to Ingot consuming the firehose record. It is bounded by firehose latency and is independent of the midnight horizon. The revocation clears Ingot's caches and nothing else. Sprue and Piri do not consult Swarf, so a per-request delegation Ingot already holds stays valid at them until it expires. Only Ingot can use it: Ingot is the delegation's audience and signs every invocation that carries it, and once the revocation has cleared its cache it asks Hilt, which answers from the new policy.

### Principal removal

`DELETE /tenants/{tenantId}/principals/{principalId}` locks the principal row with `SELECT ... FOR UPDATE` and, in one transaction and in order:

1. Publishes a revocation for every delegation of each of the principal's keys, in one Swarf request.
2. Removes the principal from every statement naming it. A statement left with no principal is deleted, and a policy left with no statement is deleted.
3. Deletes the principal's keys: rows, then vault entries.
4. Sets `deleted_at` on the principal row and commits, releasing the lock.

The row stays as a tombstone. A removed principal is absent from `GET` and the list, is refused with 422 as the `principalId` of a new key, and is skipped by evaluation should a statement still name it. An authorize request for one of its keys waits on the lock until the removal commits or rolls back, then is refused with `UnknownAccessKey` because the key is gone.

A failure at any step rolls the transaction back. The principal keeps every key and every statement it had, so a failed removal changes nothing at Hilt. The revocations already published stand and cost Ingot one refill from the unchanged policies. The console retries (see [recovery](#policy-changes-and-delegation-rotation)); the call answers 204 once the principal is removed, and 204 for an id that is removed or never existed.

A `PUT` for a removed principal's id clears `deleted_at`. The principal returns with no keys and named in no statement, so nothing of its earlier access survives, whether the id belongs to the same member re-invited or to a different member given the same id.

### Bucket deletion

`/s3/bucket/delete` deletes the bucket's delegations, its policy, and the bucket row. Hilt revokes every key's delegations over the bucket, service and principal-bound alike, in one Swarf request. Ingot removes the bucket from its local database on the same request, and its local authorization refuses any bucket not found there, so cached chains for a deleted bucket cannot be used.

### Tenant removal

`DELETE /tenants/{tenantId}` deletes a disabled tenant. It MUST publish a revocation for every delegation the tenant's keys hold before it deactivates the tenant's `did:plc`, because the tenant signs the revocations and Swarf verifies them against the tenant's DID. It then deletes the tenant's principals, policies, keys, buckets, and delegations. Ingot drops every store of the tenant's keys when the records arrive. Today the deletion publishes nothing, and a warm key keeps serving from Ingot's cache until the next UTC midnight.

## Hilt - UCAN API

The commands, their argument types, and their result types keep their shapes. The authorization steps gain a branch on how the permitted actions are derived, and bucket creation gains the policy header.

### `/s3/request/authorize`

For a request signed with a service key, the parent RFC's steps apply unchanged: the action must be in the key's `permissions`, and the bucket must be in its `buckets` or the key must have none.

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

The expiry rule is the parent RFC's. The response container carries the per-request delegations, which are not stored, with their proof chains. The key, principal, and policy rows are read with a share lock (see [policy changes](#policy-changes-and-delegation-rotation)). The path reads the access key's private key from the vault, as the parent RFC's does. A request that passes step 7 always resolves a chain, because the key's stored delegations over the bucket are exactly the commands its effective actions map to.

**Result.** The parent RFC's `AuthorizeOK`, with `tenant` carrying the tenant and `permissions` carrying the key's effective actions on the addressed bucket, sorted. For `ListBuckets` the effective actions are `["s3:ListAllMyBuckets"]`. For a service key `permissions` carries the key's own permission set, as the parent RFC specifies.

### `/s3/bucket/info`

For a principal-bound key the chain is the bucket root, bucket to tenant with `cmd: "/"`, followed by the key's stored delegations over that bucket, as for a service key scoped to named buckets. The key is a hop in the chain. The principal appears in no UCAN.

`InfoOK` keeps its shape; `permissions` carries the key's effective actions on the bucket. A key whose principal cannot reach the bucket receives `UnknownBucket`.

### `/s3/bucket/create` and `/s3/bucket/delete`

A service key reaches them by holding `s3:CreateBucket` or `s3:DeleteBucket` in its permissions. A principal-bound key receives `OperationNotPermitted` from both.

Create reads the `x-bucket-policy` header from the forwarded request when it is present, checks that the header is signed, validates the decoded document as `PUT .../policy` does, creates the bucket, and stores the policy. A failed policy write deletes the bucket (see [bucket creation](#bucket-creation)). An unsigned or invalid document refuses the create with `InvalidBucketPolicy`. Delete removes the policy with the bucket, as described above.

### `/s3/bucket/list`

Unchanged. The listing is the tenant's whole bucket set, as AWS lists names the caller cannot open. The console filters its own list against the principal's effective actions.

### Failures

`UnknownBucket` covers a bucket that does not exist, belongs to another tenant, or is out of the principal's reach. `OperationNotPermitted` covers an action outside a non-empty effective set and the two bucket operations no policy can grant. `UnknownAccessKey` covers a principal-bound key whose principal is removed. Ingot's existing mapping renders them as `NoSuchBucket` (404), `AccessDenied` (403), and `InvalidAccessKeyId` (403).

Two failures are new. `TemporarilyUnavailable` is an authorize that could not take its share lock within the lock timeout; Ingot renders it as `ServiceUnavailable` (503) and the client retries. `InvalidBucketPolicy` is a create whose policy header is unsigned or fails validation; Ingot renders it as `InvalidArgument` (400).

## Swarf

Swarf's service needs no change. A rotation, a key's deletion, a principal's removal, and a bucket's deletion publish `/ucan/revoke` for each delegation they invalidate, a UCAN command Swarf serves on its RPC endpoint, self-signed by the tenant that issued the delegation, with no witness path. That is the form of every revocation Hilt publishes. Swarf validates the delegation's signature and expiry, records the revocation, and emits a `revocation` event on `GET /revocations/:from`.

All revocations of one write go to Swarf in one request. A ucantone server executes every invocation addressed to it in a request container, and the client library exposes that as `ExecuteBatch`, which returns one receipt per invocation. Swarf's client gains a batch publish that builds one `/ucan/revoke` invocation per delegation and sends them together, and Hilt reads each receipt so that a refused revocation fails the write. A `"*"` edit therefore costs one round trip however many keys it rotates, and the request grows with the fan-out (see [open questions](#open-questions)).

## Ingot - S3 API

Ingot's request path is unchanged: authorize locally from cache when it can, and ask Hilt otherwise.

Ingot caches, per access key, what Hilt's authorize response carries: the delegations, the effective action set under each bucket Hilt named, the derived SigV4 verification key, and the tenant, all until the next UTC midnight plus clock skew. The fast path verifies the signature with the cached key, requires the action to be in the cached set for the bucket, and requires a chain to Ingot's agent for every Forge command the action maps to. Anything less goes to Hilt, whose answer refills the caches. The cached set is what makes `deny` enforceable. Several S3 actions map to the same commands: `s3:GetObject` and `s3:ListBucket` both need `/content/retrieve`, and `s3:PutObject` grants `/blob/remove` alongside `s3:DeleteObject`. A chain probe alone could not refuse one action of such a pair, or honor a policy that grants one without the other; the set does. A request for a bucket with no cached set goes to Hilt.

Operations that map to no Forge command are authorized at Hilt on every request: `ListBuckets`, `CreateBucket`, and `DeleteBucket`. No per-request delegation carries the key's expiry for them, so a cached set alone would outlive an expired key.

**The firehose consumer needs no change.** When Swarf reports a revoked CID, Ingot drops every per-key store holding a delegation with that CID, together with the key's cached signing key and tenant. Every delegation in a store is a chain member, and every chain for a principal-bound key contains one of the key's stored delegations, so a rotation always names something the store holds. A revocation that matches no store is a no-op.

Error mapping gains two rows: `TemporarilyUnavailable` renders as `ServiceUnavailable` (503) and `InvalidBucketPolicy` as `InvalidArgument` (400). `UnknownBucket` is already 404 and `OperationNotPermitted` already 403.

## Fil One console

The console holds two kinds of key per tenant.

Its service key is the tenant-wide credential it signs with today, created with no bucket list and holding `s3:CreateBucket` and `s3:DeleteBucket`. It signs every request that has no member actor and every request no policy can authorize: tenant setup, bucket creation with the policy header, bucket deletion, and background workers. For that traffic the console's own role check is the whole of the enforcement, as today.

For each member it holds one principal-bound key, created at the tenant's access-key route on the member's first request in the region. It signs everything the member does through the console: listing buckets and objects, reading, writing and deleting objects, and presigned URLs. Hilt and Ingot authorize that traffic against the member's policies, so bucket access is enforced at the storage system and the console's role checks stay in front of it.

Consequences:

- A member's key cannot create or delete a bucket, whether the console or the member created the key. The console signs those two operations with its service key after its own role check, so the tenant-wide credential stays in service.
- Hilt does not distinguish the console's key for a member from a key the member created themselves. Both are bound to the same principal, are authorized against the same policies, and are deleted with the principal.
- A presigned URL is authorized against the member's policies when it is redeemed, under the parent RFC's signature time bounds and the key's own expiry.
- A policy change reaches the member's console traffic the way it reaches every key of theirs: through the rotation of the key's delegations.

## Action vocabulary

The policy vocabulary is Hilt's S3 permission set without the three bucket-level actions, plus a wildcard:

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

Every existing key becomes a service key, because a key with no principal is exactly the parent RFC's key. Existing keys are kept: they are embedded in customers' workflows, they keep working unchanged, and customers rotate them when they choose.

**Once per Forge network.**

1. Deploy Ingot with the two new error mappings. Ingot already enforces the cached effective action set. Swarf's service needs no deployment; Hilt takes the client library with the batch publish.
2. Deploy Hilt. Its schema migration adds the principal and policy tables and the `principal` column on keys. Tenants, buckets, keys, vault entries, and delegations are untouched, and every key keeps authorizing through the parent RFC's path. Principal-bound keys are new, so every one is created with its delegations and nothing is backfilled.

**Once per organization.** The console creates a principal for each member with `PUT /tenants/{tenantId}/principals/{principalId}`, then writes a policy per bucket naming the organization's Owners and Admins. This changes nothing for members yet: the console still signs their traffic with its service key, and no policy is evaluated for a service key.

**Once per region.** When every organization in the region has its principals and policies, the console switches the region's access model from `scoped-keys` to `iam`. The access model is a per-region setting in the console; its policy routes, its policy fan-out, and its per-member keys are active only in a region whose model is `iam`. From then on the console sends `x-bucket-policy` on every bucket it creates.

**Once per member.** On the member's first request after the switch, the console creates a key bound to their principal and signs their traffic with it from then on. Until then the member's console traffic is signed with the service key, which reaches every bucket of the tenant, and no policy applies to it. A policy edit reaches a member only once they hold a principal-bound key.

No step takes a key out of service.

## Alternatives considered

### A principal invalidation record

Swarf could gain a `/principal/invalidate` command and a `principal` firehose event carrying a tenant and a `principalId`, and Ingot's consumer a branch that drops every store whose authorize result carried that pair. A policy change would publish one record per changed principal rather than one revocation per key, and Hilt would store nothing per key. Nothing in the record is a delegation, so Swarf could check no proof and would have to trust a configured publisher list. Ingot would index its stores by tenant and principal, and one key's deletion would clear every key the principal holds. Policy-derived delegations cost more rows and revocations per change and change neither Swarf nor Ingot's consumer, and what they revoke is a grant.

### A marker delegation

Each principal-bound key could hold one tenant-issued delegation naming a command no service handles, carried in every authorize response so that Ingot's store always contains it, and rotated on a policy change. It costs one row per key and makes every change one revocation per key. The revoked delegation grants nothing, so Ingot dropping the store on its revocation rests on an agreement between Hilt and Ingot recorded only here, and Sprue and Piri never see it. Policy-derived delegations revoke the grant that changed.

### Recorded per-request delegations revoked on narrowing

Hilt could persist each per-request delegation it issues and revoke the outstanding ones for a principal and bucket on a narrowing. The revocation would be exact to the bucket, and downstream services would never see a wider grant than the policy. It adds a table that churns once per key per bucket per day, and the stored delegations already name the bucket and command, so it adds no precision. Revoking them would also cut in-flight work at Sprue and Piri.

### A did:key principal in the chain

Each principal could be an ed25519 key Hilt stores, holding one delegation from the tenant that sits in every chain for the principal's keys, with the keys delegated from the principal. A policy change would then rotate one delegation per principal rather than one set per key, and a did:key principal could sign its own invocations one day. It costs a vault entry and a stored delegation per member, a hop in every proof chain, and a chain that names the tenant's whole authority rather than the buckets the policies grant. Delegating to the key keeps principals free of key material and revokes exactly the grant a policy change removes.

### Policies evaluated at Ingot

Ingot embeds versitygw, whose policy engine is disabled today by returning the admin role for every authenticated request. Storing policies at Ingot and enabling that engine would evaluate them at the edge with no Hilt round trip. It puts the rule outside the system that owns it, requires policy persistence and replication at every Ingot instance, and leaves Hilt unable to issue delegations that match the decision.

### Separate creation routes per kind

`POST /tenants/{tenantId}/service-credentials` and `POST /tenants/{tenantId}/principals/{principalId}/access-keys` would make a key's kind a property of the route it was created at, so each body would hold only the fields its kind uses and there would be no field combination to reject. A service credential would reach every bucket of the tenant rather than carrying a permission list, and the scoped-key path would leave Hilt entirely. The cost is a second set of create, list, and delete routes, a second vault path, a change to the management API contract the console already calls, and a migration that revokes and deletes every existing key before any member can be served. One route whose body takes one of two shapes leaves the console's existing calls working and keeps the parent RFC's authorization path in place for traffic that has no member actor.

### An access model flag per tenant

A tenant could carry `scoped-keys` or `iam`, and Hilt would apply one model to every key the tenant holds. Each request would then read one model from the tenant row rather than branching on the key. A tenant cannot hold a worker key and a member key at once under that rule, and the console holds both from the moment it creates its first principal.

### A permission list on a principal-bound key

A principal-bound key could carry its own permission list, as an AWS session policy does, and be authorized against the intersection of that list and the policy result. A member could then hold a read-only key and a full key on the same buckets, and the console would not have to model the difference as two principals. It puts a second document in the path of every denial, so a 403 has two possible sources and Ingot's cached set is no longer the principal's effective set. Rejecting the fields with 422 keeps one answer to what a member may do.

### A default policy template per tenant

Hilt could hold one policy document per tenant and copy it onto every bucket at create, so the create request would carry nothing. The console would have to keep the template in step with the organization's roster, which is a second copy of the same statements it writes on every bucket, and the template could not name the member who created the bucket. The header carries the same document once, on the request that needs it.

### The policy written after the create

The console could create the bucket over S3 and then write its policy through the management API, answering the member after both calls. A console failure between the two calls, or after the retry of the second, leaves a bucket that appears in the listing and that no member can open, with nothing to trigger another write. Carrying the policy on the create makes the two one.

### Filtering the bucket listing

Scoping `/s3/bucket/list` to the buckets a principal reaches was proposed for access keys in Hilt PR #48 and closed as a divergence from AWS, where a listing shows names the caller cannot open. The console filters its own list, and Ingot's behavior is unchanged.

## Open questions

1. What latency target does `GET /principals/{principalId}/access` carry? The console resolves it per request for its bucket list and activity feed. It is an index lookup at Hilt; the number is unmeasured.
2. A policy edit naming `"*"` rewrites the delegations of every key of every live principal of the tenant over that bucket: rows and tenant-key signatures inside the write, and one Swarf request whose size grows with the fan-out. A key's rows number its buckets times the commands its actions map to, up to seven per bucket. There is no figure for principals per tenant or keys per principal on Forge. If the product is large, that write amplification is the cost to watch.
3. Limits on statements per policy and principals per statement. None are specified here beyond the 8 KB request head on a create.

## Evaluation criteria

- The time from a change's acknowledgement to Ingot's first refusal, measured against a warm key. This is the staleness bound the ADR asks Hilt to publish.
- Authorization latency for a service key is unchanged from today's key.
- No existing key stops working at any point in the rollout.
- The console's in-memory IAM fake and Hilt pass the same contract tests.
- A policy `PUT` costs one Swarf revocation and one stored delegation per key of each principal whose effective actions changed.
- Swarf's service and Ingot's firehose consumer need no changes.

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

Changes to the parent RFC's schema. Types and constraints follow the existing migrations.

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
    ADD COLUMN principal TEXT,                            -- principalId
    ADD FOREIGN KEY (tenant_id, principal) REFERENCES principal(tenant_id, external_id) ON DELETE RESTRICT,
    ADD CONSTRAINT access_key_principal_unscoped
        CHECK ((principal IS NULL AND permissions IS NOT NULL)
            OR (principal IS NOT NULL AND permissions IS NULL AND buckets IS NULL));

-- A service key's name stays unique within the tenant; a principal-bound key's
-- name is unique within its principal. access_key_tenant_id_name_key is the name
-- Postgres generated for the parent RFC's UNIQUE (tenant_id, name).
ALTER TABLE access_key DROP CONSTRAINT access_key_tenant_id_name_key;
CREATE UNIQUE INDEX access_key_service_name_idx ON access_key (tenant_id, name)
    WHERE principal IS NULL;
CREATE UNIQUE INDEX access_key_principal_name_idx ON access_key (tenant_id, principal, name)
    WHERE principal IS NOT NULL;

CREATE TABLE bucket_policy (
    bucket_id  TEXT        PRIMARY KEY REFERENCES bucket(id) ON DELETE CASCADE,
    document   JSONB       NOT NULL,
    etag       TEXT        NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Which principals a policy names; "*" statements are indexed as NULL principal.
CREATE TABLE bucket_policy_principal (
    bucket_id  TEXT NOT NULL REFERENCES bucket(id) ON DELETE CASCADE,
    tenant_id  TEXT NOT NULL,
    principal  TEXT,
    FOREIGN KEY (tenant_id, principal) REFERENCES principal(tenant_id, external_id) ON DELETE CASCADE
);
-- Postgres treats NULLs as distinct in a UNIQUE constraint, so the wildcard row gets its own index.
CREATE UNIQUE INDEX bucket_policy_principal_named_idx ON bucket_policy_principal (bucket_id, principal) WHERE principal IS NOT NULL;
CREATE UNIQUE INDEX bucket_policy_principal_wildcard_idx ON bucket_policy_principal (bucket_id) WHERE principal IS NULL;
CREATE INDEX bucket_policy_principal_idx ON bucket_policy_principal (tenant_id, principal, bucket_id);
```

The `delegation` table keeps its schema; a principal-bound key's rows in it are its policy-derived grants, one per bucket and Forge command. The vault keeps the parent RFC's layout: every key at `/tenant/{tenantDID}/access-key/{accessKeyDID}`, and no entry for a principal.

### Example flows

#### Migrate an existing tenant

1. The tenant's keys keep working as service keys. Fil One changes nothing about them and keeps using the service key it holds.
2. Fil One calls `PUT /tenants/{tenantId}/principals/{principalId}` for every member.
3. Hilt stores each principal.
4. Fil One writes each bucket's policy.
5. On a member's next request, Fil One calls `POST /tenants/{tenantId}/access-keys` with that member's `principalId` and signs their traffic with the returned key.

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
3. Swarf's firehose delivers the revocations. Ingot drops the store of every key that held one of them.
4. The member's next request on any bucket reaches Hilt. On the removed bucket it receives `UnknownBucket`, which Ingot renders as `NoSuchBucket`.

#### Remove a member

1. Fil One calls `DELETE /tenants/{t}/principals/{principalId}`.
2. Hilt locks the principal row, publishes a revocation for every delegation of the member's keys in one Swarf request, strips the principal from every statement, deletes its keys, sets `deleted_at`, and commits.
3. Ingot drops each key's store on its revocation. The keys no longer resolve at Hilt.
4. If the member is re-invited, Fil One calls `PUT` for the same `principalId`. The tombstone clears and the principal starts with no keys and no statements.
