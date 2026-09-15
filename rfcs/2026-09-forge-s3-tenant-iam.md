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

On Forge the ADR settles three things: each member is a principal at the storage system, a bucket's own policy is the only thing that grants access, and a key belongs to a principal and carries whatever the policies give that principal, evaluated per request.

## Goals

1. The storage system computes what a principal may do on a bucket from that bucket's policy alone, `Allow` minus `Deny`, with an explicit `Deny` winning.
2. A policy edit changes what every key bound to a named principal may do, with no key reissued.
3. Every change to a principal's access is published to Swarf as a revocation before Hilt acknowledges it, and Ingot's existing firehose consumer clears the gateway's warm caches within firehose latency.
4. The console signs a member's traffic, presigned URLs included, with a key bound to that member's principal, and keeps a service key for traffic that has no member actor.
5. One policy model governs every principal. Existing keys keep working, and a tenant moves to principals with no key reissued and no window in which the network refuses requests.

## Concepts

### Roles

The parent RFC's roles apply. Swarf is added:

| Name  | Description                                                                                                |
| ----- | ---------------------------------------------------------------------------------------------------------- |
| Swarf | The Forge revocation service. Hilt publishes revocations to it; Ingot consumes its firehose. |

### Terms

- **Principal.** A member as Hilt knows them: `(tenant, userId)` and nothing more. A principal carries no permissions, no role, and no key material.
- **Bucket policy.** A bucket's own document of statements. Each statement has an effect, a list of principals, and a list of S3 actions. The policy is created with its bucket and destroyed with it.
- **Principal-bound key.** An S3 access key that authenticates a request and identifies a principal. It holds no authority of its own.
- **Service key.** An access key with no principal: the parent RFC's key, carrying the permissions and buckets it was created with. It signs traffic that has no member actor, such as tenant setup, bucket create and delete, and background workers. A tenant may hold several. Everything a member does in the console, from listing buckets and objects to deleting objects and redeeming presigned URLs, is signed with a key bound to that member's principal.
- **Marker.** The one delegation a principal-bound key holds: issued by the tenant to the key, naming a command nothing executes, and returned with every authorize response so the gateway's store for the key contains it. Revoking it is how a policy change reaches the gateway.

## Hilt - Tenant API

`POST /tenants/{tenantId}/access-keys` gains one optional field, `principalId`, and that field decides which kind of key it creates:

| `principalId` | Kind                | Authority                                                                   |
| ------------- | ------------------- | --------------------------------------------------------------------------- |
| absent        | Service key         | The `permissions` and `buckets` in the request, as the parent RFC specifies. |
| present       | Principal-bound key | Whatever the bucket policies give the principal, evaluated per request.      |

Hilt MUST reject with 422 a request carrying `principalId` together with a non-empty `permissions` or `buckets`. A caller that asks for a narrowed key would get one whose reach the policies decide, and neither the create response nor a later `GET` would show the difference. The management API requires `permissions` on every create today. That requirement applies only to a create without `principalId`.

`GET`, `DELETE`, and the tenant's key list serve both kinds.

### Tenant creation

`PUT /tenants/{tenantId}` is unchanged. The console creates the tenant and then calls `POST /tenants/{tenantId}/access-keys` with no `principalId` for the credential it provisions buckets with, as it does today.

A Forge network serves one region, and a tenant is unique to its region, so the external tenant id stays the organization's UUID as the contract defines it. Hilt's one-provider-per-tenant model holds unchanged.

### Service keys

A service key is the parent RFC's access key, unchanged: an ed25519 `did:key` at the vault path `/tenant/{tenant}/access/{accessKeyId}`. It holds one delegation from the tenant per bucket and Forge command mapped from its permissions, or a powerline delegation with an undefined subject when it was created with no bucket list.

A service key:

