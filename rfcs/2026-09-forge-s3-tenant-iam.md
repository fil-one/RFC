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
3. Every narrowing reaches the gateway's warm caches before Hilt acknowledges it, over the revocation path that already exists.
4. The console signs a member's traffic, presigned URLs included, with a key bound to that member's principal, and keeps one tenant-wide credential for traffic that has no member actor.
5. Hilt has one authorization model. Existing tenants move to it in place.

## Concepts

### Roles

The parent RFC's roles apply. Swarf is added:

| Name  | Description                                                                                                |
| ----- | ---------------------------------------------------------------------------------------------------------- |
| Swarf | The Forge revocation service. Hilt publishes revocations to it; Ingot consumes its revocation firehose.    |

### Terms

- **Principal.** A member as Hilt knows them: `(tenant, userId)`. A principal carries no permissions and no role. It is an ed25519 `did:key` whose private key Hilt stores.
- **Bucket policy.** A bucket's own document of statements. Each statement has an effect, a list of principals, and a list of S3 actions. The policy is created with its bucket and destroyed with it.
- **Principal-bound key.** An S3 access key that authenticates a request and identifies a principal. It holds no authority of its own.
- **Service credential.** The console's tenant-wide key for traffic that has no member actor: tenant setup, bucket create and delete, and background workers. Everything a member does in the console, from listing buckets and objects to deleting objects and redeeming presigned URLs, is signed with a key bound to that member's principal.
- **Standing delegation.** The one stored delegation from a tenant to a principal. Rotating it is how a narrowing reaches the gateway.

## Hilt - Tenant API

Two kinds of credential exist on a tenant: one service credential, and any number of access keys, each bound to a principal. The parent RFC's `POST /tenants/{tenantId}/access-keys`, which created a key with its own permissions and bucket list, is removed. Key creation moves under principals; the list, get, and delete routes at the tenant level stay.

### Tenant creation

`PUT /tenants/{tenantId}` mints the service credential during setup and includes it in the 201 body. The idempotent 200 on a repeated call does not include it.

A Forge network serves one region, and a tenant is unique to its region, so the external tenant id stays the organization's UUID as the contract defines it. Hilt's one-provider-per-tenant model holds unchanged.

### Service credential

The service credential is an ed25519 `did:key`, generated and stored like an access key, at the vault path `/tenant/{tenant}/service-credential`. Hilt MUST delegate every Forge command from the tenant to the credential as powerline delegations with no expiration, exactly as the parent RFC does for a key with no bucket list. Per request, Hilt authorizes the credential for every S3 action on every bucket of the tenant and re-delegates to the gateway as the parent RFC specifies. No policy is evaluated for it.

The credential:

- MUST be returned once, in the body of the call that minted it, as `serviceCredential: { accessKeyId, secretAccessKey }`.
- MAY be minted for a tenant that has none through `POST /tenants/{tenantId}/service-credential`, which returns it the same way and answers 409 when one exists. This is the migration path for existing tenants.
- MUST NOT appear in `GET /tenants/{tenantId}/access-keys`, MUST NOT count toward `accessKeyCount`, and MUST NOT be deletable through the access-key delete route.
- MUST NOT be bound to a principal. It is not named in any statement and reaches every bucket of the tenant.
- Has no expiry. Rotation is a later RFC.

### Principals

Routes, all under `/tenants/{tenantId}/principals` and authenticated with the partner key:

