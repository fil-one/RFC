# RFC: Trailer Hash

Status: Experimental

## Authors

- [Alan Shaw](https://github.com/alanshaw)

## Motivation

This RFC proposes an extension to the blob protocol that allows a content hash to be agreed on _after_ delivery of the data.

Currently the blob protocol requires the sender to know the hash of the data _before_ it is delivered to the storage node. This makes it difficult to proxy non-hashed data to the network without ingesting and storing the data in its entirety before forwarding it on. It effectively doubles the time it takes to upload the data and requires the proxy to have significant storage space available that must persist until the blob is offloaded.

This RFC proposes a method where the hash of the data can be agreed on by both parties _after_ delivery by agreeing only on the hashing algorithm beforehand. Provided the size of the data is known, it allows data to be streamed through the proxy to the network without having to store and forward.

## Design

### Changes to `/blob/add`

In the invocation arguments, `blob.digest` becomes optional, and an optional `blob.digestCode` field is added: the [multicodec](https://github.com/multiformats/multicodec/blob/master/table.csv) code of the [multihash](https://github.com/multiformats/multihash) function that should be used to hash the data, which is the code the computed digest will carry.

Exactly one of `blob.digest` or `blob.digestCode` MUST be provided. A multihash already carries its code, so a digest needs no `digestCode` alongside it.

When `blob.digestCode` is provided, the invocation MUST contain a unique _nonce_: two or more blobs of the same size may be added with the same digest code, and the `/http/put` principal is derived from the task link (see [Changes to `/http/put`](#changes-to-httpput)), so the task link must be unique to the upload.

The invocation MUST fail if the executor does not support adding a blob by digest code, or does not support the specified digest code, so that the client can fall back to computing the digest before adding the blob. Implementations MUST support SHA2-256.

### Changes to `/blob/allocate`

As above, the invocation argument `blob.digest` becomes optional, and an optional `blob.digestCode` field is added. Exactly one of `blob.digest` or `blob.digestCode` MUST be provided and MUST be set to the value from the `/blob/add` invocation arguments.

The invocation MUST fail if the storage node does not support the specified digest code. Implementations MUST support SHA2-256. The upload service SHOULD allocate on a storage node that supports the digest code, and SHOULD try another candidate rather than fail the `/blob/add` when one does not.

Since the blob digest is unknown, a successful receipt MUST always contain a size field that is equal to the size of the blob and MUST always contain an address field. The storage node cannot recognise content it already holds, so the data is always transferred, even when the storage node already has it.

The size is the only property of the data that is known in advance, so when the allocation does not specify a digest the address MUST reject a `PUT` whose body length differs from `blob.size`.

### Changes to `/http/put`

The invocation argument `body.digest` becomes optional, and an optional `body.digestCode` field is added. Each MUST be set to its value in the `/blob/allocate` invocation arguments, and omitted when those arguments omit it.

The subject of the invocation is normally derived from the blob digest. When no digest is specified, the subject MUST instead be the [`did:key`] of the Ed25519 key whose seed is the last 32 bytes of the multihash of the `/blob/add` task link. As before, the key is embedded in the invocation `meta` field, is a public, single-purpose token, and MUST NOT be granted any other authority.

A successful receipt gains an optional `blob.digest` field - the computed multihash digest of the data. It MUST be set if the corresponding `/blob/allocate` task did not specify a digest in its arguments.

The client that sends the data MUST compute the digest per the agreed algorithm as the data is sent, and it is this digest that the client reports in the receipt.

The storage node that receives the data MUST compute the digest per the agreed algorithm as the data is received. The computed digest MUST be stored against the allocation so that it can be verified when the blob is accepted.

### Changes to `/blob/accept`

As above, the invocation argument `blob.digest` becomes optional, and an optional `blob.digestCode` field is added. Each MUST be set to its value in the `/blob/allocate` invocation arguments, and omitted when those arguments omit it.

When the corresponding `/blob/allocate` task did not specify a digest, the `/blob/add` executor cannot know the digest when it creates the `/blob/accept` task, so the digest is instead taken from the result the `_put` promise resolves to: the `blob.digest` field of the `/http/put` receipt. In this case the `/http/put` receipt MUST be transmitted in the container with the `/blob/accept` invocation.

A `/blob/accept` invocation MUST fail with the error name `BlobDigestMismatch` if the digest computed by the storage node for the received data does not match the digest in the invocation arguments or, when the arguments do not specify one, the digest in the `/http/put` receipt. It MUST also fail if the size of the received data differs from `blob.size`.

## End-to-end example

The client in this example is an S3 gateway (`did:web:s3.example.com`) proxying an S3 `PUT`: it adds a 2MiB blob to Alice's space without knowing its digest. The gateway knows the size in advance: it follows from the object's `Content-Length` and the encryption overhead. Both the client and the storage node hash the bytes as they stream, and `/blob/accept` succeeds only if the two digests match.

Examples follow the conventions of the [blob protocol] specification: they show invocation payloads with the signed envelope elided, `// "/": "bafy.."` comments denote task links, and receipts show only their salient fields. Values are [DAG-JSON]: bytes are unpadded standard base64 with no multibase prefix, and links and bytes are elided with `...`. The digest code `18` is the multicodec code for SHA2-256 (`0x12`).

### 1. Client invokes `/blob/add`

The invocation names the hashing algorithm instead of a digest. Its nonce makes the task unique, since other blobs of the same size and digest code can be added the same way.

```jsonc
{ // "/": "bafy..add"
  "iss": "did:web:s3.example.com",
  "aud": "did:web:upload.example.com",
  "sub": "did:key:zAliceSpace",
  "cmd": "/blob/add",
  "args": {
    "blob": {
      // multihash function code for the digest to come (sha2-256)
      "digestCode": 18,
      "size": 2097152
    }
  },
  "prf": [{ "/": "bafy..dlgAliceSpace" }],
  "nonce": { "/": { "bytes": "dHJhaWxlcg" } },
  "exp": 1735689600
}
```

### 2. Upload service invokes `/blob/allocate`

```jsonc
{ // "/": "bafy..alloc"
  "iss": "did:web:upload.example.com",
  "aud": "did:key:zStorageNode",
  "sub": "did:key:zStorageNode",
  "cmd": "/blob/allocate",
  "args": {
    "space": "did:key:zAliceSpace",
    "blob": {
      "digestCode": 18,
      "size": 2097152
    },
    "cause": { "/": "bafy..add" }
  },
  "prf": [{ "/": "bafy..dlgStorageNode" }],
  "nonce": { "/": { "bytes": "YWxsb2NhdGU" } },
  "exp": 1735689600
}
```

The storage node cannot tell whether it already holds the content, so the receipt always carries the full size and an upload address.

```jsonc
{
  "iss": "did:key:zStorageNode",
  "aud": "did:web:upload.example.com",
  "sub": "did:key:zStorageNode",
  "cmd": "/ucan/assert/receipt",
  "args": {
    "ran": { "/": "bafy..alloc" },
    "out": {
      "ok": {
        "size": 2097152,
        "address": {
          "url": "https://storage-node.example.com/pdp/piece/upload/6f1c...",
          "headers": {},
          "expires": 1735776000
        }
      }
    }
  },
  "iat": 1735689510
}
```

### 3. Upload service builds the `/http/put` and `/blob/accept` tasks

With no digest available, the upload service derives the `/http/put` principal from the `/blob/add` task link (see [Changes to `/http/put`](#changes-to-httpput)). The client can derive the same key, and the nonce on `/blob/add` makes it unique to this upload.

```jsonc
{ // "/": "bafy..put"
  // Ed25519 key derived from the /blob/add task link
  "iss": "did:key:zAdd...der",
  "aud": "did:key:zAdd...der",
  "sub": "did:key:zAdd...der",
  "cmd": "/http/put",
  "args": {
    "body": {
      "digestCode": 18,
      "size": 2097152
    },
    "destination": { "await/ok": { "/": "bafy..alloc" } }
  },
  "meta": {
    "keys": {
      "id": "did:key:zAdd...der",
      "keys": {
        "did:key:zAdd...der": { "/": { "bytes": "gCY...priv" } }
      }
    }
  },
  "prf": [],
  "nonce": { "/": { "bytes": "cHV0" } },
  "exp": 1735689600
}
```

The `/blob/accept` task keeps its empty nonce, so its link can still be derived from its arguments. The arguments carry no digest; the storage node takes it from the resolved `_put` promise, which is the `/http/put` receipt the client delivers in step 5.

```jsonc
{ // "/": "bafy..accept"
  "iss": "did:web:upload.example.com",
  "aud": "did:key:zStorageNode",
  "sub": "did:key:zStorageNode",
  "cmd": "/blob/accept",
  "args": {
    "space": "did:key:zAliceSpace",
    "blob": {
      "digestCode": 18,
      "size": 2097152
    },
    // resolves to the /http/put result, which carries the digest
    "_put": { "await/ok": { "/": "bafy..put" } }
  },
  "prf": [{ "/": "bafy..dlgStorageNode" }],
  "nonce": { "/": { "bytes": "" } },
  "exp": 1735689600
}
```

### 4. Upload service responds to `/blob/add`

The receipt and its container are unchanged: `site` awaits the accept task, and the container carries the allocate invocation and receipt, and the put and accept invocations.

```jsonc
{
  "iss": "did:web:upload.example.com",
  "aud": "did:web:upload.example.com",
  "sub": "did:web:upload.example.com",
  "cmd": "/ucan/assert/receipt",
  "args": {
    "ran": { "/": "bafy..add" },
    "out": {
      "ok": {
        "site": { "await/ok": { "/": "bafy..accept" } }
      }
    }
  },
  "iat": 1735689500
}
```

### 5. Client streams the data and concludes `/http/put`

The client sends an HTTP `PUT` of exactly 2,097,152 bytes to the allocated URL, hashing the bytes with SHA2-256 as it sends them. The storage node hashes them as it receives them and records its digest against the allocation.

When the `PUT` completes, the client issues the `/http/put` receipt, signed with the derived key. Its result carries the digest the client computed.

```jsonc
{ // "/": "bafy..putReceipt"
  "iss": "did:key:zAdd...der",
  "aud": "did:key:zAdd...der",
  "sub": "did:key:zAdd...der",
  "cmd": "/ucan/assert/receipt",
  "args": {
    "ran": { "/": "bafy..put" },
    "out": {
      "ok": {
        "blob": {
          // digest the client computed while sending
          "digest": { "/": { "bytes": "EiC...9xQw" } }
        }
      }
    }
  },
  "iat": 1735689560
}
```

The client delivers the receipt with `/ucan/conclude`, as it does today.

```jsonc
{
  "iss": "did:web:s3.example.com",
  "aud": "did:web:upload.example.com",
  "sub": "did:web:upload.example.com",
  "cmd": "/ucan/conclude",
  "args": {
    "receipts": [{ "/": "bafy..putReceipt" }]
  },
  "prf": [],
  "nonce": { "/": { "bytes": "Y29uY2x1ZGU" } },
  "exp": 1735689600
}
```

### 6. Storage node accepts the blob

The upload service executes the accept task and sends the `/http/put` receipt in the same container, so the storage node can resolve `_put`. The storage node compares the resolved digest with the one it recorded for the allocation. They match, so it issues a location commitment for the digest and returns it in the accept receipt.

```jsonc
{ // "/": "bafy..site" (invocation link)
  "iss": "did:key:zStorageNode",
  "aud": "did:key:zStorageNode",
  "sub": "did:key:zStorageNode",
  "cmd": "/assert/location",
  "args": {
    "space": "did:key:zAliceSpace",
    // the agreed digest
    "content": { "/": { "bytes": "EiC...9xQw" } },
    "location": ["https://storage-node.example.com/blob/zQm...9xQw"],
    "range": { "start": 0, "end": 2097151 }
  },
  "prf": [],
  "nonce": { "/": { "bytes": "c2l0ZQ" } },
  "exp": null
}
```

```jsonc
{
  "iss": "did:key:zStorageNode",
  "aud": "did:web:upload.example.com",
  "sub": "did:key:zStorageNode",
  "cmd": "/ucan/assert/receipt",
  "args": {
    "ran": { "/": "bafy..accept" },
    "out": {
      "ok": {
        "site": { "/": "bafy..site" },
        "pdp": { "await/ok": { "/": "bafy..pdpAccept" } }
      }
    }
  },
  "iat": 1735689570
}
```

The client reads the accept receipt from the `/ucan/conclude` response container, as it does today, and learns the location of the blob it streamed.

### When the digests differ

If the bytes were corrupted in transit, the storage node's digest differs from the one in the `/http/put` receipt. The accept task fails, no location commitment is issued, and the allocation is released when it expires or when the client aborts it.

```jsonc
{
  "iss": "did:key:zStorageNode",
  "aud": "did:web:upload.example.com",
  "sub": "did:key:zStorageNode",
  "cmd": "/ucan/assert/receipt",
  "args": {
    "ran": { "/": "bafy..accept" },
    "out": {
      "error": {
        "name": "BlobDigestMismatch",
        "message": "received content hashes to EiC...a1Rk, /http/put reports EiC...9xQw"
      }
    }
  },
  "iat": 1735689570
}
```

## Cancelling an upload

An upload is cancelled with `/blob/abort`, which the upload service translates into `/blob/reject` on the storage node (see the [blob removal] RFC). Both identify the upload by `(digest, space)`, but an upload without a digest has none until its data has been received, so the storage node, `/blob/abort` and `/blob/reject` need another way to identify it until then.

### Allocations without a digest

A storage node MUST identify an allocation made without a digest by the link of the `/blob/allocate` task that created it, since the node issued the receipt for that task.

When the data has been received, the storage node MUST also record the allocation against `(computed digest, space)`, so that the checks the [blob removal] RFC requires before physical deletion count it as a claim on the digest.

Data from an upload that never completes has no computed digest and never counts as a claim on any digest. The storage node deletes it when the allocation is rejected or expires.

If the computed digest matches data the storage node already holds, for this space or another, the newly received copy is a duplicate. The storage node MAY delete it once the blob is accepted, because the existing copy satisfies the acceptance.

### Changes to `/blob/abort`

The invocation argument `digest` becomes optional. It MUST be set if the `/blob/add` task linked by `cause` specified a digest.

When `digest` is omitted, `cause` alone identifies the upload. The upload service recovers the `/blob/allocate` task from the `/blob/add` task's receipt chain, and forwards its link to the storage node in `/blob/reject`.

A client MUST NOT set `digest` to a digest it computed for an upload that did not specify one: an upload that failed part way through has no agreed digest, and the digest of the data sent so far identifies nothing.

### Changes to `/blob/reject`

The invocation argument `digest` becomes optional, and an optional `allocation` field is added: the link to the `/blob/allocate` task whose allocation is dropped. Exactly one of `digest` or `allocation` MUST be provided.

When `allocation` is provided, the storage node drops that allocation only. It MUST refuse with `BlobAccepted` only if that allocation was itself accepted. An acceptance of the same computed digest by the same space through a different allocation MUST NOT block the reject, for the same reason the [blob removal] RFC scopes the guard to the invoking space: otherwise one upload's acceptance would strand another upload's allocation until it expires.

The `cause` of the abort is still not forwarded. The `/blob/allocate` task link is forwarded instead because, unlike `cause`, it identifies something the storage node holds.

### Accepted blobs are unaffected

`/blob/remove` and `/blob/release` operate only on accepted blobs, whose digest has been agreed, so they are unchanged. A client MUST record the agreed digest, the `content` of the location commitment, so it can remove the blob later.

The receipt chain the upload service walks to find the storage node is also unchanged: the `/blob/accept` task link exists when `/blob/add` is answered, so `site` can await it.

### Example: aborting an upload without a digest

The client abandons the upload from the end-to-end example above before sending all of its data. It knows the `/blob/add` task link but has no digest.

```jsonc
{
  "iss": "did:web:s3.example.com",
  "aud": "did:web:upload.example.com",
  "sub": "did:key:zAliceSpace",
  "cmd": "/blob/abort",
  "args": {
    // no digest: the /blob/add task did not specify one
    "cause": { "/": "bafy..add" }
  },
  "prf": [{ "/": "bafy..dlgAliceSpace" }],
  "nonce": { "/": { "bytes": "YWJvcnQ" } },
  "exp": 1735689600
}
```

The upload service finds the `/blob/allocate` task and the storage node from the receipt chain of `bafy..add`, and forwards the allocation link:

```jsonc
{
  "iss": "did:web:upload.example.com",
  "aud": "did:key:zStorageNode",
  "sub": "did:key:zStorageNode",
  "cmd": "/blob/reject",
  "args": {
    "space": "did:key:zAliceSpace",
    // identifies the allocation in place of a digest
    "allocation": { "/": "bafy..alloc" }
  },
  "prf": [{ "/": "bafy..dlgStorageNode" }],
  "nonce": { "/": { "bytes": "cmVqZWN0" } },
  "exp": 1735689600
}
```

The storage node drops the allocation and deletes any data it received for it.

## Security considerations

Trust in location commitments is unchanged. The storage node issues the commitment for the digest it computed from the data it received, and accepts only when that digest matches the one the client reports, so a client cannot obtain a commitment for data it did not send.

The `/http/put` key is derived from the `/blob/add` task link, so anyone who holds the task link can derive the key and issue a `/http/put` receipt with a different digest. Such a receipt can only cause the `/blob/accept` task to fail with `BlobDigestMismatch`: it can disrupt that one upload, but cannot cause the storage node to commit to data it does not hold. The same is true today of anyone holding the blob digest.

## Alternatives considered

### Sending the digest as an HTTP trailer

The name of this RFC comes from the digest trailing the data. An obvious alternative is to send it literally as an HTTP trailer on the `PUT`, so the storage node receives the client's digest with the data and can check it before responding.

This RFC reports the digest in the `/http/put` receipt instead. The receipt is signed and is already delivered to the upload service with `/ucan/conclude`, and the storage node receives it with the `/blob/accept` invocation. A trailer would put the agreed digest in unsigned HTTP metadata, visible only to the storage node, and the upload service would still need to learn the digest some other way before it could register the blob. Trailers are also dropped or rejected by some HTTP intermediaries.

[blob protocol]: https://github.com/fil-forge/ucan-protocol-specs/blob/main/blob.md
[DAG-JSON]: https://ipld.io/specs/codecs/dag-json/spec/
[blob removal]: ./2026-07-forge-blob-removal.md
[`did:key`]: https://w3c-ccg.github.io/did-method-key/
