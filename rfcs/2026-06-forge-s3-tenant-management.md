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


## Hilt - an S3 tenant management service

A trusted centralized service for tenant management exists so that storing secrets is not a requirement for Ingot deployments. Hilt implements the [Tenant API](https://github.com/fil-one/fil-one/blob/main/docs/architectural-decisions/2026-04-service-orchestrator-management-api.md), provides a UCAN API for retrieving proof chains for invocations into the Forge network and speaks to the Forge upload service.

### Tenant API

Hilt MUST be configured with a pre-shared bearer key allowing only the **Fil One service** to call its [Tenant API](https://github.com/fil-one/fil-one/blob/main/docs/architectural-decisions/2026-04-service-orchestrator-management-api.md).

The tenant API enables "tenants" and S3 style access keys to be created. Both concepts correspond to private keys and are stored securely by Hilt.

#### Tenant creation

Hilt MUST create and store a secret key per tenant. Per tenant keys limit blast radius in case of a security breach.

Tenant keys are used to provision buckets with Sprue - the Forge upload service.

Tenant keys MUST be cryptographic key pairs, they MUST also be `did:plc` keys, allowing them to be rotated if necessary.

After a tenant key has been created, the Ingot service MUST issue a `/account/add` invocation to Sprue to register the account. Note: this is a new capability that does not exist at time of writing. The means of obtaining authority (delegation) needed for Ingot to invoke this capability is out of scope of this document.

#### Bucket creation

A bucket MUST be a cryptographic key pair. They SHOULD be ed25519 keys although other key types MAY be used. A bucket is a space in Forge.

After a bucket is created via the tenant API, a delegation to the tenant MUST be issued and stored. The delegation MUST grant the tenant full authority over the bucket (issuer = bucket, audience = tenant, subject = bucket, command = "/"). See [top](https://github.com/ucan-wg/spec#-aka-top). e.g.

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

Hilt MUST then issue a `/provider/add` invocation to Sprue to register the bucket/space and assert the bucket's ownership by the tenant.

##### Bucket name mapping

Hilt MUST maintain a mapping of bucket names to identifiers (DIDs) so that requests with bucket names in the URL can be mapped to bucket identifiers for authorization (see [access key creation](#access-key-creation)).

#### Access key creation

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

Currently Fil One supports the following S3 actions:

Object-level:

- `s3:GetObject`, `s3:GetObjectVersion`, `s3:GetObjectRetention`, `s3:GetObjectLegalHold`
- `s3:PutObject`, `s3:PutObjectRetention`, `s3:PutObjectLegalHold`
- `s3:DeleteObject`, `s3:DeleteObjectVersion`
- `s3:ListBucket`, `s3:ListBucketVersions`

Bucket-level:

- `s3:CreateBucket`, `s3:ListAllMyBuckets`, `s3:DeleteBucket`

Note: there is an AWS quirk where `s3:ListBucket` lists _objects_ in a bucket, while `s3:ListAllMyBuckets` lists buckets. Similarly, `s3:ListBucketVersions` allows listing _object_ versions in a bucket.

When an access key is created, Forge related permissions are delegated from the _tenant_ to the _access key_, depending on what S3 permissions are selected:

| S3 Action                | Forge Command                            |
| ------------------------ | ---------------------------------------- |
| `s3:GetObject`           | `/content/retrieve`                      |
| `s3:GetObjectVersion`    | `/content/retrieve`                      |
| `s3:GetObjectRetention`  | `/content/retrieve`                      |
| `s3:GetObjectLegalHold`  | `/content/retrieve`                      |
| `s3:PutObject`           | `/blob/add`, `/index/add`, `/upload/add` |
| `s3:PutObjectRetention`  | `/blob/add`, `/index/add`, `/upload/add` |
| `s3:PutObjectLegalHold`  | `/blob/add`, `/index/add`, `/upload/add` |
| `s3:DeleteObject`        | `/blob/remove`, `/upload/remove`         |
| `s3:DeleteObjectVersion` | `/blob/remove`, `/upload/remove`         |
| `s3:ListBucket`          | `/blob/list`, `/upload/list`             |
| `s3:ListBucketVersions`  | `/content/retrieve`                      |
| `s3:CreateBucket`        | n/a                                      |
| `s3:ListAllMyBuckets`    | n/a                                      |
| `s3:DeleteBucket`        | n/a                                      |

Observe that in the table above, multiple actions map to the same forge command and some actions do not have an equivalent Forge command. Hilt and Ingot MUST be able to determine if an access key is authorized to perform the S3 action and cannot do this by simply receiving Forge delegations alone.

To facilitate authorization **S3 IAM role based permissions are modeled as UCAN delegations**, where the AWS "resource" is the UCAN subject, the "action" is the UCAN command, and the "condition" is the UCAN policy. This allows us to use existing machinery to validate an access key is permitted to perform an action.

Actions are mapped to commands by lowercasing, replacing ":" with "/" and prefixing with "/". e.g. `s3:GetObject` becomes `/s3/getobject`. This makes actions compliant with the rules for [command segment structure](https://github.com/ucan-wg/spec#segment-structure).

The combination of S3 action delegations and Forge delegations allows Ingot to determine if an access key is authorized to perform an S3 action, and map the S3 action to an authorized invocation into the Forge network.

In the future this MAY enable Ingot to accept UCAN invocations for these delegations.

Given a tenant key `did:plc:tenant`, a bucket `did:key:bucket`, and an access key `did:key:access`, granting the `s3:GetObject` and `s3:PutObject` actions involves creating and storing the following 2 UCAN delegations:

```jsonc
{
  "iss": "did:plc:tenant",
  "aud": "did:key:access",
  "sub": "did:key:bucket",
  "cmd": "/s3/getobject",
  "pol": [],
  // ...
}
```

```jsonc
{
  "iss": "did:plc:tenant",
  "aud": "did:key:access",
  "sub": "did:key:bucket",
  "cmd": "/s3/putobject",
  "pol": [],
  // ...
}
```

...and the following 4 Forge delegations:

```jsonc
{
  "iss": "did:plc:tenant",
  "aud": "did:key:access",
  "sub": "did:key:bucket",
  "cmd": "/content/retrieve",
  "pol": [],
  // ...
}
```

```jsonc
{
  "iss": "did:plc:tenant",
  "aud": "did:key:access",
  "sub": "did:key:bucket",
  "cmd": "/blob/add",
  "pol": [],
  // ...
}
```

```jsonc
{
  "iss": "did:plc:tenant",
  "aud": "did:key:access",
  "sub": "did:key:bucket",
  "cmd": "/index/add",
  "pol": [],
  // ...
}
```

```jsonc
{
  "iss": "did:plc:tenant",
  "aud": "did:key:access",
  "sub": "did:key:bucket",
  "cmd": "/upload/add",
  "pol": [],
  // ...
}
```

Delegations MUST be indexed by audience and MAY be indexed by subject and command as well. This allows efficient lookup when validating a request and facilitates removal when an access key is deleted.

Note: for multiple buckets, multiple delegations are created. Powerline delegations MAY be created to allow access to _any_ bucket existing or future (see [bucket access](#bucket-access)).

### Bucket access

It's typical to allow the access key to access any bucket created by the tenant that exists at the time of creation as well as any bucket that may exist at any time in the future. This is modeled as a "powerline" delegation, where the UCAN subject is `null`.

### Key expiry

Key expiry is simply a UCAN expiration time set on the delegations created for the key.

### UCAN API

Hilt is configured with a list of Ingot service DIDs that are allowed to call its UCAN API. It provides a single UCAN command that may be invoked:

#### `/aws/request/authorize`

Issuer: Ingot
Audience: Hilt
Subject: Hilt

Authorizes Sigv4 signed AWS API requests. Given the incoming request and Sigv4 signature, this handler:

1. Determines validity of the signature.
1. Finds stored delegations for the access key.
1. Derives a new Sigv4a signing key from the access key's private key.
1. Re-delegates capabilities to the derived signing key.
1. Returns the derived signing key and delegations.

##### Arguments

**IPLD schema**

```ipldsch
type AuthorizeArgs struct {
  Request Request
  Sigv4   Bytes
}

type Request struct {
  Method  String
  Headers { String: [String] }
  URL     String
}
```

<details>
<summary>Go syntax</summary>

```go
type AuthorizeArgs struct {
  Request Request `cborgen:"request"`
  Sigv4   []byte  `cborgen:"sigv4"`
}

type Request struct {
  Method  string              `cborgen:"method"`
  Headers map[string][]string `cborgen:"headers"`
  URL     string              `cborgen:"url"`
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

##### Result

A successful authorization will return a Sigv4a derived private key that can be used to sign requests, and a set of delegations for that key that allows access to the Forge network.

**IPLD schema**

```ipldsch
type AuthorizeOK struct {
  keys        { String: Bytes }
  delegations { String: [Link] }
}
```

<details>
<summary>Go syntax</summary>

```go
type AuthorizeOK struct {
  Keys        map[string][]byte    `cborgen:"keys"`
  Delegations map[string][]cid.Cid `cborgen:"delegations"`
}
```
</details>

e.g.

```jsonc
// Encoded as dag-json for readability
{
  "keys": {
    "did:key:zDnaeWGBiLkmSn4kA3UDNM3SjCKo1nn5qhP2fsNe1iLTD7mFj": {
      "/": { "bytes": "AlakovSxZAhU2LJQTBCny+nw5uyTZ8ShwwAKco1ZQzy2" }
    }
  },
  "delegations": {
    "bafyreienos3cw7hcga5vwani3pberioe2qscnz5jk2um5jajo4v7bwmvvm": [
      // Root delegation:
      // space → tenant
      { "/": "bafyreiabuvg5hkupzjfn2kqywbdp5xhsb25pmhviyfz77yxhspssvxsv5y" },
      // Intermediate delegation:
      // tenant → access key
      { "/": "bafyreigngbemvzgbmelwddwoms2ak2g32vmhcpxg6dqlwvb6spiezoc4py" },
      // Leaf delegation:
      // access key → derived key (did:key:zDnaeWGBiLkmSn4kA3UDNM3SjCKo1nn5qhP2fsNe1iLTD7mFj)
      { "/": "bafyreienos3cw7hcga5vwani3pberioe2qscnz5jk2um5jajo4v7bwmvvm" }
    ]
  }
}
```

`keys` is a map of `DID` → `bytes` (private key).

`delegations` is a map of `string(CID)` → `[CID]`. Keys are string encoded CID of a delegation whose audience is the key in `keys`. Values are the proof chain required to make an invocation (strictly in the order required for invocation).

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

##### Authorization steps

When handling an authorization request the following steps are performed:

1. Lookup the secret key corresponding to the `accessKeyId` from the `Authorization` header credential (signed requests) or the `X-Amz-Credential` query parameter (presigned URLs).
1. Verify SigV4 signature.
1. Lookup bucket DID (the subject) from its name in the request URL.
1. Find proofs necessary to authorize a request to the specific endpoint.
    * Example: for `PUT /:bucket/:key` (`s3:PutObject`) it's necessary to find a proof chain ending with a delegation whose audience = access key, subject = bucket and command = `/s3/putobject`.
1. Validate the proof chain.
1. Optionally, if a request into the Forge network is necessary:
    1. Lookup delegations necessary to perform action(s) within the Forge network.
    1. Include as proofs for onward invocations signed by access key.

It is RECOMMENDED to perform some of these steps in parallel and make use of in-memory caches.


## Ingot - an S3 facade co-located with a Forge Piri node

### S3 API

#### Bucket operations

In the future this section will document how Ingot will perform bucket level operations. Tentatively a `/aws/request/authorize` invocation is enough to action bucket level operations on Hilt with an authorized API key.


## Alternatives considered

### Bucket delegations via `/access/delegate`

We talked about Ingot creating bucket keys, delegating [top](https://github.com/ucan-wg/spec#-aka-top) (`/`) access to the tenant and storing them in the upload service with `/access/delegate`.

This is not possible for the reason that `/access/delegate` requires the subject (space/bucket) to have been provisioned by the tenant - something that Ingot is not authorized to do (it never sees the tenant private key so cannot sign a `/provider/add` invocation).

### Alternative for tracking IAM role based permissions

An alternative to modeling S3 permissions as UCAN delegations would be to have a separate store that maps access keys to their S3 permission set. The `/aws/request/authorize` invocation would need to return this list in the response, so that Hilt can determine whether the access key is permitted to perform the requested S3 action.

## Resources

Some resources that were used in the making of this document:

* Canonical request https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_sigv-create-signed-request.html#create-canonical-request
* Sigv4a signing examples https://github.com/aws-samples/sigv4a-signing-examples