- Reaches the buckets in its list, or every bucket of the tenant when it has none. It is named in no statement and no policy is evaluated for it.
- MAY hold `s3:CreateBucket` and `s3:DeleteBucket`, which no policy grants (see [action vocabulary](#action-vocabulary)).
- Appears in `GET /tenants/{tenantId}/access-keys`, counts toward `accessKeyCount`, and is deleted through the access-key delete route, as today.
- Keeps the parent RFC's optional `expiresAt`. Rotation is minting a new key, moving the caller to it, and deleting the old one.

Its bucket list is fixed at creation, so a service key scoped to named buckets cannot reach a bucket created afterwards. Tenant setup, bucket create, and bucket delete therefore use a key with no bucket list.

Deleting a service key MUST publish revocations for its delegations before deleting the vault entry and the row, as the parent RFC's key delete does.

### Principals

Routes, all under `/tenants/{tenantId}/principals` and authenticated with the partner key:

| Method | Path                                 | Purpose                                                 |
| ------ | ------------------------------------ | ------------------------------------------------------- |
| PUT    | `/principals/{userId}`               | Create the principal; 200 if it exists. Idempotent.     |
| GET    | `/principals`                        | List principals.                                        |
| GET    | `/principals/{userId}`               | Principal detail.                                       |
| DELETE | `/principals/{userId}`               | Remove the principal (see [removal](#principal-removal)). 204 if already gone. |
| GET    | `/principals/{userId}/policies`      | Every bucket policy with a statement naming the principal or `*`. |
| GET    | `/principals/{userId}/access`        | The principal's effective actions per bucket.           |
| GET    | `/principals/{userId}/access-keys`   | List the principal's keys.                              |

A key bound to a principal is created at the tenant's access-key route with `principalId` set (see [principal-bound access keys](#principal-bound-access-keys)).

`userId` is an opaque string supplied by the caller, unique within the tenant.

A principal is represented as:

```jsonc
{
  "userId": "8f2c...",
  "createdAt": "2026-09-09T10:00:00Z"
}
```

Creating a principal stores the row and nothing else. A principal has no DID, no vault entry, and no delegation. It appears in no UCAN: Hilt evaluates the bucket policy for the principal and signs the per-request delegation with the tenant key, so the proof chain runs from the bucket to the tenant to the gateway.

The principal's effective actions are read through `GET /principals/{userId}/access`:

```jsonc
{
  "buckets": [
    { "name": "photos", "actions": ["s3:GetObject", "s3:ListBucket"] },
    { "name": "backups", "actions": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"] }
  ]
}
```

Buckets with an empty effective set are omitted. The read is computed from Hilt's store on every call and is consistent with Hilt's last write. It makes no network call.

### Bucket policies

A bucket has at most one policy, addressed by the bucket's name:

| Method | Path                                          | Purpose                                                   |
| ------ | --------------------------------------------- | --------------------------------------------------------- |
| GET    | `/tenants/{tenantId}/buckets/{bucketName}/policy` | Read the policy. Returns an `ETag`.                    |
| PUT    | `/tenants/{tenantId}/buckets/{bucketName}/policy` | Create or replace. Carries `If-Match` or `If-None-Match: *`. |
| DELETE | `/tenants/{tenantId}/buckets/{bucketName}/policy` | Delete. Carries `If-Match`.                            |

The document:

```jsonc
{
  "statements": [
    {
      "effect": "Allow",                       // "Allow" | "Deny"
      "principals": ["8f2c...", "a91e..."],    // userIds, or ["*"] for every principal of the tenant
      "actions": ["s3:GetObject", "s3:ListBucket"]
    }
  ]
}
```

Hilt MUST reject with 422:

- an action outside the [policy vocabulary](#action-vocabulary); in particular `s3:CreateBucket`, `s3:DeleteBucket`, and `s3:ListAllMyBuckets`,
- a `userId` that is not a principal of the tenant,
- an empty `statements` list (the caller deletes the policy instead).

A bucket that does not exist or belongs to another tenant is 404 on all three routes.

Hilt MUST accept a `Deny` naming `*`. The management API carries no actor; the console decides whether to write one.

**Evaluation.** The effective actions of principal `p` on bucket `b` are

```
effective(p, b) = union(actions of Allow statements naming p or *)
                \ union(actions of Deny statements naming p or *)
```

A bucket with no policy has an empty effective set for every principal. `s3:ListAllMyBuckets` is outside the per-bucket set: every principal holds it, and `ListBuckets` is authorized without consulting any policy (see [action vocabulary](#action-vocabulary)). A service key is not a principal and is not evaluated against policies.

**Compare-and-set.** `GET` returns a strong `ETag` computed over Hilt's canonical encoding of the stored document. `PUT` and `DELETE` MUST carry `If-Match` with that value; a mismatch is 412 and nothing is written. A `PUT` creating the first policy carries `If-None-Match: *` and is 412 if a policy already exists. Callers treat the ETag as opaque. A successful `PUT` returns the new `ETag`.

**Storage.** Hilt stores the document with its bucket and maintains an index from principal to the buckets whose statements name it, so the two principal reads above cost an index lookup. Statements naming `*` are indexed under the tenant. The policy row is deleted with the bucket.

### Buckets without a policy

A bucket created over S3 with a service key has no policy, and no principal reaches it until one is written. The console creates a bucket over S3 and writes its policy with `If-None-Match: *` before answering the member, so a member never learns of a bucket they cannot open. A console failure between the two calls leaves a bucket only service keys reach. The console writes the policy on its retry, since a repeated `CreateBucket` for a name the tenant owns is reported as already owned.

### Principal-bound access keys

`POST /tenants/{tenantId}/access-keys`

```jsonc
{
  "name": "laptop",                            // unique per principal
  "principalId": "8f2c...",                    // makes the key principal-bound
  "expiresAt": "2026-12-31T00:00:00Z"          // optional
}
```

Hilt MUST generate and store the key exactly as the parent RFC specifies, MUST record the principal on the key row, and MUST store an empty permission set and no bucket list. 422 on a `principalId` that is not a principal of the tenant, and 409 on a duplicate name for that principal. The response is the parent RFC's `CreatedAccessKey` with a `principal` field carrying the `userId` in place of `permissions` and `buckets`.

Hilt MUST also issue the key one delegation, its marker, and store it in the `delegation` table with the tenant's other grants:

```jsonc
{
  "iss": "did:plc:tenant",
  "aud": "did:key:accessKey",
  "sub": "did:plc:tenant",
  "cmd": "/s3/key/marker",
  // exp: the key's expiresAt, or none
}
```

The marker grants nothing. No service handles its command and it is a hop in no proof chain, so Sprue and Piri never see it. It exists so that the key has a delegation with a CID the gateway holds and Hilt can revoke. A fresh nonce gives each marker its own CID.

A service key's name stays unique within the tenant, as today. A principal-bound key's name is unique within its principal, so two members may each hold a key named `laptop`.

The console holds one such key per member for its own traffic, minted on the member's first request in the region, and signs that member's bucket listing, object listing, object reads and deletes, and presigned URLs with it. Hilt does not distinguish it from a key the member created: it is bound to the same principal, authorized against the same policies, and deleted with the principal. A presigned URL signed with it is authorized against the member's policies when it is redeemed, with the parent RFC's signature time bounds and the key's own expiry applying as for any request.

`GET /tenants/{tenantId}/access-keys` and `GET /tenants/{tenantId}/access-keys/{accessKeyId}` return the `principal` field. A key without it is a service key and carries its own `permissions` and `buckets`. The key list does not carry each key's access; the console reads a principal's access once through the principal route instead of once per key.

`DELETE /tenants/{tenantId}/access-keys/{accessKeyId}` deletes either kind in the parent RFC's sequence: a revocation for each of the key's delegations, then the vault entry, the delegations, and the row. For a principal-bound key the only delegation is the marker, so the gateway drops that key's store and no other.

Both kinds count toward `accessKeyCount`. Hilt enforces no key limit today, and none is added.

### Policy changes and marker rotation

A policy `PUT` or `DELETE` changes the effective set of the principals it names, `*` expanding to every principal of the tenant. For each principal named in the old or new document, Hilt MUST determine whether its effective set on the bucket changed. Before acknowledging the write, it MUST rotate the marker of every key bound to a changed principal: publish a revocation for the current marker, signed by the tenant, then issue and store a new one.

The gateway caches proof chains and effective action sets per access key until the next UTC midnight, and it drops a key's whole store when any delegation in it is revoked. Nothing in the chains identifies the principal, and the per-request delegations are not stored, so neither gives Hilt a CID to revoke. The marker gives it one. Every authorize response for the key carries it, so every store the gateway holds for the key contains it, and one revocation clears them all. A widening needs the rotation as much as a narrowing: the gateway refuses an action outside a cached set on its own, so a grant would otherwise wait for the cache to expire.

Hilt MUST publish inside the transaction that commits the change, after locking the policy row and the rows of the keys whose markers it rotates, and before committing. `/s3/request/authorize` reads the bucket's policy row, the key row, and the principal row with `SELECT ... FOR SHARE`. An authorize request that arrives between the publish and the commit therefore waits for the commit and is answered from the new policy with the new marker. Without the lock, a gateway that had consumed the revocation could refill its cache from the old policy, holding a marker already revoked, and keep it until midnight. Authorize requests for a bucket wait one Swarf round trip per rotated key during a write to that bucket's policy; the publishes MAY run concurrently. The lock does not cover an authorize answered before the lock was taken whose response reaches the gateway after the revocation does. Swarf polls its store once a second before it emits a record, so that response would have to be in flight for longer than that.

If any publish fails, Hilt MUST return 500 and MUST NOT commit the change. A retry rotates again. A failed change leaves the principal with its old access and never with more. A revocation published for a rotation that did not commit leaves the old marker in service and revoked at Swarf, and a gateway may refill a store with it before the retry. The retry revokes that marker again, and Swarf records the repeat as a new event because every revocation Hilt publishes carries a fresh nonce (see [Swarf](#swarf)), so the gateway clears the store once more.

Deleting a key revokes only that key's marker. The principal's other keys keep their cached proofs.

The staleness bound for a change is the time from Hilt's acknowledgement to the gateway consuming the firehose record. It is bounded by firehose latency and is independent of the midnight horizon. The revocation clears gateway caches and nothing else: a per-request delegation the gateway already holds in flight stays valid at Sprue and Piri until its expiry, because those services check revocation by delegation CID and the marker is in none of their chains.

### Principal removal

`DELETE /tenants/{tenantId}/principals/{userId}` MUST lock the principal row for the whole removal and, in order:

1. Publish a revocation for the marker of each of the principal's keys.
2. Remove the principal from every statement naming it, deleting statements left with no principal and policies left with no statement.
3. Delete the principal's keys: markers, vault entries, and rows.
4. Delete the principal row and commit, releasing the lock.

Steps 2 and 3 are idempotent. A failure before step 4 rolls back the principal row and leaves the principal with narrower access; the call is retried and answers 204 once the principal is gone. An authorize request for the principal waits on the lock until the removal commits or rolls back.

A removed member who is later re-invited gets a new principal row. No statement survives to restore their old access.

### Bucket deletion

`/s3/bucket/delete` deletes the bucket's policy with the bucket. Hilt revokes a service key's delegations over the bucket and publishes nothing for principal-bound keys: a marker names no bucket, and per-request delegations are not stored. The gateway removes the bucket from its registry on the same request, and its local authorization refuses any bucket it does not know, so cached chains for the deleted bucket cannot be used. This holds because a Forge network runs one gateway: Ingot's registry and caches are local to the instance, and a second gateway serving the same network would keep serving the bucket from its own caches until they expired. A network with several gateways needs a bucket invalidation record, which this RFC does not define (see [open questions](#open-questions)).

Deleting a tenant cascades over principals, policies, and access keys. As today, it publishes no revocations and deactivates the tenant's `did:plc`.

## Hilt - UCAN API

The commands, their argument types, and their result types keep their shapes. The authorization steps gain a branch on whether the key has a principal, and the response container for a principal-bound key gains the key's marker.

### `/s3/request/authorize`

For a request signed with a service key, the parent RFC's steps apply unchanged: the action must be in the key's `permissions`, and the bucket must be in its `buckets` or the key must have none.

For a request signed with a principal-bound key, Hilt performs the parent RFC's steps through signature verification, tenant status, issuer, and region, then:

1. Derive the S3 operation and its action from method, path, and query (see [action vocabulary](#action-vocabulary)).
2. Load the key's principal.
3. If the operation is `ListBuckets`, authorize it: every principal holds `s3:ListAllMyBuckets`.
4. If the operation is `CreateBucket` or `DeleteBucket`, refuse with `OperationNotPermitted`. No policy grants them.
5. Resolve the bucket by name. If it does not exist, or belongs to another tenant, refuse with `UnknownBucket`.
6. Compute `effective(principal, bucket)`. If it is empty, refuse with `UnknownBucket`.
7. If the action is not in the effective set, refuse with `OperationNotPermitted`.
8. Issue one per-request delegation per Forge command mapped from the action, signed with the tenant key:

```jsonc
{
  "iss": "did:plc:tenant",
  "aud": "did:web:gateway",     // the invocation issuer
  "sub": "did:key:bucket",
  "cmd": "/content/retrieve",
  // exp: next UTC midnight plus clock skew, capped at the key's expiry
}
```

The expiry rule is the parent RFC's. The response container carries the per-request delegations, which are not stored, and the key's marker. Hilt already loads the tenant key to provision buckets, so signing with it adds no vault read the request path does not have today.

**Result.** The parent RFC's `AuthorizeOK`, with `permissions` carrying the key's effective actions on the addressed bucket. For `ListBuckets` the effective actions are `["s3:ListAllMyBuckets"]`. For a service key `permissions` carries the key's own permission set, as the parent RFC specifies.

### `/s3/bucket/info`

For a principal-bound key the chain is the bucket root alone, bucket to tenant with `cmd: "/"`, and the container carries the key's marker with it. Neither the principal nor the key is a hop.

`InfoOK` keeps its shape; `permissions` carries the key's effective actions on the bucket. A key whose principal cannot reach the bucket receives `UnknownBucket`.

### `/s3/bucket/create` and `/s3/bucket/delete`

Unchanged for a service key. It reaches them by holding `s3:CreateBucket` or `s3:DeleteBucket` in its permissions. A principal-bound key receives `OperationNotPermitted` from both. Bucket deletion additionally removes the policy, as described above.

### `/s3/bucket/list`

Unchanged. The listing is the tenant's whole bucket set, as AWS lists names the caller cannot open. The console filters its own list against the principal's access.

### Failures

No new failure names. `UnknownBucket` covers a bucket that does not exist, belongs to another tenant, or is out of the principal's reach. `OperationNotPermitted` covers an action outside a non-empty effective set and the two bucket operations no policy can grant. Ingot's existing mapping renders them as `NoSuchBucket` (404) and `AccessDenied` (403).

## Swarf

Swarf's service needs no change. A marker rotation and a principal-bound key's deletion publish `/ucan/revoke` for the marker, self-signed by the tenant that issued it, with no witness path. That is the form of every revocation Hilt publishes. Swarf validates the marker's signature and expiry, records the revocation, and emits a `revocation` event on `GET /revocations/:from`. `GET /revocation/:cid` answers for a marker as for any delegation. Only gateways ever hold one, so no other verifier asks.

Swarf's client library builds each revocation invocation without a nonce, so a repeat for the same delegation has the same CID and Swarf's insert ignores it. Marker rotation relies on a repeat recording a new event (see [policy changes](#policy-changes-and-marker-rotation)), so `Publish` gains an option to set a nonce, and Hilt sets a fresh one on every revocation it publishes.

## Ingot - S3 API

Ingot's request path is unchanged: authorize locally from cache when it can, and ask Hilt otherwise.

**Ingot MUST enforce the effective action set.** A proof-chain probe alone cannot. Several S3 actions map to the same commands: `s3:GetObject` and `s3:ListBucket` both need `/content/retrieve`, and `s3:PutObject` grants `/blob/remove` alongside `s3:DeleteObject`. A probe cannot enforce a `Deny` on one action of such a pair, or a policy that grants one without the other. Ingot MUST cache the `permissions` value per access key and bucket, in the same per-key store as the chains so a revocation drops both, and MUST refuse a fast-path request whose action is not in the cached set. A request for a bucket with no cached set goes to Hilt. The two bucket-configuration reads map to no Forge command and are authorized at Hilt on every request, as every bucket-level operation is today. No per-request delegation carries the key's expiry for them, so a cached set alone would outlive an expired key. Ingot serves the value from its registry once Hilt answers.

**The firehose consumer needs no change.** It drops every per-key store containing a revoked CID, and clears the key's derived signing key and tenant with it. Ingot adds every delegation an authorize response carries to the key's store, chain member or not, so each store for a principal-bound key contains the key's marker and a revocation of the marker drops the store. Ingot MUST keep adding every delegation the container carries. A store holding only chain members would hold nothing a rotation could name. The handler is idempotent and a revocation matching nothing is a no-op. A service key's store is cleared by the same path it uses today.

Error mapping needs no change: `UnknownBucket` is already 404 and `OperationNotPermitted` already 403.

## Action vocabulary

The policy vocabulary is Hilt's S3 permission set without the three bucket-level actions, plus the two bucket-configuration reads the console already offers on a key:

| S3 action                          | Forge commands                                                                | Note                              |
| ---------------------------------- | ----------------------------------------------------------------------------- | --------------------------------- |
| `s3:GetObject`                     | `/content/retrieve`                                                           |                                   |
| `s3:GetObjectVersion`              | `/content/retrieve`                                                           |                                   |
| `s3:GetObjectRetention`            | `/content/retrieve`                                                           |                                   |
| `s3:GetObjectLegalHold`            | `/content/retrieve`                                                           |                                   |
| `s3:PutObject`                     | `/blob/add`, `/index/add`, `/upload/add`, `/content/retrieve`, `/blob/abort`, `/blob/remove` |                    |
| `s3:PutObjectRetention`            | as `s3:PutObject`                                                             |                                   |
| `s3:PutObjectLegalHold`            | as `s3:PutObject`                                                             |                                   |
| `s3:DeleteObject`                  | `/blob/remove`, `/upload/remove`                                              |                                   |
| `s3:DeleteObjectVersion`           | `/blob/remove`, `/upload/remove`                                              |                                   |
| `s3:ListBucket`                    | `/content/retrieve`                                                           |                                   |
| `s3:ListBucketVersions`            | `/content/retrieve`                                                           |                                   |
| `s3:ListBucketMultipartUploads`    | `/content/retrieve`                                                           |                                   |
| `s3:ListMultipartUploadParts`      | `/content/retrieve`                                                           |                                   |
| `s3:AbortMultipartUpload`          | `/blob/abort`, `/blob/remove`                                                 |                                   |
| `s3:GetBucketVersioning`           | none                                                                          | new; served by Ingot from its registry |
| `s3:GetBucketObjectLockConfiguration` | none                                                                       | new; served by Ingot from its registry |

Excluded from policies: `s3:CreateBucket` and `s3:DeleteBucket`, because a principal holding them acts outside the policy that granted it, and `s3:ListAllMyBuckets`, which every principal holds. A service key still holds the two bucket actions through its own permissions, which is how the console creates and deletes buckets.

The two bucket-configuration reads are new to Hilt. Its operation classifier MUST recognize `GET /{bucket}?versioning` as `GetBucketVersioning` and `GET /{bucket}?object-lock` as `GetBucketObjectLockConfiguration`; today both classify as `ListBucket`. The management API's action enum, now used by policy statements, gains both, along with `s3:AbortMultipartUpload` and `s3:ListMultipartUploadParts`, which Hilt already accepts.

Retention and legal-hold writes pass through the API like any other action. The rule that only an Owner may grant them is the console's, and Hilt does not know it.

## Migration

Every existing key becomes a service key, because a key with no principal is exactly the parent RFC's key. Per Forge network:

1. Deploy Ingot with the effective-action check on every gateway serving the network. Swarf needs no deployment.
2. Deploy Hilt. Its schema migration adds the `principal` column and its indexes. Tenants, buckets, keys, vault entries, and delegations are untouched, and every key keeps authorizing through the parent RFC's path. Principal-bound keys are new, so every one is created with its marker and nothing is backfilled.
3. The console creates a principal for each existing member with `PUT /tenants/{tenantId}/principals/{userId}`, then writes a policy per bucket naming the org's Owners and Admins.
4. The console flips the region's registry entry to `iam` and starts signing each member's traffic with a key bound to their principal. The tenant credential it already holds keeps signing traffic with no member actor.
5. The console deletes the keys the tenant no longer uses, through the existing delete route.

No step takes the network's keys out of service. Until the console moves a member to a principal-bound key, that member's access is whatever their old key carries and no policy applies to it. A policy edit reaches a member only after step 4 has run for them.

## Alternatives considered

### A principal invalidation record

Swarf could gain a `/principal/invalidate` command and a `principal` firehose event carrying a tenant and a `userId`, and Ingot's consumer a branch that drops every store whose authorize result carried that pair. A policy change would publish one record per changed principal rather than one revocation per key, and Hilt would store nothing per key. Nothing in the record is a delegation, so Swarf could check no proof and would have to trust a configured publisher list. Ingot would index its stores by tenant and principal, and one key's deletion would clear every key the principal holds. The marker costs a few more records per change and changes neither Swarf nor Ingot's consumer.

### One marker per principal

A single tenant-issued marker per principal, carried in every response for the principal's keys, would clear all of them with one revocation and cost one row per principal. It needs a new issue-and-rotate path in Hilt and a stored delegation bound to no key, and it clears every key of the principal when one is deleted. Per key, the same rows and revocations fall out of the paths a service key already uses, at the keys-per-principal factor in publishes.

### Allow statements materialized as stored delegations

Hilt could write one tenant-to-principal delegation per bucket and Forge command that an Allow grants, and revoke exactly those on a narrowing. Stored authority would then equal granted authority, and revocation would name the precise delegation. It costs principals times buckets times commands rows, turns an Owner promotion into one delegation set per bucket, and still cannot express `Deny`, which would be enforced at Hilt and Ingot regardless. One marker per key produces the same cache effect from one row per key.

### Recorded per-request delegations revoked on narrowing

Hilt could persist each per-request delegation it issues and revoke the outstanding ones for a principal and bucket on a narrowing. The revocation would be exact to the bucket, and downstream services would never see a wider grant than the policy. It adds a table that churns once per key per bucket per day, and the gateway drops the whole per-key store on any revoked CID anyway, so the precision changes nothing at the cache. Revoking them would also cut in-flight work at Sprue and Piri, on a widening as on a narrowing.

### A did:key principal with a standing delegation

Each principal could be an ed25519 key Hilt stores, holding one powerline delegation from the tenant that sits in every chain for the principal's keys. A narrowing would then rotate that delegation and revoke the old one by CID through Swarf's existing command, so Swarf and Ingot's consumer would change nothing and every downstream verifier would honor the revocation. A did:key principal could also sign its own invocations one day. It costs a vault entry and a stored delegation per member, a hop in every proof chain, and a rotation on every narrowing. The marker keeps the rotation and drops the vault entry and the hop. It sits in no chain, so only gateways act on its revocation.

### The access key as a proof-chain hop

Storing a tenant-to-key delegation and re-delegating from the key per request would mirror today's mechanism, let a key deletion revoke one CID, and clear the gateway through the existing revocation path. It adds a hop to every chain and makes the key's delegation carry authority, which the ADR's model says it does not.

### Policies evaluated at the gateway

Ingot embeds versitygw, whose policy engine is disabled today by returning the admin role for every authenticated request. Storing policies at Ingot and enabling that engine would evaluate them at the edge with no Hilt round trip. It puts the rule outside the system that owns it, requires policy persistence and replication at every gateway, and leaves Hilt unable to issue delegations that match the decision.

### Separate creation routes per kind

`POST /tenants/{tenantId}/service-credentials` and `POST /tenants/{tenantId}/principals/{userId}/access-keys` would make a key's kind a property of the route it was created at, so each body would hold only the fields its kind uses and there would be no field combination to reject. A service credential would reach every bucket of the tenant rather than carrying a permission list, and the scoped-key path would leave Hilt entirely. The cost is a second set of create, list, and delete routes, a second vault path, a change to the management API contract the console already calls, and a migration that revokes and deletes every existing key before any member can be served. One route with an optional `principalId` leaves the console's existing calls working and keeps the parent RFC's authorization path in place for traffic that has no member actor.

### An access model flag per tenant

A tenant could carry `scoped-keys` or `iam`, and Hilt would apply one model to every key the tenant holds. Each request would then read one model from the tenant row rather than branching on the key. A tenant cannot hold a worker key and a member key at once under that rule, and the console holds both from the moment it creates its first principal.

### A permission list on a principal-bound key

A principal-bound key could carry its own permission list, as an AWS session policy does, and be authorized against the intersection of that list and the policy result. A member could then hold a read-only key and a full key on the same buckets, and the console would not have to model the difference as two principals. It puts a second document in the path of every denial, so a 403 has two possible sources and Ingot's cached set is no longer the principal's effective set. Rejecting the fields with 422 keeps one answer to what a member may do.

### Filtering the bucket listing

Scoping `/s3/bucket/list` to the buckets a principal reaches was proposed for access keys in Hilt PR #48 and closed as a divergence from AWS, where a listing shows names the caller cannot open. The console filters its own list, and the gateway's behavior is unchanged.

## Open questions

1. What latency target does `GET /principals/{userId}/access` carry? The console resolves it per request for its bucket list and activity feed. It is an index lookup at Hilt; the number is unmeasured.
2. A policy edit naming `*` rotates the marker of every key of every principal of the tenant, one Swarf round trip each inside the write's transaction. There is no figure for principals per tenant or keys per principal on Forge. The publishes can run concurrently. If the product is large, that fan-out is the cost to watch.
3. Limits on statements per policy and principals per statement. None are specified here.
4. Bucket invalidation for a network with several gateways. Bucket deletion clears only the gateway that handled it, and Ingot's registry is per instance, so such a network needs a record the firehose carries and a registry the gateways share.

## Evaluation criteria

- The time from a change's acknowledgement to the gateway's first refusal, measured against a warm key. This is the staleness bound the ADR asks Hilt to publish.
- Authorization latency for a service key is unchanged from today's key.
- No existing key stops working at any point in the rollout.
- The console's in-memory IAM fake and Hilt pass the same contract tests.
- A policy `PUT` costs one Swarf revocation and one stored delegation per key of each principal whose access changed.
- Swarf's service and Ingot's firehose consumer need no changes.

## References

- [Forge S3 tenant management](./2026-06-forge-s3-tenant-management.md), the parent RFC.
- [Bucket access by region: scoped keys on Aurora and FTH, IAM on Forge](https://github.com/fil-one/fil-one/blob/main/docs/architectural-decisions/2026-08-bucket-policies-m2.md), the ADR implemented here.
- [Organizations, membership, and roles](https://github.com/fil-one/fil-one/blob/main/docs/architectural-decisions/2026-08-organizations-roles-m1.md), for roles and the permission registry the console applies before calling Hilt.
- [Service Orchestrator Management API](https://github.com/fil-one/fil-one/blob/main/docs/service-orchestrator-integration/management-openapi.yaml), the contract the HTTP additions land in.
- [UCAN revocation](https://github.com/ucan-wg/revocation), for path witnesses.
- Hilt PR #48, the closed proposal for a scoped bucket listing.

## Appendix

### Schema

Changes to the parent RFC's schema. Types and constraints follow the existing migrations.

```sql
CREATE TABLE principal (
    tenant_id   TEXT        NOT NULL REFERENCES tenant(id) ON DELETE RESTRICT,
    external_id TEXT        NOT NULL,                     -- console userId
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tenant_id, external_id)
);

-- A key with no principal is the parent RFC's key, authorized from its own
-- permissions and buckets. A key with one carries no authority of its own and is
-- authorized from the bucket policies, so it holds neither. Existing rows have no
-- principal and need no backfill.
ALTER TABLE access_key
    ADD COLUMN principal TEXT,                            -- console userId
    ADD FOREIGN KEY (tenant_id, principal) REFERENCES principal(tenant_id, external_id) ON DELETE RESTRICT,
    ADD CONSTRAINT access_key_principal_unscoped
        CHECK (principal IS NULL OR (permissions = '{}' AND COALESCE(buckets, '{}') = '{}'));

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

-- Which principals a policy names; '*' statements are indexed as NULL principal.
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

The `delegation` table keeps its schema; a principal-bound key's one row in it is its marker. The vault keeps the parent RFC's layout: every key at `/tenant/{tenant}/access/{accessKeyId}`, and no entry for a principal.

### Example flows

#### Migrate an existing tenant

1. The tenant's keys keep working as service keys. Fil One changes nothing about them and keeps using the tenant credential it holds.
2. Fil One calls `PUT /tenants/{tenantId}/principals/{userId}` for every member.
3. Hilt stores each principal.
4. Fil One writes each bucket's policy.
5. On a member's next request, Fil One calls `POST /tenants/{tenantId}/access-keys` with that member's `principalId`, signs their traffic with the returned key, and deletes the key they held before.

#### Grant a member a bucket

1. Fil One reads `GET /tenants/{t}/buckets/photos/policy` and its `ETag`.
2. Fil One writes the document with an added `Allow` statement, sending `If-Match`.
3. Hilt compares effective sets before and after for every named principal. The added member gained actions, so Hilt locks the policy row and the member's key rows, publishes a revocation for each key's marker, stores a new marker per key, and commits the document. The response carries the new `ETag`.
4. Swarf's firehose delivers the revocations and Ingot drops the store of every key that held one of the markers. The member's next request on `photos` reaches Hilt and is authorized against the new policy.

#### Put an object with a principal-bound key

1. A client sends `PUT /photos/cat.jpg` to Ingot, signed with the member's key.
2. Ingot finds no cached effective set for that key and bucket and invokes `/s3/request/authorize`.
3. Hilt verifies the signature, loads the key's principal, resolves `photos`, computes `effective(principal, photos)`, finds `s3:PutObject`, and signs tenant-to-gateway delegations for the six write commands with the tenant key, expiring at the next UTC midnight. It returns them with the key's marker.
4. Ingot invokes `/s3/bucket/info` for the bucket root, caches the chains, the marker, and the effective set under the key, and serves the request.
5. Later requests from the key on `photos` are authorized locally: the action is in the cached set and a chain resolves for each command.

#### Remove a member from a bucket

1. Fil One writes the policy without the member's statement, with `If-Match`.
2. Hilt finds the member's effective set on the bucket shrank, locks the policy row and the member's key rows, publishes a revocation for each key's marker, stores new markers, and commits the document. An authorize request for `photos` that arrives meanwhile waits for the commit.
3. Swarf's firehose delivers the revocations. Ingot drops the store of every key that held one of the markers.
4. The member's next request on any bucket reaches Hilt. On the removed bucket it receives `UnknownBucket`, which Ingot renders as `NoSuchBucket`.

#### Remove a member

1. Fil One calls `DELETE /tenants/{t}/principals/{userId}`.
2. Hilt locks the principal row, publishes a revocation for each of the member's key markers, strips the principal from every statement, deletes its keys and markers, and deletes the principal row.
3. Ingot drops each key's store on its revocation. The keys no longer resolve at Hilt.
