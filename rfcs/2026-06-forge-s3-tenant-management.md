# RFC: Forge S3 tenant management

Status: experimental

This document proposes how the Forge S3 layer (referred to as Ingot here) manages tenants and how S3 style access credentials are created and mapped to UCAN authorizations. In this context a tenant is roughly analogous to a Fil One _organization_, or an AWS _account_.

It is RECOMMENDED to first read the [Fil One Service Orchestrator API](https://github.com/fil-one/fil-one/blob/main/docs/architectural-decisions/2026-04-service-orchestrator-management-api.md) ADR to gain context on how Fil One manages tenants.


## Tenant creation

Ingot MUST create and store a secret key per tenant. Per tenant keys limit blast radius in case of a security breach.

Tenant keys are used to provision buckets with Sprue - the Forge upload service.

Tenant keys MUST be cryptographic key pairs, they MUST also be `did:plc` keys, allowing them to be rotated if necessary.

After a tenant key has been created, the Ingot service MUST issue a `/account/add` invocation to Sprue to register the account. Note: this is a new capability that does not exist at time of writing. The authority (delegation) needed for Ingot to invoke this capability is out of scope of this document.


## Bucket creation

A bucket MUST be a cryptographic key pair. They SHOULD be ed25519 keys although other key types MAY be used. A bucket maps to a space in Forge.

After a bucket is created, a delegation to the tenant MUST be issued and stored. The delegation MUST grant the tenant full authority over the bucket (issuer = bucket, audience = tenant, subject = bucket, command = "/"). See [top](https://github.com/ucan-wg/spec#-aka-top).

Ingot MUST then issue a `/provider/add` invocation to Sprue to register the bucket/space and assert the bucket's ownership by the tenant.

If a bucket is created _after_ an access key has been created, a root delegation will be issued and stored for each powerline access key owned by the tenant. See [bucket access](#bucket-access).

### Bucket name mapping

Ingot MUST maintain a mapping of bucket names to identifiers (DIDs) so that requests with buckets names in the URL can be mapped to bucket identifiers for authorization (see [access key creation](#access-key-creation)).


## Access key creation

Ingot MUST create and store a secret key per S3 access key.

Access keys MUST be cryptographic keys. They SHOULD be ed25519 keys, where the `accessKeyId` is the `did:key` (public key) and the `secretAccessKey` is the 32 byte ed25519 private key, prefixed with multiformat varint for ed25519 (`0x1300`) and encoded using multibase base64pad.

You might generate this in ucantone with the following code:

```go
package main

import (
	"fmt"
	"github.com/fil-forge/ucantone/principal/ed25519"
)

func main() {
	sk, _ := ed25519.Generate()
	fmt.Printf("accessKeyId: %s\n", sk.DID())
	fmt.Printf("secretAccessKey: %s\n", ed25519.Format(sk))
}
```

Output:

```sh
accessKeyId: did:key:z6Mkmsr9AESUCv2XJWjfKpLBediyHP8p9yUFE9BmhGLfhFR9
secretAccessKey: MgCZdkM3wKlJc0T8muCVH5lQVSmIPQGWQo+gP5vTuD3l+DA==
```

**S3 IAM role based permissions are modeled as UCAN delegations**, where the AWS "resource" is the UCAN subject, the "action" is the UCAN command, and the "condition" is the UCAN policy. This allows us to use existing machinery to validate an access key is permitted to perform an action.

Actions are mapped to commands by lowercasing, replacing ":" with "/" and prefixing with "/". e.g. `s3:GetObject` becomes `/s3/getobject`. This makes actions compliant with the rules for [command segment structure](https://github.com/ucan-wg/spec#segment-structure).

In the future, Ingot MAY accept UCAN invocations for these delegations.

Currently Fil One supports the following S3 actions:

Bucket-level:

- `s3:CreateBucket`, `s3:ListAllMyBuckets`, `s3:DeleteBucket`.

Object-level:

- `s3:GetObject`, `s3:GetObjectVersion`, `s3:GetObjectRetention`, `s3:GetObjectLegalHold`
- `s3:PutObject`, `s3:PutObjectRetention`, `s3:PutObjectLegalHold`
- `s3:ListBucket`, `s3:ListBucketVersions`
- `s3:DeleteObject`, `s3:DeleteObjectVersion`

Note: there is an AWS quirk where `s3:ListBucket` lists _objects_ in a bucket, while `s3:ListAllMyBuckets` lists buckets. Similarly, `s3:ListBucketVersions` allows listing _object_ versions in a bucket.

Given a tenant key `did:key:tenant`, a bucket `did:key:bucket`, and an access key `did:key:access`, granting the `s3:GetObject` and `s3:PutObject` actions involves creating and storing the following 2 UCAN delegations:

```json
{
  "iss": "did:key:tenant",
  "aud": "did:key:access",
  "sub": "did:key:bucket",
  "cmd": "/s3/getobject",
  "pol": [],
  // ...
}
```

```json
{
  "iss": "did:key:tenant",
  "aud": "did:key:access",
  "sub": "did:key:bucket",
  "cmd": "/s3/putobject",
  "pol": [],
  // ...
}
```

Note: for multiple buckets, multiple delegations are created. Powerline delegations MAY be created to allow access to _any_ bucket existing or future (see [bucket access](#bucket-access)).

Delegations MUST be indexed by audience and MAY be indexed by subject and command as well. This allows efficient lookup when validating a request and facilitates removal when an access key is deleted.

In addition to the S3 permissions, Forge related permissions are also delegated from the tenant to the access key, depending on what S3 permissions are selected:

| S3 Action         | Forge Capability                         |
| ----------------- | ---------------------------------------- |
| `s3:GetObject`    | `/content/retrieve`                      |
| `s3:PutObject`    | `/blob/add`, `/index/add`, `/upload/add` |
| `s3:DeleteObject` | `/blob/remove`, `/upload/remove`         |
| `s3:ListBucket`   | `/blob/list`, `/upload/list`             |

Onward invocations made by Ingot to the Forge network MUST use the access key to sign, NOT the tenant key. This is for the purpose of auditing and accounting.

### Bucket access

It's typical to allow the access key to access any bucket created by the tenant that exists at the time of creation as well as any bucket that may exist at any time in the future. This is modeled as a "powerline" delegation.

If a bucket is created _after_ an access key has been created, a root delegation will be issued and stored (where issuer = tenant, subject = bucket and audience = access key) to allow validation to succeed ([powerline cannot be the root delegation](https://github.com/ucan-wg/delegation#powerline)).

### Key expiry

Key expiry is simply a UCAN expiration time set on the delegations created for the key.


## Authorization steps

When receiving an S3 API request the following authorization steps are performed:

1. Lookup the secret key corresponding to the `accessKeyId` from `x-amz-credential` header.
1. Verify SigV4 signature.
1. Lookup bucket DID (the subject) from its name in the request URL.
1. Find proofs necessary to authorize a request to the specific endpoint.
    * Example: for `PUT /bucket/:key` (`s3:PutObject`) it's necessary to find a proof chain ending with a delegation whose audience = access key, subject = bucket and command = `/s3/putobject`.
1. Validate the proof chain.
1. Optionally, if a request into the Forge network is necessary:
    1. Lookup delegations necessary to perform action(s) within the Forge network.
    1. Include as proofs for onward invocations signed by access key.

It is RECOMMENDED to perform some of these steps in parallel and make use of in-memory caches.
