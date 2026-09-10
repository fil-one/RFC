# RFC: Forge S3 tenant IAM

Status: Experimental

Extends: [Forge S3 tenant management](./2026-06-forge-s3-tenant-management.md)

## Authors

- [Srdjan](https://github.com/pyropy)

## Introduction

Hilt gains principals, bucket policies, and access keys bound to principals, and Ingot enforces the actions those policies grant. Together they give the Fil One console what it needs to grant a member a subset of an organization's buckets, as decided in the [bucket access by region](https://github.com/fil-one/fil-one/blob/main/docs/architectural-decisions/2026-08-bucket-policies-m2.md) ADR.

This model replaces the [parent RFC's](./2026-06-forge-s3-tenant-management.md) access keys. Its tenant, bucket, and delegation mechanics stay in force.

## Motivation

An access key on Hilt is a row of permissions and bucket names fixed at creation. Nothing can change what a key carries afterwards, and there is no object between the tenant and its keys that a change could attach to. The console therefore cannot give a member a subset of buckets today, and cannot widen or narrow that subset later without reissuing every key the member holds.

On Forge the ADR settles three things: each member is a principal at the storage system, a bucket's own policy is the only thing that grants access, and a key belongs to a principal and carries whatever the policies give that principal, evaluated per request.

## Goals

1. The storage system computes what a principal may do on a bucket from that bucket's policy alone, `Allow` minus `Deny`, with an explicit `Deny` winning.
2. A policy edit changes what every key bound to a named principal may do, with no key reissued.
3. Every change to a principal's access is published to Swarf before Hilt acknowledges it, and Ingot's firehose consumer clears the gateway's warm caches within firehose latency.
4. The console signs a member's traffic, presigned URLs included, with a key bound to that member's principal, and keeps a tenant-wide service credential for traffic that has no member actor.
5. Hilt has one authorization model. Existing tenants move to it in place.

## Concepts

### Roles

The parent RFC's roles apply. Swarf is added:

| Name  | Description                                                                                                |
| ----- | ---------------------------------------------------------------------------------------------------------- |
| Swarf | The Forge revocation service. Hilt publishes revocations and principal invalidations to it; Ingot consumes its firehose. |

### Terms

- **Principal.** A member as Hilt knows them: `(tenant, userId)` and nothing more. A principal carries no permissions, no role, and no key material.
- **Bucket policy.** A bucket's own document of statements. Each statement has an effect, a list of principals, and a list of S3 actions. The policy is created with its bucket and destroyed with it.
- **Principal-bound key.** An S3 access key that authenticates a request and identifies a principal. It holds no authority of its own.
- **Service credential.** A tenant-wide key for traffic that has no member actor: tenant setup, bucket create and delete, and background workers. A tenant may hold several. Everything a member does in the console, from listing buckets and objects to deleting objects and redeeming presigned URLs, is signed with a key bound to that member's principal.
- **Principal invalidation.** The Swarf record that tells every gateway to forget the proofs it cached for a principal's keys. Publishing it is how a policy change reaches the gateway.

## Hilt - Tenant API

Two kinds of credential exist on a tenant: service credentials, which reach every bucket, and access keys, each bound to a principal. The parent RFC's `POST /tenants/{tenantId}/access-keys`, which created a key with its own permissions and bucket list, is removed. Key creation moves under principals; the list, get, and delete routes at the tenant level stay.

### Tenant creation

`PUT /tenants/{tenantId}` mints a service credential during setup and includes it in the 201 body. The idempotent 200 on a repeated call does not include it.

A Forge network serves one region, and a tenant is unique to its region, so the external tenant id stays the organization's UUID as the contract defines it. Hilt's one-provider-per-tenant model holds unchanged.

### Service credential

A service credential is an ed25519 `did:key`, generated and stored like an access key, at the vault path `/tenant/{tenant}/service-credential/{accessKeyId}`. Hilt MUST delegate every Forge command from the tenant to the credential as powerline delegations with no expiration, exactly as the parent RFC does for a key with no bucket list. Per request, Hilt authorizes the credential for every S3 action on every bucket of the tenant and re-delegates to the gateway as the parent RFC specifies. No policy is evaluated for it.

Routes, under `/tenants/{tenantId}/service-credentials` and authenticated with the partner key:

| Method | Path                                  | Purpose                                                     |
| ------ | ------------------------------------- | ----------------------------------------------------------- |
| POST   | `/service-credentials`                | Mint a credential. 201 with the secret.                     |
| GET    | `/service-credentials`                | List credentials: `accessKeyId` and `createdAt`, no secret. |
| DELETE | `/service-credentials/{accessKeyId}`  | Delete a credential. 204 if already gone.                   |

A credential:

- MUST be returned once, in the body of the call that minted it, as `serviceCredential: { accessKeyId, secretAccessKey }`. A caller that loses that response mints another and deletes the one it cannot use.
- MUST NOT appear in `GET /tenants/{tenantId}/access-keys`, MUST NOT count toward `accessKeyCount`, and MUST NOT be deletable through the access-key delete route.
- MUST NOT be bound to a principal. It is not named in any statement and reaches every bucket of the tenant.
- Has no expiry. Rotation is minting a new credential, moving the caller to it, and deleting the old one.

Deleting a credential MUST publish revocations for its powerline delegations before deleting the vault entry and the row, as the parent RFC's key delete does. `POST /tenants/{tenantId}/service-credentials` is also the migration path for a tenant that predates this model.

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
| POST   | `/principals/{userId}/access-keys`   | Issue a key bound to the principal.                     |
| GET    | `/principals/{userId}/access-keys`   | List the principal's keys.                              |

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

A bucket with no policy has an empty effective set for every principal. `s3:ListAllMyBuckets` is outside the per-bucket set: every principal holds it, and `ListBuckets` is authorized without consulting any policy (see [action vocabulary](#action-vocabulary)). A service credential is not a principal and is not evaluated against policies.

**Compare-and-set.** `GET` returns a strong `ETag` computed over Hilt's canonical encoding of the stored document. `PUT` and `DELETE` MUST carry `If-Match` with that value; a mismatch is 412 and nothing is written. A `PUT` creating the first policy carries `If-None-Match: *` and is 412 if a policy already exists. Callers treat the ETag as opaque. A successful `PUT` returns the new `ETag`.

**Storage.** Hilt stores the document with its bucket and maintains an index from principal to the buckets whose statements name it, so the two principal reads above cost an index lookup. Statements naming `*` are indexed under the tenant. The policy row is deleted with the bucket.

### Buckets without a policy

A bucket created over S3 with a service credential has no policy, and no principal reaches it until one is written. The console creates a bucket over S3 and writes its policy with `If-None-Match: *` before answering the member, so a member never learns of a bucket they cannot open. A console failure between the two calls leaves a bucket only service credentials reach. The console writes the policy on its retry, since a repeated `CreateBucket` for a name the tenant owns is reported as already owned.

### Principal-bound access keys

`POST /tenants/{tenantId}/principals/{userId}/access-keys`

```jsonc
{
  "name": "laptop",                            // unique per principal
  "expiresAt": "2026-12-31T00:00:00Z"          // optional
}
```

Hilt MUST generate and store the key exactly as the parent RFC specifies for an access key, and MUST record the principal on the key row. It MUST NOT issue any delegation to the key. The response is the parent RFC's `CreatedAccessKey` with a `principal` field carrying the `userId` in place of `permissions` and `buckets`. 409 on a duplicate name for that principal.

The console holds one such key per member for its own traffic, minted on the member's first request in the region, and signs that member's bucket listing, object listing, object reads and deletes, and presigned URLs with it. Hilt does not distinguish it from a key the member created: it is bound to the same principal, authorized against the same policies, and deleted with the principal. A presigned URL signed with it is authorized against the member's policies when it is redeemed, with the parent RFC's signature time bounds and the key's own expiry applying as for any request.

`GET /tenants/{tenantId}/access-keys` and `GET /tenants/{tenantId}/access-keys/{accessKeyId}` return the `principal` field. The key list does not carry each key's access; the console reads a principal's access once through the principal route instead of once per key.

`DELETE /tenants/{tenantId}/access-keys/{accessKeyId}` is unchanged. It MUST publish a principal invalidation before deleting the key (see [principal invalidation](#policy-changes-and-principal-invalidation)), because the key holds no delegation of its own that a revocation could name.

Access keys count toward `accessKeyCount`. Hilt enforces no key limit today, and none is added.

### Policy changes and principal invalidation

A policy `PUT` or `DELETE` changes the effective set of the principals it names, `*` expanding to every principal of the tenant. Hilt MUST compute, for each principal named in the old or new document, whether its effective set on the bucket changed, and MUST publish a principal invalidation for each such principal before acknowledging the write. Deleting one of a principal's keys and removing a principal each publish one invalidation for that principal.

The gateway caches proof chains and effective action sets per access key until the next UTC midnight. Nothing in those chains identifies the principal, so no delegation revocation can clear them. The invalidation record names the principal directly (see [Swarf - principal invalidation](#swarf---principal-invalidation)), and the gateway drops every store it holds for that principal's keys. A widening needs the record as much as a narrowing: the gateway refuses an action outside a cached set on its own, so a grant would otherwise wait for the cache to expire.

Hilt MUST publish inside the transaction that commits the change, after locking the rows the change writes and before committing them. `/s3/request/authorize` reads the bucket's policy row, the key row, and the principal row with `SELECT ... FOR SHARE`, so an authorize request that arrives between the publish and the commit waits for the commit and is answered from the new state. Without the lock, a gateway that had consumed the record could refill its cache from the old policy and hold it until midnight. Authorize requests for a bucket wait one Swarf round trip during a write to that bucket's policy. The lock does not cover an authorize answered before the lock was taken whose response reaches the gateway after the record does. Swarf polls its store once a second before it emits a record, so that response would have to be in flight for longer than that.

If publishing fails, Hilt MUST return 500 and MUST NOT commit the change. A retry publishes again. A failed change leaves the principal with its old access and never with more. A record published for a change that did not commit is harmless: the gateway refills from the state Hilt holds.

Deleting a key publishes one invalidation for the key's principal; the principal's other keys lose their cached proofs too and refill on their next request.

The staleness bound for a change is the time from Hilt's acknowledgement to the gateway consuming the firehose record. It is bounded by firehose latency and is independent of the midnight horizon. The record clears gateway caches and nothing else: a per-request delegation the gateway already holds in flight stays valid at Sprue and Piri until its expiry, because those services check revocation by delegation CID and the record names none.

### Principal removal

`DELETE /tenants/{tenantId}/principals/{userId}` MUST, in one transaction and in order:

1. Lock the principal row and publish a principal invalidation.
2. Remove the principal from every statement naming it, deleting statements left with no principal and policies left with no statement.
3. Delete the principal's keys: vault entries and rows.
4. Delete the principal row.

A removed member who is later re-invited gets a new principal row. No statement survives to restore their old access.

### Bucket deletion

`/s3/bucket/delete` deletes the bucket's policy with the bucket. There is nothing to revoke: principals hold no delegations, and per-request delegations are not stored. Hilt publishes nothing. The gateway removes the bucket from its registry on the same request, and its local authorization refuses any bucket it does not know, so cached chains for the deleted bucket cannot be used. This holds because a Forge network runs one gateway: Ingot's registry and caches are local to the instance, and a second gateway serving the same network would keep serving the bucket from its own caches until they expired. A network with several gateways needs a bucket invalidation record, which this RFC does not define (see [open questions](#open-questions)).

Deleting a tenant cascades over principals, policies, keys, and service credentials. As today, it publishes no revocations and deactivates the tenant's `did:plc`.

## Hilt - UCAN API

The commands and their argument types are unchanged. The result types gain one field, and the authorization steps gain a branch for access keys.

### `/s3/request/authorize`

For a request signed with a service credential, the parent RFC's steps apply with every action permitted on every bucket of the tenant.

For a request signed with an access key, Hilt performs the parent RFC's steps through signature verification, tenant status, issuer, and region, then:

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

The expiry rule is the parent RFC's. Per-request delegations are returned in the response container and are not stored. Hilt already loads the tenant key to provision buckets, so signing with it adds no vault read the request path does not have today.

**Result.** `AuthorizeOK` gains an optional `principal` field, and `permissions` carries the key's effective actions on the addressed bucket:

```ipldsch
type AuthorizeOK struct {
  bucket      optional String      # bucket DID; absent for ListBuckets and CreateBucket
  tenant      String               # tenant DID
  principal   optional String      # principal userId; absent for a service credential
  permissions { String: [String] } # access key DID → effective actions on `bucket`
  keys        { String: [VerificationKey] }
  delegations { String: [Link] }
}
```

<details>
<summary>Go syntax</summary>

```go
type AuthorizeOK struct {
  Bucket      *did.DID         `cborgen:"bucket,omitempty"`
  Tenant      did.DID          `cborgen:"tenant"`
  Principal   *string          `cborgen:"principal,omitempty"`
  Permissions s3.PermissionSet `cborgen:"permissions"`
  Keys        s3.KeySet        `cborgen:"keys"`
  Delegations s3.ProofSet      `cborgen:"delegations"`
}
```
</details>

For `ListBuckets` the effective actions are `["s3:ListAllMyBuckets"]`. For a service credential `permissions` carries the full vocabulary. `tenant` and `principal` together identify the principal, and are what the gateway indexes its caches by.

### `/s3/bucket/info`

For an access key the chain is the bucket root alone, bucket → tenant with `cmd: "/"`. Neither the principal nor the key is a hop.

`InfoOK` gains the same optional `principal` field, and `permissions` carries the key's effective actions on the bucket. A key whose principal cannot reach the bucket receives `UnknownBucket`.

### `/s3/bucket/create` and `/s3/bucket/delete`

Unchanged for a service credential. An access key receives `OperationNotPermitted` from both. Bucket deletion additionally removes the policy, as described above.

### `/s3/bucket/list`

Unchanged. The listing is the tenant's whole bucket set, as AWS lists names the caller cannot open. The console filters its own list against the principal's access.

### Failures

No new failure names. `UnknownBucket` covers a bucket that does not exist, belongs to another tenant, or is out of the principal's reach. `OperationNotPermitted` covers an action outside a non-empty effective set and the two bucket operations no policy can grant. Ingot's existing mapping renders them as `NoSuchBucket` (404) and `AccessDenied` (403).

## Swarf - principal invalidation

Swarf gains one command and one firehose event. Both carry a principal identity and no delegation.

### `/principal/invalidate`

* Issuer: Hilt, as its service identity (`did:web:auth.<network>`)
* Audience: Swarf
* Subject: Swarf

Records that every proof a gateway cached for the principal's keys is void.

#### Arguments

```ipldsch
type InvalidateArguments struct {
  tenant    String   # tenant DID
  principal String   # userId, unique within the tenant
}
```

<details>
<summary>Go syntax</summary>

```go
type InvalidateArguments struct {
  Tenant    did.DID `cborgen:"tenant"`
  Principal string  `cborgen:"principal"`
}
```
</details>

#### Result

A unit result (`{}`).

#### Authorization

There is no delegation to the principal, so no witness path can prove the invoker's authority over it. Swarf MUST accept the command only from issuers in a configured publisher list, and MUST refuse it from anyone else. The list holds the Hilt service identities of the networks Swarf serves.

### Firehose

`GET /revocations/:from` emits a second event kind alongside `revocation`:

```
id: bafyreif5fzax7oygfafacvxq2ndhtkshz2av5m42hqeixea7giirdxe5dm
event: principal
data: {"tenant":"did:plc:tenant","principal":"8f2c...","cause":{"/":"bafyreif5fz..."},"recorded_at":"2026-09-09T10:00:00Z"}
```

The cursor, the inclusive resume rule, and deduplication by `cause` are the same as for revocation events. `GET /revocation/:cid` is unaffected: the record revokes no CID, and a verifier that checks proofs by CID never sees it.

## Ingot - S3 API

Ingot's request path is unchanged: authorize locally from cache when it can, and ask Hilt otherwise.

**Ingot MUST enforce the effective action set.** Today Ingot discards the `permissions` field and authorizes locally by checking that its cache holds a proof chain for each Forge command the operation needs. Several S3 actions map to the same commands: `s3:GetObject` and `s3:ListBucket` both need `/content/retrieve`, and `s3:PutObject` grants `/blob/remove` alongside `s3:DeleteObject`. A chain probe alone therefore cannot enforce a `Deny` on one of them, or a policy that grants one without the other. Ingot MUST cache the `permissions` value per access key and bucket, in the same per-key store as the chains so an invalidation drops both, and MUST refuse a fast-path request whose action is not in the cached set. A request for a bucket with no cached set goes to Hilt. The two bucket-configuration reads map to no Forge command and are authorized at Hilt on every request, as every bucket-level operation is today. No per-request delegation carries the key's expiry for them, so a cached set alone would outlive an expired key. Ingot serves the value from its registry once Hilt answers.

**The firehose consumer gains a branch.** It drops every per-key store containing a revoked CID today. On a `principal` event it MUST drop every store for access keys whose authorize result carried the event's tenant and principal, which requires Ingot to index its per-key stores by that pair. Dropping the proof stores is enough; the derived signing keys may stay, since a request with a signing key and no chain goes to Hilt. Like the existing revoker, the handler is idempotent and an event matching nothing is a no-op.

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

Excluded from policies: `s3:CreateBucket` and `s3:DeleteBucket`, because a key holding them acts outside the policy that granted it, and `s3:ListAllMyBuckets`, which every principal holds.

The two bucket-configuration reads are new to Hilt. Its operation classifier MUST recognize `GET /{bucket}?versioning` as `GetBucketVersioning` and `GET /{bucket}?object-lock` as `GetBucketObjectLockConfiguration`; today both classify as `ListBucket`. The management API's action enum, now used by policy statements, gains both, along with `s3:AbortMultipartUpload` and `s3:ListMultipartUploadParts`, which Hilt already accepts.

Retention and legal-hold writes pass through the API like any other action. The rule that only an Owner may grant them is the console's, and Hilt does not know it.

## Migration

Forge runs demo and dev environments only, so existing tenants are migrated in place. Per Forge network:

1. Deploy Swarf with the publisher list, and Ingot with the effective-action check and the `principal` event branch, on every gateway serving the network.
2. Deploy Hilt. Its migration deletes every existing access key, publishing revocations for their delegations first, and drops the key's `permissions` and `buckets` columns. Tenants, buckets, and bucket root delegations are untouched.
3. The console calls `POST /tenants/{tenantId}/service-credentials` for each existing tenant and stores the result where it keeps the tenant's console credential today.
4. The console creates a principal for each existing member with `PUT /tenants/{tenantId}/principals/{userId}`, then writes a policy per bucket naming the org's Owners and Admins.
5. The console flips the region's registry entry to `iam`.

Between steps 2 and 5 no key on the network works. Demo and dev tolerate that window.

## Alternatives considered

### Allow statements materialized as stored delegations

Hilt could write one tenant-to-principal delegation per bucket and Forge command that an Allow grants, and revoke exactly those on a narrowing. Stored authority would then equal granted authority, and revocation would name the precise delegation. It costs principals times buckets times commands rows, turns an Owner promotion into one delegation set per bucket, and still cannot express `Deny`, which would be enforced at Hilt and Ingot regardless. One invalidation record per principal produces the same cache effect from no rows.

### Recorded per-request delegations revoked on narrowing

Hilt could persist each per-request delegation it issues and revoke the outstanding ones for a principal and bucket on a narrowing. The revocation would be exact to the bucket, and downstream services would never see a wider grant than the policy. It adds a table that churns once per key per bucket per day, and the gateway drops the whole per-key store on any revoked CID anyway, so the precision changes nothing at the cache.

### A did:key principal with a standing delegation

Each principal could be an ed25519 key Hilt stores, holding one powerline delegation from the tenant that sits in every chain for the principal's keys. A narrowing would then rotate that delegation and revoke the old one by CID through Swarf's existing command, so Swarf and Ingot's consumer would change nothing and every downstream verifier would honor the revocation. A did:key principal could also sign its own invocations one day. It costs a vault entry and a stored delegation per member, a hop in every proof chain, and a rotation on every narrowing. The invalidation record trades all of that for a signal only gateways act on.

### The access key as a proof-chain hop

Storing a tenant-to-key delegation and re-delegating from the key per request would mirror today's mechanism, let a key deletion revoke one CID, and clear the gateway through the existing revocation path. It adds a hop to every chain and gives the key a delegation of its own, which the ADR's model says it does not have, and a policy edit would still have nothing to revoke.

### Policies evaluated at the gateway

Ingot embeds versitygw, whose policy engine is disabled today by returning the admin role for every authenticated request. Storing policies at Ingot and enabling that engine would evaluate them at the edge with no Hilt round trip. It puts the rule outside the system that owns it, requires policy persistence and replication at every gateway, and leaves Hilt unable to issue delegations that match the decision.

### Keeping the parent RFC's keys alongside

An access model per tenant, `scoped-keys` or `iam`, would let existing tenants keep their keys until the console moved them. It leaves two authorization paths per request and two meanings for a key indefinitely. The tenants it would protect are demo and dev.

### Filtering the bucket listing

Scoping `/s3/bucket/list` to the buckets a principal reaches was proposed for access keys in Hilt PR #48 and closed as a divergence from AWS, where a listing shows names the caller cannot open. The console filters its own list, and the gateway's behavior is unchanged.

## Open questions

1. What latency target does `GET /principals/{userId}/access` carry? The console resolves it per request for its bucket list and activity feed. It is an index lookup at Hilt; the number is unmeasured.
2. A policy edit naming `*` publishes one invalidation per principal of the tenant. There is no figure for principals per tenant on Forge. If it is large, that fan-out is the cost to watch.
3. Swarf trusts the publisher list and checks no proof. Whether Swarf should verify that a tenant owns the named principal, for example by consulting Hilt, is open.
4. Limits on statements per policy and principals per statement. None are specified here.
5. Bucket invalidation for a network with several gateways. Bucket deletion clears only the gateway that handled it, and Ingot's registry is per instance, so such a network needs a record the firehose carries and a registry the gateways share.

## Evaluation criteria

- The time from a change's acknowledgement to the gateway's first refusal, measured against a warm key. This is the staleness bound the ADR asks Hilt to publish.
- Authorization latency for a service credential is unchanged from today's tenant-wide key.
- The console's in-memory IAM fake and Hilt pass the same contract tests.
- A policy `PUT` costs one Swarf record per principal whose access changed, and no delegation writes.

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

DELETE FROM access_key;                                   -- revocations published by the migration first
ALTER TABLE access_key
    DROP COLUMN permissions,
    DROP COLUMN buckets,
    ADD COLUMN principal TEXT,                            -- console userId
    ADD COLUMN service   BOOLEAN NOT NULL DEFAULT FALSE,
    ADD FOREIGN KEY (tenant_id, principal) REFERENCES principal(tenant_id, external_id) ON DELETE RESTRICT,
    ADD CONSTRAINT access_key_kind CHECK (service <> (principal IS NOT NULL));

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

The `delegation` table is unchanged. The vault gains `/tenant/{tenant}/service-credential/{accessKeyId}` and nothing for principals.

### Example flows

#### Migrate an existing tenant

1. Hilt's migration has already deleted the tenant's keys and published their revocations.
2. Fil One calls `POST /tenants/{tenantId}/service-credentials`. Hilt mints the credential, issues its powerline delegations, and returns it.
3. Fil One calls `PUT /tenants/{tenantId}/principals/{userId}` for every member.
4. Hilt stores each principal.
5. Fil One writes each bucket's policy.

#### Grant a member a bucket

1. Fil One reads `GET /tenants/{t}/buckets/photos/policy` and its `ETag`.
2. Fil One writes the document with an added `Allow` statement, sending `If-Match`.
3. Hilt compares effective sets before and after for every named principal. The added member gained actions, so Hilt locks the policy row, invokes `/principal/invalidate` on Swarf for them, and commits the document. The response carries the new `ETag`.
4. Swarf's firehose delivers the `principal` event and Ingot drops every cached store for the member's keys. The member's next request on `photos` reaches Hilt and is authorized against the new policy.

#### Put an object with an access key

1. A client sends `PUT /photos/cat.jpg` to Ingot, signed with the member's key.
2. Ingot finds no cached effective set for that key and bucket and invokes `/s3/request/authorize`.
3. Hilt verifies the signature, loads the key's principal, resolves `photos`, computes `effective(principal, photos)`, finds `s3:PutObject`, and signs tenant-to-gateway delegations for the six write commands with the tenant key, expiring at the next UTC midnight.
4. Ingot invokes `/s3/bucket/info` for the bucket root, caches the chains and the effective set under the key, indexed by tenant and principal, and serves the request.
5. Later requests from the key on `photos` are authorized locally: the action is in the cached set and a chain resolves for each command.

#### Remove a member from a bucket

1. Fil One writes the policy without the member's statement, with `If-Match`.
2. Hilt finds the member's effective set on the bucket shrank, locks the policy row, invokes `/principal/invalidate` on Swarf for the member, and commits the document. An authorize request for `photos` that arrives meanwhile waits for the commit.
3. Swarf's firehose delivers the `principal` event. Ingot drops every cached store for the member's keys.
4. The member's next request on any bucket reaches Hilt. On the removed bucket it receives `UnknownBucket`, which Ingot renders as `NoSuchBucket`.

#### Remove a member

1. Fil One calls `DELETE /tenants/{t}/principals/{userId}`.
2. Hilt locks the principal row, invokes `/principal/invalidate` on Swarf, strips the principal from every statement, deletes its keys, and deletes the principal row.
3. Ingot drops the member's cached stores on the `principal` event. Their keys no longer resolve at Hilt.