| Method | Path                                 | Purpose                                                 |
| ------ | ------------------------------------ | ------------------------------------------------------- |
| PUT    | `/principals/{userId}`               | Create the principal; 200 if it exists. Idempotent.     |
| POST   | `/principals`                        | Batch create: `{"userIds": [...]}`. Idempotent per id.  |
| GET    | `/principals`                        | List principals.                                        |
| GET    | `/principals/{userId}`               | Principal detail.                                       |
| DELETE | `/principals/{userId}`               | Remove the principal (see [removal](#principal-removal)). 204 if already gone. |
| GET    | `/principals/{userId}/policies`      | Every bucket policy with a statement naming the principal or `*`. |
| GET    | `/principals/{userId}/access`        | The principal's effective actions per bucket.           |
| POST   | `/principals/{userId}/access-keys`   | Issue a key bound to the principal.                     |
| GET    | `/principals/{userId}/access-keys`   | List the principal's keys.                              |

`userId` is an opaque string supplied by the caller, unique within the tenant. The batch route exists for the console's provisioning sweep when a network moves to `iam`.

A principal is represented as:

```jsonc
{
  "userId": "8f2c...",
  "did": "did:key:z6Mk...",
  "createdAt": "2026-09-09T10:00:00Z"
}
```

On creation Hilt MUST:

1. Generate an ed25519 key. Its DID is the principal's DID. Store the private key at `/tenant/{tenant}/principal/{did}`.
2. Issue and store the standing delegation, signed with the tenant key:

```jsonc
{
  "iss": "did:plc:tenant",
  "aud": "did:key:principal",
  "sub": null,        // powerline: every bucket the tenant owns, now or later
  "cmd": "/",
  "pol": [],
  // no expiration
}
```

The standing delegation is a proof-chain hop and grants nothing at the request. Hilt holds the principal's private key and issues per-request delegations no wider than the bucket's policy allows. This is the position the parent RFC's powerline key delegations already occupy.

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
- an empty `statements` list (the caller deletes the policy instead),
- a bucket that belongs to another tenant, reported as 404.

Hilt MUST accept a `Deny` naming `*`. The management API carries no actor; the console decides whether to write one.

**Evaluation.** The effective actions of principal `p` on bucket `b` are

```
effective(p, b) = union(actions of Allow statements naming p or *)
                \ union(actions of Deny statements naming p or *)
```

plus `s3:ListAllMyBuckets` (see [action vocabulary](#action-vocabulary)). A bucket with no policy has an empty effective set for every principal. The service credential is not a principal and is not evaluated against policies.

**Compare-and-set.** `GET` returns a strong `ETag` computed over Hilt's canonical encoding of the stored document. `PUT` and `DELETE` MUST carry `If-Match` with that value; a mismatch is 412 and nothing is written. A `PUT` creating the first policy carries `If-None-Match: *` and is 412 if a policy already exists. Callers treat the ETag as opaque. A successful `PUT` returns the new `ETag`.

**Storage.** Hilt stores the document with its bucket and maintains an index from principal to the buckets whose statements name it, so the two principal reads above cost an index lookup. Statements naming `*` are indexed under the tenant. The policy row is deleted with the bucket.

### Bucket creation with a policy

A bucket created over S3 with the service credential has no policy, so no principal reaches it until the console writes one. To close that window the Tenant API gains a bucket create:

`POST /tenants/{tenantId}/buckets`

```jsonc
{
  "name": "photos",
  "policy": { "statements": [ /* ... */ ] }   // optional
}
```

Hilt MUST create the bucket exactly as `/s3/bucket/create` does (ephemeral bucket key, root delegation to the tenant, space provisioned with Sprue, routing policy applied) and commit the bucket and its policy together after Sprue provisioning succeeds. A failure leaves no bucket. The response is 201 with the bucket's name, DID, creation time, and the policy's `ETag`. 409 if the name is taken.

The gateway has not seen a bucket created here. Ingot registers it on its first `/s3/bucket/info` for the name (see [Ingot - S3 API](#ingot---s3-api)).

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

`DELETE /tenants/{tenantId}/access-keys/{accessKeyId}` is unchanged. It MUST rotate the principal's standing delegation before deleting the key (see [narrowing](#narrowing-and-the-standing-delegation)), because the key holds no delegation of its own that a revocation could name.

Access keys count toward `accessKeyCount`. Hilt enforces no key limit today, and none is added.

### Narrowing and the standing delegation

A narrowing is any change that removes an action a principal held on a bucket: a policy `PUT` or `DELETE` whose effective set for the principal shrinks, the deletion of one of the principal's keys, or the removal of the principal. Hilt MUST publish the revocations a narrowing requires before acknowledging it.

The gateway caches proof chains until the next UTC midnight and drops a cached chain only when Swarf's firehose names a delegation in it. Every chain for a principal's keys contains the principal's standing delegation, so revoking that one delegation drops every warm chain for every key the principal holds. Hilt therefore rotates the standing delegation on each narrowing:

1. Issue a new standing delegation for the principal and store it.
2. Publish a revocation of the previous one to Swarf, signed with the tenant key. The tenant issued it, so no witness path is needed.
3. Commit the change that caused the narrowing together with the new delegation, and delete the previous delegation row.

If publishing fails, Hilt MUST return 500 and MUST NOT commit the change. A retry rotates again. The previous delegation stays valid until the revocation lands, so a failed narrowing leaves the principal with its old access and never with more.

A policy write MUST compute, for each principal named in the old or new document (expanding `*` to every principal of the tenant), whether its effective set on the bucket lost any action, and rotate exactly those principals. A widening rotates nobody and publishes nothing.

The staleness bound for a narrowing is the time from Hilt's acknowledgement to the gateway consuming the firehose record. It is bounded by firehose latency and is independent of the midnight horizon.

### Principal removal

`DELETE /tenants/{tenantId}/principals/{userId}` MUST, in order:

1. Publish a revocation of the principal's standing delegation.
2. Remove the principal from every statement naming it, deleting statements left with no principal and policies left with no statement.
3. Delete the principal's keys: vault entries and rows.
4. Delete the standing delegation row, the principal's vault entry, and the principal row.

A removed member who is later re-invited gets a new principal with a new DID. No statement survives to restore their old access.

### Bucket deletion

`/s3/bucket/delete` deletes the bucket's policy with the bucket. For an `iam` tenant there are no subject-scoped delegations to revoke: principals hold powerline delegations, and per-request delegations are not stored. Hilt publishes nothing. The gateway removes the bucket from its registry on the same request, and its local authorization refuses any bucket it does not know, so cached chains for the deleted bucket cannot be used.

Deleting a tenant cascades over principals, policies, keys, and the service credential. As today, it publishes no revocations and deactivates the tenant's `did:plc`.

## Hilt - UCAN API

The commands and their argument types are unchanged. The result types gain one field, and the authorization steps gain a branch for `iam` tenants.

### `/s3/request/authorize`

For a request signed with the service credential, the parent RFC's steps apply with every action permitted on every bucket of the tenant.

For a request signed with an access key, Hilt performs the parent RFC's steps through signature verification, tenant status, issuer, and region, then:

1. Derive the S3 operation and its action from method, path, and query (see [action vocabulary](#action-vocabulary)).
2. Load the key's principal.
3. If the operation is `ListBuckets`, authorize it: every principal holds `s3:ListAllMyBuckets`.
4. If the operation is `CreateBucket` or `DeleteBucket`, refuse with `OperationNotPermitted`. No policy grants them.
5. Resolve the bucket by name. If it does not exist, or belongs to another tenant, refuse with `UnknownBucket`.
6. Compute `effective(principal, bucket)`. If it is empty, refuse with `UnknownBucket`.
7. If the action is not in the effective set, refuse with `OperationNotPermitted`.
8. Issue one per-request delegation per Forge command mapped from the action, signed with the principal's private key:

```jsonc
{
  "iss": "did:key:principal",
  "aud": "did:web:gateway",     // the invocation issuer
  "sub": "did:key:bucket",
  "cmd": "/content/retrieve",
  // exp: next UTC midnight plus clock skew, capped at the key's expiry
}
```

The expiry rule is the parent RFC's. Per-request delegations are returned in the response container and are not stored.

**Result.** `AuthorizeOK` gains an optional `principal` field, and `permissions` carries the key's effective actions on the addressed bucket:

```ipldsch
type AuthorizeOK struct {
  bucket      optional String      # bucket DID; absent for ListBuckets and CreateBucket
  tenant      String               # tenant DID
  principal   optional String      # principal DID; absent for the service credential
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
  Principal   *did.DID         `cborgen:"principal,omitempty"`
  Permissions s3.PermissionSet `cborgen:"permissions"`
  Keys        s3.KeySet        `cborgen:"keys"`
  Delegations s3.ProofSet      `cborgen:"delegations"`
}
```
</details>

For `ListBuckets` the effective actions are `["s3:ListAllMyBuckets"]`. For the service credential `permissions` carries the full vocabulary.

### `/s3/bucket/info`

For an access key the chain runs from the bucket to the principal, since the key is not a hop:

- root: bucket → tenant, `cmd: "/"`
- standing: tenant → principal, `sub: null`, `cmd: "/"`

`InfoOK` gains the same optional `principal` field, and `permissions` carries the key's effective actions on the bucket. A key whose principal cannot reach the bucket receives `UnknownBucket`.

### `/s3/bucket/create` and `/s3/bucket/delete`

Unchanged for the service credential. An access key receives `OperationNotPermitted` from both. Bucket deletion additionally removes the policy, as described above.

### `/s3/bucket/list`

Unchanged. The listing is the tenant's whole bucket set, as AWS lists names the caller cannot open. The console filters its own list against the principal's access.

### Failures

No new failure names. `UnknownBucket` covers a bucket that does not exist, belongs to another tenant, or is out of the principal's reach. `OperationNotPermitted` covers an action outside a non-empty effective set and the two bucket operations no policy can grant. Ingot's existing mapping renders them as `NoSuchBucket` (404) and `AccessDenied` (403).

## Ingot - S3 API

Ingot's request path is unchanged: authorize locally from cache when it can, and ask Hilt otherwise.

**Ingot MUST enforce the effective action set.** Today Ingot discards the `permissions` field and authorizes locally by checking that its cache holds a proof chain for each Forge command the operation needs. Several S3 actions map to the same commands: `s3:GetObject` and `s3:ListBucket` both need `/content/retrieve`, and `s3:PutObject` grants `/blob/remove` alongside `s3:DeleteObject`. A chain probe alone therefore cannot enforce a `Deny` on one of them, or a policy that grants one without the other. Ingot MUST cache the `permissions` value per access key and bucket, in the same per-key store as the chains so a revocation drops both, and MUST refuse a fast-path request whose action is not in the cached set. A request for a bucket with no cached set goes to Hilt.

**Cache invalidation is unchanged.** The firehose consumer drops every per-key store containing a revoked CID. The standing delegation is in every chain for a principal's keys, so a rotation clears them all. Ingot needs no new record type and no per-principal index.

**Buckets created through the Tenant API.** Ingot registers a bucket in its own registry only inside its `CreateBucket` handler today. It MUST also register a bucket it first learns of from a successful `/s3/bucket/info`, with default versioning and object-lock state, so a bucket created by `POST /tenants/{tenantId}/buckets` is servable.

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

1. Deploy Ingot with the effective-action check on every gateway serving the network.
2. Deploy Hilt. Its migration deletes every existing access key, publishing revocations for their delegations first, and drops the key's `permissions` and `buckets` columns. Tenants, buckets, and bucket root delegations are untouched.
3. The console calls `POST /tenants/{tenantId}/service-credential` for each existing tenant and stores the result where it keeps the tenant's console credential today.
4. The console writes principals for existing members through the batch route, and a policy per bucket naming the org's Owners and Admins.
5. The console flips the region's registry entry to `iam`.

Between steps 2 and 5 no key on the network works. Demo and dev tolerate that window.

## Alternatives considered

### Allow statements materialized as stored delegations

Hilt could write one tenant-to-principal delegation per bucket and Forge command that an Allow grants, and revoke exactly those on a narrowing. Stored authority would then equal granted authority, and revocation would name the precise delegation. It costs principals times buckets times commands rows, turns an Owner promotion into one delegation set per bucket, and still cannot express `Deny`, which would be enforced at Hilt and Ingot regardless. Rotating one standing delegation per principal produces the same cache effect from one row.

### Recorded per-request delegations revoked on narrowing

Hilt could persist each per-request delegation it issues and revoke the outstanding ones for a principal and bucket on a narrowing. The revocation would be exact to the bucket, and downstream services would never see a wider grant than the policy. It adds a table that churns once per key per bucket per day, and the gateway drops the whole per-key store on any revoked CID anyway, so the precision changes nothing at the cache.

### A new Swarf record type

An audience-scoped invalidation record naming the principal would let the standing delegation stay fixed. It changes Swarf and Ingot's consumer for a result the existing `/ucan/revoke` already produces once the standing delegation is one per principal.

### The access key as a proof-chain hop

Storing a principal-to-key delegation and re-delegating from the key per request would mirror today's mechanism and let a key deletion revoke one CID. It adds a hop to every chain and gives the key a delegation of its own, which the ADR's model says it does not have.

### Policies evaluated at the gateway

Ingot embeds versitygw, whose policy engine is disabled today by returning the admin role for every authenticated request. Storing policies at Ingot and enabling that engine would evaluate them at the edge with no Hilt round trip. It puts the rule outside the system that owns it, requires policy persistence and replication at every gateway, and leaves Hilt unable to issue delegations that match the decision.

### Principals without key material

A principal could be a plain row, with Hilt signing per-request delegations directly from the tenant key. The chain would be one hop shorter and the vault untouched. A principal that is a `did:key` can later sign its own invocations and is a real hop that a revocation can target, which is what the propagation design relies on.

### Keeping the parent RFC's keys alongside

An access model per tenant, `scoped-keys` or `iam`, would let existing tenants keep their keys until the console moved them. It leaves two authorization paths per request and two meanings for a key indefinitely. The tenants it would protect are demo and dev.

### Filtering the bucket listing

Scoping `/s3/bucket/list` to the buckets a principal reaches was proposed for access keys in Hilt PR #48 and closed as a divergence from AWS, where a listing shows names the caller cannot open. The console filters its own list, and the gateway's behavior is unchanged.

## Open questions

1. What latency target does `GET /principals/{userId}/access` carry? The console resolves it per request for its bucket list and activity feed. It is an index lookup at Hilt; the number is unmeasured.
2. A policy edit naming `*` rotates every principal of the tenant. There is no figure for principals per tenant on Forge. If it is large, the rotation is the cost to watch.
3. Limits on statements per policy and principals per statement. None are specified here.
4. Rotation of the service credential. The credential has no expiry and no rotate route.

## Evaluation criteria

- The time from a narrowing's acknowledgement to the gateway's first refusal, measured against a warm key. This is the staleness bound the ADR asks Hilt to publish.
- Authorization latency for the service credential is unchanged from today's tenant-wide key.
- The console's in-memory IAM fake and Hilt pass the same contract tests.
- A policy `PUT` costs one rotation per principal whose access shrank, and nothing else.

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
    id          TEXT        PRIMARY KEY,                  -- DID
    tenant_id   TEXT        NOT NULL REFERENCES tenant(id) ON DELETE RESTRICT,
    external_id TEXT        NOT NULL,                     -- console userId
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, external_id)
);

DELETE FROM access_key;                                   -- revocations published by the migration first
ALTER TABLE access_key
    DROP COLUMN permissions,
    DROP COLUMN buckets,
    ADD COLUMN principal_id TEXT REFERENCES principal(id) ON DELETE RESTRICT,
    ADD COLUMN service      BOOLEAN NOT NULL DEFAULT FALSE,
    ADD CONSTRAINT access_key_kind CHECK (service <> (principal_id IS NOT NULL));

CREATE UNIQUE INDEX access_key_principal_name_idx ON access_key (principal_id, name)
    WHERE principal_id IS NOT NULL;

CREATE TABLE bucket_policy (
    bucket_id  TEXT        PRIMARY KEY REFERENCES bucket(id) ON DELETE CASCADE,
    document   JSONB       NOT NULL,
    etag       TEXT        NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Which principals a policy names; '*' statements are indexed as NULL principal.
CREATE TABLE bucket_policy_principal (
    bucket_id    TEXT NOT NULL REFERENCES bucket(id) ON DELETE CASCADE,
    principal_id TEXT REFERENCES principal(id) ON DELETE CASCADE,
    UNIQUE (bucket_id, principal_id)
);
CREATE INDEX bucket_policy_principal_idx ON bucket_policy_principal (principal_id, bucket_id);
```

The standing delegation lives in the existing `delegation` table with `audience` = principal DID and `subject` NULL. The vault gains `/tenant/{tenant}/principal/{did}` and `/tenant/{tenant}/service-credential`.

### Example flows

#### Migrate an existing tenant

1. Hilt's migration has already deleted the tenant's keys and published their revocations.
2. Fil One calls `POST /tenants/{tenantId}/service-credential`. Hilt mints the credential, issues its powerline delegations, and returns it.
3. Fil One calls `POST /tenants/{tenantId}/principals` with every member's `userId`.
4. Hilt generates a key and a standing delegation per principal.
5. Fil One writes each bucket's policy.

#### Grant a member a bucket

1. Fil One reads `GET /tenants/{t}/buckets/photos/policy` and its `ETag`.
2. Fil One writes the document with an added `Allow` statement, sending `If-Match`.
3. Hilt compares effective sets before and after for every named principal. Nobody lost an action, so nothing is rotated or published. Hilt stores the document and returns the new `ETag`.
4. The member's next request on `photos` misses the gateway cache, reaches Hilt, and is authorized against the new policy.

#### Put an object with an access key

1. A client sends `PUT /photos/cat.jpg` to Ingot, signed with the member's key.
2. Ingot finds no cached effective set for that key and bucket and invokes `/s3/request/authorize`.
3. Hilt verifies the signature, loads the key's principal, resolves `photos`, computes `effective(principal, photos)`, finds `s3:PutObject`, and issues principal-to-gateway delegations for the six write commands, expiring at the next UTC midnight.
4. Ingot invokes `/s3/bucket/info` for the root and standing delegations, caches the chains and the effective set under the key, and serves the request.
5. Later requests from the key on `photos` are authorized locally: the action is in the cached set and a chain resolves for each command.

#### Remove a member from a bucket

1. Fil One writes the policy without the member's statement, with `If-Match`.
2. Hilt finds the member's effective set on the bucket shrank, issues a new standing delegation for the principal, publishes a revocation of the old one to Swarf, commits the document and the new delegation, and deletes the old delegation row.
3. Swarf's firehose delivers the revoked CID. Ingot drops every cached store containing it, which is every store for the member's keys.
4. The member's next request on any bucket reaches Hilt. On the removed bucket it receives `UnknownBucket`, which Ingot renders as `NoSuchBucket`.

#### Remove a member

1. Fil One calls `DELETE /tenants/{t}/principals/{userId}`.
2. Hilt publishes a revocation of the standing delegation, strips the principal from every statement, deletes its keys, and deletes the principal.
3. Ingot drops the member's cached stores on the firehose record. Their keys no longer resolve at Hilt.
