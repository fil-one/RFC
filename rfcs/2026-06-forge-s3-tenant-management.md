# RFC: Forge S3 tenant management

Status: experimental

This document proposes how the Forge manages Fil One tenants and how S3 style access credentials are created and mapped to UCAN authorizations. In this context a tenant is roughly analogous to a Fil One _organization_, or an AWS _account_.

Pre-reading: [Fil One Service Orchestrator API](https://github.com/fil-one/fil-one/blob/main/docs/architectural-decisions/2026-04-service-orchestrator-management-api.md) ADR to gain context on how Fil One manages tenants.

## Concepts

### Roles

| Name    | Description                                                  |
| ------- | ------------------------------------------------------------ |
| Fil One | The backend of the fil.one console application.              |
| Sprue   | The Forge network upload service.                            |
| Hilt    | Service for managing tenants of Ingot and their secret keys. |
| Ingot   | An S3 facade typically co-located with a Forge Piri node.    |
| Piri    | A Forge network storage node.                                |
| Tenant  | A Fil One organization.                                      |


## Hilt - Tenant API

A trusted centralized service for tenant management exists so that storing secrets is not a requirement for Ingot deployments. Hilt implements the [Tenant API](https://github.com/fil-one/fil-one/blob/main/docs/service-orchestrator-integration/management-openapi.yaml), provides a UCAN API for retrieving proof chains for invocations into the Forge network and speaks to the Forge upload service.

Hilt MUST be configured with a pre-shared bearer key allowing only the **Fil One service** to call its [Tenant API](https://github.com/fil-one/fil-one/blob/main/docs/service-orchestrator-integration/management-openapi.yaml).

The tenant API enables "tenants" and S3 style access keys to be created. Both concepts correspond to private keys and are stored securely by Hilt.

### Tenant creation

Hilt MUST create and store a secret key per tenant. Per tenant keys limit blast radius in case of a security breach.

Tenant keys are used to provision buckets with Sprue - the Forge upload service.

Tenant keys MUST be cryptographic key pairs, they MUST also be `did:plc` keys, allowing them to be rotated if necessary.

After Ingot creates a tenant key, it MUST issue a `/account/add` invocation to Sprue to register the account. Note: this is a new capability that does not exist at time of writing. The means of obtaining authority (delegation) needed for Ingot to invoke this capability is out of scope of this document.

### Bucket creation

A bucket MUST be a cryptographic key pair. They SHOULD be ed25519 keys although other key types MAY be used. A bucket is a space in Forge.

A bucket is created by Hilt via the tenant API. After creation, a delegation to the tenant MUST be issued and stored. The delegation MUST grant the tenant full authority over the bucket (issuer = bucket, audience = tenant, subject = bucket, command = "/"). See [top](https://github.com/ucan-wg/spec#-aka-top). e.g.

```jsonc
{
  "iss": "did:key:bucket",
  "aud": "did:plc:tenant",
  "sub": "did:key:bucket",
  "cmd": "/",
  "pol": [],
  // ...
}
```

Hilt MUST then issue a `/provider/add` invocation to Sprue to register the bucket (space) and assert the bucket's ownership by the tenant.

#### Bucket name mapping

Hilt MUST maintain a mapping of bucket names to identifiers (DIDs) so that requests with bucket names in the URL can be mapped to bucket identifiers for authorization (see [access key creation](#access-key-creation)).

### Access key creation

Hilt MUST create and store a secret key per S3 access key.

Access keys MUST be cryptographic keys. They SHOULD be ed25519 keys. The `accessKeyId` is the ed25519 public key, encoded as a DID but with `did:key:` prefix removed. The `secretAccessKey` is the 32 byte ed25519 private key, prefixed with multiformat varint for ed25519 (`0x1300`) and encoded using multibase `base64url`.

You might generate this in Go with the following code:

```go
package main

import (
	"fmt"
	"github.com/fil-forge/ucantone/principal/ed25519"
	"github.com/multiformats/go-multibase"
)

func main() {
	sk, _ := ed25519.Generate()
	fmt.Printf("accessKeyId: %s\n", sk.DID().Identifier())
	secretAccessKey, _ := multibase.Encode(multibase.Base64url, sk.Bytes())
	fmt.Printf("secretAccessKey: %s\n", secretAccessKey)
}
```

Example output:

```sh
accessKeyId: z6MkjFRxLLGdBqQSLkZbVnuwUFiomK8eGBkPtim9ETvP7vec
secretAccessKey: ugCa2qwJXrMAE7WGQe1n2XrdQri_wigrO_mOU1OX75cXx7w
```

Currently Fil One supports the following S3 permissions:

Object-level:

- `s3:GetObject`, `s3:GetObjectVersion`, `s3:GetObjectRetention`, `s3:GetObjectLegalHold`
- `s3:PutObject`, `s3:PutObjectRetention`, `s3:PutObjectLegalHold`
- `s3:DeleteObject`, `s3:DeleteObjectVersion`
- `s3:ListBucket`, `s3:ListBucketVersions`

Bucket-level:

- `s3:CreateBucket`, `s3:ListAllMyBuckets`, `s3:DeleteBucket`

Note: there is an AWS quirk where `s3:ListBucket` lists _objects_ in a bucket, while `s3:ListAllMyBuckets` lists buckets. Similarly, `s3:ListBucketVersions` allows listing _object_ versions in a bucket.

When an access key is created, Forge related permissions are delegated from the _tenant_ to the _access key_, depending on what S3 permissions are selected:

| S3 Permission            | Forge Command                                                 |
| ------------------------ | ------------------------------------------------------------- |
| `s3:GetObject`           | `/content/retrieve`                                           |
| `s3:GetObjectVersion`    | `/content/retrieve`                                           |
| `s3:GetObjectRetention`  | `/content/retrieve`                                           |
| `s3:GetObjectLegalHold`  | `/content/retrieve`                                           |
| `s3:PutObject`           | `/content/retrieve`, `/blob/add`, `/index/add`, `/upload/add` |
| `s3:PutObjectRetention`  | `/content/retrieve`, `/blob/add`, `/index/add`, `/upload/add` |
| `s3:PutObjectLegalHold`  | `/content/retrieve`, `/blob/add`, `/index/add`, `/upload/add` |
| `s3:DeleteObject`        | `/blob/remove`, `/upload/remove`                              |
| `s3:DeleteObjectVersion` | `/blob/remove`, `/upload/remove`                              |
| `s3:ListBucket`          | `/content/retrieve`                                           |
| `s3:ListBucketVersions`  | `/content/retrieve`                                           |
| `s3:CreateBucket`        | n/a                                                           |
| `s3:ListAllMyBuckets`    | n/a                                                           |
| `s3:DeleteBucket`        | n/a                                                           |

Observe that in the table above, multiple permissions map to the same forge command and some permissions do not have an equivalent Forge command. Hilt and Ingot MUST be able to determine if an access key is authorized to perform the S3 action and cannot do this by simply receiving Forge delegations alone.

To address this problem and facilitate authorization at Ingot, delegations for accessing the Forge network MUST be accompanied by the list of S3 permissions assigned to the access key. See [`/s3/request/authorize`](#s3requestauthorize).

Given a tenant key `did:plc:tenant`, a bucket `did:key:bucket`, and an access key `did:key:access`, granting the `s3:GetObject` and `s3:PutObject` permissions involves creating and storing the following 4 UCAN delegations:

```jsonc
{
  "iss": "did:plc:tenant",
  "aud": "did:key:access",
  "sub": "did:key:bucket",
  "cmd": "/content/retrieve",
  // ...
}
```

```jsonc
{
  "iss": "did:plc:tenant",
  "aud": "did:key:access",
  "sub": "did:key:bucket",
  "cmd": "/blob/add",
  // ...
}
```

```jsonc
{
  "iss": "did:plc:tenant",
  "aud": "did:key:access",
  "sub": "did:key:bucket",
  "cmd": "/index/add",
  // ...
}
```

```jsonc
{
  "iss": "did:plc:tenant",
  "aud": "did:key:access",
  "sub": "did:key:bucket",
  "cmd": "/upload/add",
  // ...
}
```

Delegations MUST be indexed by audience and MAY be indexed by subject and command as well. This allows efficient lookup when validating a request and facilitates removal when an access key is deleted.

For multiple buckets, multiple delegations are created. Powerline delegations MAY be created to allow access to _any_ bucket existing or future (see [bucket access](#bucket-access)).

S3 permissions where there is not an equivalent Forge delegation MUST be handled directly by Ingot or forwarded to a UCAN API at Hilt.

### Bucket access

It's typical to allow the access key to access any bucket created by the tenant that exists at the time of creation as well as any bucket that may exist at any time in the future. This is modeled as a "powerline" delegation, where the UCAN subject is `null`.

### Key expiry

Key expiry is simply a UCAN expiration time set on the delegations created for the key.

## Hilt - UCAN API

Hilt is a trusted centralized service for tenant management exists so that storing secrets is not a requirement for Ingot deployments. Hilt provides a UCAN API for Ingot to retrieving proof chains for invocations into the Forge network and it also speaks to the Forge upload service.

Hilt provides a UCAN RPC API for the following commands:

### `/s3/request/authorize`

* Issuer: Ingot
* Audience: Hilt
* Subject: Hilt

Authorizes AWS S3 API requests. Given the incoming request and Sigv4 signature, this handler:

1. Determines validity of the signature.
1. Derives a Sigv4 signing key from the access key's private key.
1. Delegates capabilities needed for the request to the **invocation issuer**.
1. Returns the derived signing key and delegations.

#### Arguments

**IPLD schema**

```ipldsch
type AuthorizeArguments struct {
  request Request # S3 API request to put object
}

# AWS API request
type Request struct {
  method  String
  headers { String: String }
  url     String
  # TODO: needs hashed payload?
}
```

<details>
<summary>Go syntax</summary>

```go
type AuthorizeArguments struct {
  Request Request `cborgen:"request"`
}

type Request struct {
  Method  string            `cborgen:"method"`
  Headers map[string]string `cborgen:"headers"`
  URL     string            `cborgen:"url"`
}
```
</details>

e.g.

```jsonc
// Encoded as dag-json for readability
{
  "request": {
    "method": "GET",
    "headers": {
      "host": "region.s3.fil.one"
    },
    "url": "https://region.s3.fil.one/bucket/path?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=z6MkjFRxLLGdBqQSLkZbVnuwUFiomK8eGBkPtim9ETvP7vec%2F20260616%2Fauto%2Fs3%2Faws4_request&X-Amz-Date=20260616T091923Z&X-Amz-Expires=86400&X-Amz-Signature=a3eaedd5acb2512ed6ffa19d23f7425f9c7af3ee84ed3ad711eb38c53f3a4112&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject",
  }
}
```

#### Result

A successful authorization will return a SigV4/SigV4a derived signing key that can be used to verify requests, and a set of delegations for the invocation issuer that allows access to the Forge network. The delegations MUST have a 24 hour or less TTL.

**IPLD schema**

```ipldsch
type AuthorizeOK struct {
  bucket      optional String      # DID (optional since not all requests address a bucket e.g. CreateBucket, ListAllMyBuckets etc.)
  permissions { String: [String] } # S3 permission set for the access key
  keys        { String: [VerificationKey] }
  delegations { String: [Link] }
}

type KeyKind enum {
   | sigv4
   | sigv4a
}

type VerificationKey struct {
   kind KeyKind
   data Bytes
}
```

<details>
<summary>Go syntax</summary>

```go
type AuthorizeOK struct {
  Bucket      *did.DID                      `cborgen:"bucket,omitempty"`
  Permissions map[did.DID][]string          `cborgen:"permissions"`
  Keys        map[did.DID][]VerificationKey `cborgen:"keys"`
  Delegations map[cid.Cid][]cid.Cid         `cborgen:"delegations"`
}

type VerificationKey struct {
  Kind string `cborgen:"kind"` // "sigv4" or "sigv4a"
  Data []byte `cborgen:"data"`
}
```
</details>

e.g.

```jsonc
// Encoded as dag-json for readability
{
  "bucket": "did:key:z6MkmNBgCewjYfEDTdKLpHkbMWUogJk29CxmiVdLeW4Kz3UG",
  "permissions": {
    // access key → S3 permissions
    "did:key:z6MkjFRxLLGdBqQSLkZbVnuwUFiomK8eGBkPtim9ETvP7vec": [
      "s3:GetObject",
      "s3:PutObject"
    ]
  },
  "keys": {
    // access key → derived signing key(s)
    "did:key:z6MkjFRxLLGdBqQSLkZbVnuwUFiomK8eGBkPtim9ETvP7vec": [{
      "kind": "sigv4a",
      "data": { "/": { "bytes": "AlakovSxZAhU2LJQTBCny+nw5uyTZ8ShwwAKco1ZQzy2" } }
    }]
  },
  "delegations": {
    "bafyreienos3cw7hcga5vwani3pberioe2qscnz5jk2um5jajo4v7bwmvvm": [
      // access key → invocation issuer
      { "/": "bafyreienos3cw7hcga5vwani3pberioe2qscnz5jk2um5jajo4v7bwmvvm" }
    ]
  }
}
```

Where:

* `permissions` is a map of `DID` (access key) → list of assigned S3 permission strings.
* `keys` is a map of `DID` (access key) → `bytes` (derived signing key). Hilt MUST return s derived signing key that matches the signing key kind used in the AWS API request. It MAY return the other key kind.
* `delegations` is a map of `string(CID)` → `[CID]`. Keys are string encoded CID of a delegation whose audience is the invocation issuer. Values are a proof chain of delegation links. This will be only 1 value in the initial implementation but is defined as a list to allow multiple to be included in the future if necessary.

<details>
<summary>Sigv4a derived signing key</summary>

A [Sigv4a derived signing key](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_sigv-create-signed-request.html#derive-signing-key-sigv4a) is an ECDSA P256 key. To encode as a DID, you need to multibase base58 encode the compressed public key bytes and tag with the ECDSA P256 public key codec. e.g.

```go
package main

import (
	"crypto/elliptic"
	"crypto/rand"
	"fmt"
	"github.com/fil-forge/ucantone/did"
	"github.com/fil-forge/ucantone/principal/multiformat"
	"github.com/multiformats/go-multibase"
)

const P256 = 0x1200

func main() {
	_, x, y, _ := elliptic.GenerateKey(elliptic.P256(), rand.Reader)
	b := elliptic.MarshalCompressed(elliptic.P256(), x, y)
	tagged := multiformat.TagWith(P256, b)
	b58key, _ := multibase.Encode(multibase.Base58BTC, tagged)
	id, _ := did.Parse(did.KeyPrefix + b58key)
	fmt.Println(id)
}
```
</details>

#### Authorization steps

When handling an authorization request the following steps are performed:

1. Lookup the secret key corresponding to the `accessKeyId` from the `Authorization` header credential (signed requests) or the `X-Amz-Credential` query parameter (presigned URLs).
1. Derive signing key and verify request signature.
1. Lookup bucket DID (the subject) from its name in the request URL.
1. Verify the access key has correct permission for the requested S3 action.
1. Verify the tenant associated with the access key matches the invocation issuer.
1. Fetch stored delegations for the access key and re-delegate to the invocation issuer.

It is RECOMMENDED to perform some of these steps in parallel and make use of in-memory caches.

### `/s3/bucket/create`

* Issuer: Ingot
* Audience: Hilt
* Subject: Hilt

Creates a bucket and provisions it with Sprue. Returns the bucket DID and any existing delegations that now automatically have access to it.

#### Arguments

**IPLD schema**

```ipldsch
type CreateArguments struct {
  request Request # S3 API request to create a bucket
}
```

<details>
<summary>Go syntax</summary>

```go
type CreateArguments struct {
  Request Request `cborgen:"request"`
}
```
</details>

e.g.

```jsonc
// Encoded as dag-json for readability
{
  "request": {
    "method": "GET",
    "headers": {
      "host": "region.s3.fil.one"
    },
    "url": "https://region.s3.fil.one/bucket/path?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=z6MkjFRxLLGdBqQSLkZbVnuwUFiomK8eGBkPtim9ETvP7vec%2F20260616%2Fauto%2Fs3%2Faws4_request&X-Amz-Date=20260616T091923Z&X-Amz-Expires=86400&X-Amz-Signature=a3eaedd5acb2512ed6ffa19d23f7425f9c7af3ee84ed3ad711eb38c53f3a4112&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=CreateBucket",
  }
}
```

#### Result

This returns the same structure as a call to [`/s3/request/authorize`](#s3requestauthorize).

### `/s3/bucket/delete`

* Issuer: Ingot
* Audience: Hilt
* Subject: Hilt

Deletes an _empty_ bucket by removing it from Hilt's stores and revoking any delegations that allow access to it.

#### Arguments

**IPLD schema**

```ipldsch
type DeleteArguments struct {
  request Request
}
```

<details>
<summary>Go syntax</summary>

```go
type DeleteArguments struct {
  Request Request `cborgen:"request"`
}
```
</details>

#### Result

Successful deletion returns a unit result (`{}`).

### `/s3/bucket/info`

* Issuer: Ingot
* Audience: Hilt
* Subject: Hilt

Retrieves information for a bucket by global name. Returns the bucket DID and a delegation chain from the bucket to the access key DID provided in the arguments.

#### Arguments

**IPLD schema**

```ipldsch
type InfoArguments struct {
  name      String
  accessKey String
}
```

<details>
<summary>Go syntax</summary>

```go
type InfoArguments struct {
  Name      string  `cborgen:"name"`
  AccessKey did.DID `cborgen:"accessKey"`
}
```
</details>

#### Result

**IPLD schema**

```ipldsch
type InfoOK struct {
  id          String               # Bucket DID
  permissions { String: [String] } # S3 permissions for access key
  delegations { String: [Link] }
}
```

<details>
<summary>Go syntax</summary>

```go
type InfoOK struct {
  ID          did.DID               `cborgen:"id"`
  Permissions map[did.DID][]string  `cborgen:"permissions"`
  Delegations map[cid.Cid][]cid.Cid `cborgen:"delegations"`
}
```
</details>

e.g.

```jsonc
// Encoded as dag-json for readability
{
  "id": "did:key:z6MkmNBgCewjYfEDTdKLpHkbMWUogJk29CxmiVdLeW4Kz3UG",
  "permissions": {
    // access key → S3 permissions
    "did:key:z6MkjFRxLLGdBqQSLkZbVnuwUFiomK8eGBkPtim9ETvP7vec": [
      "s3:GetObject",
      "s3:PutObject"
    ]
  },
  "delegations": {
    "bafyreigngbemvzgbmelwddwoms2ak2g32vmhcpxg6dqlwvb6spiezoc4py": [
      // Root delegation:
      // space → tenant
      { "/": "bafyreiabuvg5hkupzjfn2kqywbdp5xhsb25pmhviyfz77yxhspssvxsv5y" },
      // Intermediate delegation:
      // tenant → access key
      { "/": "bafyreigngbemvzgbmelwddwoms2ak2g32vmhcpxg6dqlwvb6spiezoc4py" },
    ]
  }
}
```

Where:

* `id` is the DID of the bucket.
* `permissions` is a map of `DID` (access key) → list of assigned S3 permission strings.
* `delegations` is a map of `string(CID)` → `[CID]`. Keys are string encoded CID of a delegation whose audience is the access key DID. Values are a proof chain of delegation links from the bucket to the access key.


### `/s3/bucket/list`

* Issuer: Ingot
* Audience: Hilt
* Subject: Hilt

Lists buckets for the tenant.

#### Arguments

**IPLD schema**

```ipldsch
type ListArguments struct {
  request Request
}
```

<details>
<summary>Go syntax</summary>

```go
type ListArguments struct {
  Request Request `cborgen:"request"`
}
```
</details>

#### Result

**IPLD schema**

```ipldsch
type ListOK struct {
  buckets           [Bucket]
  continuationToken String
  owner             Owner
  prefix            String
}

type Bucket struct {
  arn          String
  region       String
  creationDate String
  name         String
}

type Owner struct {
  displayName String
  id          String
}
```

<details>
<summary>Go syntax</summary>

```go
type ListOK struct {
  Buckets           []Bucket `cborgen:"buckets"`
  ContinuationToken string   `cborgen:"continuationToken"`
  Owner             Owner    `cborgen:"owner"`
  Prefix            string   `cborgen:"prefix"`
}

type Bucket struct {
  ARN          string `cborgen:"arn"`
  Region       string `cborgen:"region"`
  CreationDate string `cborgen:"creationDate"`
  Name         string `cborgen:"name"`
}

type Owner struct {
  DisplayName string `cborgen:"displayName"`
  ID          string `cborgen:"id"`
}
```
</details>


## Ingot - S3 API

Ingot is an S3 facade co-located with a Forge Piri node.

### Bucket operations

Bucket operations are forwarded onto Hilt as [`/s3/bucket/create`](#s3bucketcreate), [`/s3/bucket/delete`](#s3bucketdelete) and [`/s3/bucket/list`](#s3bucketlist) invocations.

### Object operations

All object operations are handled by Ingot, after request authorization via local cache or via Hilt.


## Alternatives considered

### Bucket delegations via `/access/delegate`

We talked about Ingot creating bucket keys, delegating [top](https://github.com/ucan-wg/spec#-aka-top) (`/`) access to the tenant and storing them in the upload service with `/access/delegate`.

This is not possible for the reason that `/access/delegate` requires the subject (space/bucket) to have been provisioned by the tenant - something that Ingot is not authorized to do (it never sees the tenant private key so cannot sign a `/provider/add` invocation).

Pre-authorizing Ingot to `/provider/add` raises additional concerns:

* It's not clear how this delegation is communicated to Ingot. It is on-demand or a one time thing at setup?
    * For on-demand we'd have to create an API for that on Hilt, have Ingot call it to retrieve the delegation and provide some params to verify the request is legit (comes from a valid access key).
    * If it's one-time at setup then we have to build an setup step that calls the delegator(?) or rely on engineers to manually issue a delegation and have the operator add it to config.
    * Also, a one-time at setup delegation would have to be allowed to provision _any_ space, since we don't know ahead of time what the DID is. We should limit trust where possible.
* How does Hilt know when to `/access/claim`? I assume on the first read/write after bucket creation?
* How does Hilt remove the delegation from Sprue when a bucket is deleted? There's currently no facility for this in Sprue.

### Alternative for tracking IAM role based permissions

We considered modelling S3 IAM role based permissions as UCAN delegations. The AWS "resource" is the UCAN subject, the "action" is the UCAN command, and the "condition" is the UCAN policy. This would have allowed us to use existing machinery to validate an access key is permitted to perform an action.

Permissions are mapped to commands by lowercasing, replacing ":" with "/" and prefixing with "/". e.g. `s3:GetObject` becomes `/s3/getobject`. This makes permissions compliant with the rules for [command segment structure](https://github.com/ucan-wg/spec#segment-structure).

The combination of S3 action delegations and Forge delegations allows Ingot to determine if an access key is authorized to perform an S3 action, and map the S3 action to an authorized invocation into the Forge network.

This MAY have enabled Ingot to accept UCAN invocations for these delegations in the future.


## Resources

Some resources that were used in the making of this document:

* Canonical request https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_sigv-create-signed-request.html#create-canonical-request
* Sigv4a signing examples https://github.com/aws-samples/sigv4a-signing-examples


## Appendix

### Example Hilt Authorization Flows

Examples for authorization flow and state in Hilt. Data structures are defined in order first encountered.

#### Create a Regional Provider

##### 1. Hilt

A engineer managed table of Ingot instances by DID.

```sql
CREATE TABLE provider (
    id         TEXT PRIMARY KEY, -- DID
    region     TEXT, -- e.g. us-east-1
    created_at TIMESTAMPZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPZ
);
```

e.g.

```sql
INSERT INTO provider (id, region) VALUES ('provider DID', 'us-east-1');
```

#### Create a Tenant

##### 1. Fil One

`PUT /tenant/{tenantId}` to Hilt.

See [OpenAPI spec](https://github.com/fil-one/fil-one/blob/d11bdb4aca27d1177156200436cb3c05857234f4/docs/service-orchestrator-integration/management-openapi.yaml#L54) for method.

##### 2. Hilt

Generate and store private key in KMS:

`/tenant/{id}` → bytes (private key)

Store tenant info in DB:

```sql
CREATE TABLE tenant (
    id          TEXT PRIMARY KEY, -- DID
    external_id TEXT UNIQUE,      -- external Tenant API id ({tenantId})
    provider_id TEXT REFERENCES provider(id),
    name        TEXT,
    status      TEXT NOT NULL, -- active, write-locked, disabled
    created_at  TIMESTAMPZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPZ
);
```

#### Create an Access Key

##### 1. Fil One

`POST /tenants/{tenantId}/access-keys` to Hilt.

See [OpenAPI spec](https://github.com/fil-one/fil-one/blob/d11bdb4aca27d1177156200436cb3c05857234f4/docs/service-orchestrator-integration/management-openapi.yaml#L191) for method.

##### 2. Hilt

The following verifications are performed:

* The bucket(s) in the request exist.
* The bucket(s) belong to the tenant in question.
* The permissions are all in the allowed set of AWS S3 permissions that may be requested.

Generate and store access key in KMS:

`/tenant/{tenantId}/access/{accessKeyId}` → bytes (private key)

Store access key details in DB:

```sql
CREATE TABLE access_key (
    id          TEXT PRIMARY KEY, -- DID
    tenant_id   TEXT REFERENCES tenant(id),
    name        TEXT,
    buckets     TEXT ARRAY,
    permissions TEXT ARRAY NOT NULL,
    created_at  TIMESTAMPZ NOT NULL DEFAULT NOW(),
    expires_at  TIMESTAMPZ
);
```

Get tenant private key.

Create UCAN delegations, signed with the tenant key:

```jsonc
{
  "iss": "tenant DID",
  "aud": "access key DID",
  "sub": null, // OR per bucket in request
  "cmd": "...", // per command mapped from permissions
}
```

Store in DB:

```sql
CREATE TABLE delegation (
    id         TEXT PRIMARY KEY, -- CID of delegation
    audience   TEXT NOT NULL,    -- DID
    subject    TEXT,             -- DID
    command    TEXT,
    data       BYTEA NOT NULL,
    expires_at TIMESTAMPZ
);

CREATE INDEX aud_sub_cmd_idx ON delegation (audience, subject, command);
```

#### Create an Bucket

##### 1. Any Client

Could be Fil One, but also generic S3 client with an access key.

`PUT /{bucket}` to Ingot.

##### 2. Ingot

Invoke `/s3/bucket/create` on Hilt. e.g.

```jsonc
{
  "iss": "ingot DID",
  "aud": "hilt DID",
  "sub": "hilt DID",
  "cmd": "/s3/bucket/create",
  "args": {
    "request": {
      "method": "PUT",
      "url": "/bucket",
      "headers": { /* ... */ }
    }
  }
}
```

Invocation arguments schema:

```ipldsch
type CreateArguments struct {
  request Request # S3 API request to create a bucket
}

# AWS API request
type Request struct {
  method  String
  headers { String: [String] }
  url     String
  # TODO: needs hashed payload?
}
```

##### 3. Hilt

Hilt unpacks the access key ID from the AWS API request, and verifies the signature.

The access key details are fetched from the DB and the following verifications are performed:

* The `s3:CreateBucket` action is present in the authorized `permissions` for the key.
* The access key `tenant_id` matches the invocation issuer DID.

Hilt generates a bucket key.

Store bucket details in DB:

```sql
CREATE TABLE bucket (
    id         TEXT PRIMARY KEY, -- DID
    tenant_id  TEXT REFERENCES tenant(id),
    name       TEXT UNIQUE,
    created_at TIMESTAMPZ NOT NULL DEFAULT NOW()
);
```

Create UCAN delegation to tenant, sign with bucket key:

```jsonc
{
  "iss": "bucket DID",
  "aud": "tenant DID",
  "sub": "bucket DID",
  "cmd": "/" // "top"
}
```

Store in `delegation` DB.

Discard the bucket private key.

Delegations from the bucket DID to the access key are recursively fetched from the `delegation` table and included in the response.

This returns the same structure as a call to [`/s3/request/authroize`](#s3requestauthorize).

This response should be cached by Ingot in the same way as `/s3/bucket/info` responses, so that it can handle listing buckets for the access key.

#### Put an Object

Also applies to object retrieval and deletion.

##### 1. Any Client

Could be Fil One, but also generic S3 client with an access key.

`PUT /{bucket}/{key}` to Ingot.

##### 2. Ingot

Invoke `/s3/request/authorize` on Hilt.

```jsonc
{
  "iss": "ingot DID",
  "aud": "hilt DID",
  "sub": "hilt DID",
  "cmd": "/s3/request/authorize",
  "args": {
    "request": {
      "method": "PUT",
      "url": "/bucket/key",
      "headers": { /* ... */ }
    }
  }
}
```

Invocation arguments schema:

```ipldsch
type AuthorizeArguments struct {
  request Request # S3 API request to put object
}
```

##### 3. Hilt

Hilt unpacks the access key ID from the AWS API request, and verifies the signature.

The access key details are fetched from the DB and the following verifications are performed:

* The permission for the action in the request is present in the authorized `permissions` for the key.
* The access key `tenant_id` matches the invocation issuer DID.
* The bucket from the request exists, and the access key lists it in `buckets` or `buckets` is `NULL`.

```ipldsch
type AuthorizeOK struct {
  bucket      String               # DID
  permissions { String: [String] } # S3 permission set for the access key
  keys        { String: [VerificationKey] }
  delegations { String: [Link] }
}

type KeyKind enum {
   | sigv4
   | sigv4a
}

type VerificationKey struct {
   kind KeyKind
   data Bytes
}
```

Where:

* `permissions` is a map of `DID` (access key) → list of assigned S3 permission strings.
* `keys` is a map of `DID` (access key) → `bytes` (derived signing key). Hilt MUST return s derived signing key that matches the signing key kind used in the AWS API request. It MAY return the other key kind.
* `delegations` is a map of `string(CID)` → `[CID]`. Keys are string encoded CID of a delegation whose audience is the invocation issuer. Values are a proof chain of delegation links. This will be only 1 value in the initial implementation but is defined as a list to allow multiple to be included in the future if necessary.

Hilt then fetches ALL delegations for the access key. e.g.

```sql
SELECT * FROM delegation WHERE audience = 'access key DID'
```

The access key is fetched from KMS and re-delegations are created from the access key to the invocation issuer (Ingot) for each access key delegation found.

The response only includes _temporary_ (24 hour expiry) delegations from the access key to the invocation issuer. The delegation chain from bucket → tenant → access key must be obtained via a call to `/s3/bucket/info`, which can happen in parallel, but is expressed here in serial.

##### 4. Ingot

Invoke `/s3/bucket/info` on Hilt. e.g.

```jsonc
{
  "iss": "ingot DID",
  "aud": "hilt DID",
  "sub": "hilt DID",
  "cmd": "/s3/bucket/info",
  "args": {
    "name": "bucket name",
    "accessKey": "access key ID"
  }
}
```

Invocation arguments schema:

```ipldsch
type InfoArguments struct {
  name      String
  accessKey String
}
```

##### 5. Hilt

Hilt looks up the DID corresponding to the bucket `name`.

Delegations from the bucket DID to the `accessKey` are recursively fetched from the `delegation` table and included in the response.

```ipldsch
type InfoOK struct {
  id          String               # Bucket DID
  permissions { String: [String] } # S3 permissions for access key
  delegations { String: [Link] }
}
```

Where:

* `id` is the DID of the bucket.
* `permissions` is a map of `DID` (access key) → list of assigned S3 permission strings.
* `delegations` is a map of `string(CID)` → `[CID]`. Keys are string encoded CID of a delegation whose audience is the access key DID. Values are a proof chain of delegation links from the bucket to the access key.

##### 6. Ingot

Ingot now has all the delegations needed to make onward requests to the Forge network.

#### Delete an Access Key

##### 1. Fil One

`DELETE /tenants/{tenantId}/access-keys/{accessKeyId}` to Hilt.

See [OpenAPI spec](https://github.com/fil-one/fil-one/blob/d11bdb4aca27d1177156200436cb3c05857234f4/docs/service-orchestrator-integration/management-openapi.yaml#L245) for method.

##### 2. Hilt

Fetch ALL delegations for the access key. e.g.

```sql
SELECT * FROM delegation WHERE audience = 'access key DID'
```

Using the tenant key, sign UCAN invocations to revoke delegations and send to revocation service.

Remove access key delegations from DB:

```sql
DELETE FROM delegation WHERE audience = 'access key DID';
```

Remove access key from KMS.


#### Delete a Bucket

##### 1. Any Client

Could be Fil One, but also generic S3 client with an access key.

`DELETE /{bucketName}` to Ingot.

##### 2. Ingot

Invoke `/s3/bucket/delete` on Hilt.

```ipldsch
type DeleteArguments struct {
  request Request
}
```

On success, Ingot removes any cached delegations for the bucket.

##### 3. Hilt

Hilt unpacks the access key ID from the AWS API request, and verifies the signature.

The access key details are fetched from the DB and the following verifications are performed:

* The `s3:DeleteBucket` action is present in the authorized `permissions` for the key.
* The access key `tenant_id` matches the invocation issuer DID.

Hilt must then verify the bucket is empty. This involves fetching the tenant key and delegation from the bucket to the tenant and invoking `/blob/list`.

If the bucket is empty, it can be removed.

Hilt _may_ revoke any delegations it has generated from the tenant to existing access keys.

Remove the bucket. e.g.

```sql
DELETE FROM delegation WHERE subject = 'bucket DID';
DELETE FROM bucket WHERE id = 'bucket DID';
```
