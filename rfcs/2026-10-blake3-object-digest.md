# RFC: BLAKE3 object digest

Status: Experimental

## Authors

- [Alan Shaw](https://github.com/alanshaw)

## Motivation

An Ingot object has no digest a client can verify against. A single `PUT` records a whole-object SHA-256 in the manifest, but it is internal and never returned. A multipart object records no whole-object digest at all, because SHA-256 cannot be assembled from the parts and re-reading every part at `CompleteMultipartUpload` is too slow. A client that wants to check what it downloaded has the S3 ETag, which for a multipart object is a hash of part MD5s and verifies nothing about the bytes, and the optional S3 checksums, which for multipart are composite values with the same limitation. Nothing lets a client verify a byte range.

This RFC proposes a whole-object BLAKE3 digest for every object, single `PUT` or multipart, returned to clients as a CID, plus a small tree of chaining values stored in the manifest that lets a client verify ranged reads. BLAKE3 is a Merkle tree, so the digest of a multipart object is assembled from per-part subtrees at `Complete` with no re-read of the data, and the same tree structure gives range verification for free.

Blob addressing on the Forge network stays SHA-256. This RFC adds an object-level digest alongside it and changes nothing about how blobs are stored, located or proven.

## Goals

- Every object has a whole-object BLAKE3 digest, computed in the same streaming pass that writes its blobs.
- A multipart object's digest is assembled at `Complete` from values recorded per part. Re-reading part data is a fallback for unusual clients, never the common path.
- A client can verify a whole-object read with the digest alone and a standard BLAKE3 implementation.
- A client can verify a ranged read by fetching the object's leaf list once: a few hundred bytes for a small object, growing with the square root of the size.
- The S3 data path is unchanged. Bodies are raw bytes, ETags and S3 checksums behave as they do today, and a client that ignores the extension sees nothing new.
- Manifest growth stays small for ordinary objects and under 1 MiB for the largest. The leaf list grows with the square root of the object size and is capped.

## Background: the BLAKE3 tree

BLAKE3 hashes input as a binary Merkle tree over 1 KiB chunks. Each chunk is compressed to a 32-byte chaining value (CV). The compression of a chunk mixes in the chunk's index within the input, so the CV of a chunk depends on where in the input it sits. Pairs of CVs are compressed into parent CVs, with the index set to zero. The topmost compression is flagged as the root and its output is the hash.

The tree shape is fixed by the input length. For an input of more than one chunk, the left subtree holds the largest power-of-two number of chunks that is strictly less than the total, and the right subtree holds the rest, recursively. A consequence this RFC relies on: any aligned power-of-two block of chunks that lies within the input is a node of the tree. A block of 2^k chunks starting at a multiple of 2^k chunks has a CV that appears in the final tree, whatever the total length turns out to be.

Two further consequences follow. A contiguous byte range that starts on a chunk boundary decomposes into a short sequence of such aligned blocks, at most two per tree level, and its CVs can be computed knowing only the bytes and the starting offset. And the hash of the whole input can be computed from the CVs of any set of aligned blocks that partition it, with one parent compression per merge and the root flag applied at the last.

This is the structure that [Bao] and iroh's [bao-tree] use for verified streaming. Their "outboard" is the list of parent nodes, which lets a reader holding only the root verify any range. This RFC stores a reduced form of that outboard and leaves the Bao wire format for a possible later mode.

## Design

### The object digest

The object digest is the BLAKE3 hash of the object's plaintext body. It is stored in the manifest as a [multihash] with code `0x1e` (`blake3`) and a 32-byte digest.

Clients receive it as a [CID]: version 1, codec `raw` (`0x55`), with that multihash, encoded with [multibase] base32 lowercase. A CID is self-describing, names content the same way the rest of the Forge stack does, and parses with any CID library. The `raw` codec states that the identified bytes are the object body itself, with no framing.

The CID is returned in a response header on `GetObject` and `HeadObject`, including ranged and `?partNumber` reads. The header name is provisional:

```
x-cid: bafkr4i...
```

A client verifying a whole-object read hashes the body with any BLAKE3 implementation and compares. It needs nothing else from this RFC.

### Single `PUT`

The body already streams through a SHA-256 hasher and an MD5 hasher as it is split into blobs. A BLAKE3 hasher joins the same pass. It produces the object digest and the leaf CVs described under [Range verification](#range-verification). Nothing is read twice.

### Multipart upload

Parts arrive in any order and the server learns a part's true byte offset only once every lower-numbered part exists. Each `UploadPart` therefore hashes its body at a guessed offset and records what `CompleteMultipartUpload` needs to finish the tree. `CompleteMultipartUpload` checks the guesses against the true offsets and re-hashes only the parts whose guess was wrong.

#### What `UploadPart` records

Each part record gains:

- `TreeOffset`: the byte offset the part was hashed at.
- `TreeNodes`: the CVs of the part's aligned subtrees, in order. A part of any length starting on a chunk boundary decomposes into at most two aligned power-of-two blocks per tree level, so this list holds a few dozen 32-byte entries at most.
- `TreeLeaves`: the part's leaf CVs at the part's own group (see [Range verification](#range-verification)), a few hundred entries at most for a 5 GiB part.
- For part 1 only, `TreeRoot`: the BLAKE3 hash of the part on its own. Part 1 is always at offset 0, so its standalone hash falls out of the same pass with a root-flagged final compression.

A part that supersedes an earlier upload of the same part number replaces its record, as today.

#### Guessing the offset

- Part 1 is at offset 0.
- Part _n_ when every part below _n_ is already recorded: the sum of their sizes. This is exact.
- Part _n_ when some lower part is recorded: assume a uniform part size and use (_n_ − 1) times the size of the lowest-numbered recorded part.
- Part _n_ when no lower part is recorded: (_n_ − 1) times this part's own size.

Clients upload parts of one size in parallel, with a shorter final part. The rules above guess right for every part of a uniform-size upload except a final part that lands before any of its predecessors, which is re-hashed at `Complete` and is the smallest part of the upload.

A part created by copy is hashed as its bytes are read, like an uploaded part. Where the copy does not read the bytes, the part records no tree and takes the re-hash path.

#### What `CompleteMultipartUpload` does

`CompleteMultipartUpload` already walks the requested parts in order and computes each part's true offset. For each part it checks that the true offset is a multiple of 1 KiB and equals `TreeOffset`. A part that fails either check is re-hashed at its true offset by reading its blobs back, and its record is replaced before the merge proceeds. A part offset that is not a multiple of 1 KiB means some earlier part has a length that is not a multiple of 1 KiB, which places a chunk boundary inside a part. Such parts and every later part are re-hashed. This is rare: clients size parts in MiB.

With every part's `TreeNodes` at correct offsets, the object digest is computed by merging adjacent aligned subtrees into parents and applying the root flag at the final merge. This is one compression per node, a few hundred at most, and reads no data.

An upload completed with a single part has no merge. Its digest is part 1's `TreeRoot`.

The leaf CVs for the object are built from the parts' `TreeLeaves` and `TreeNodes` as described in the next section. The root computed by merging the leaf CVs up must equal the root computed from the parts' subtrees. This is a free consistency check on the assembly.

### Range verification

A client that reads ranges needs more than the root. It needs the CVs of the tree nodes covering the bytes it read, and enough of the rest of the tree to connect them to the root. The full Bao outboard provides this at 1 KiB granularity and costs about 6% of the object size. Storing it is out of the question in a manifest, and even at bao-tree's 16 KiB default it costs 4 MiB per GiB.

This RFC scales the leaf size with the object. The group is the geometric mean of the object size and 16 KiB, so the number of leaves grows with the square root of the size. A small object keeps a leaf list of a few hundred bytes, and the smallest verifiable read on a large object stays a small fraction of it.

#### The group

The **group** is the leaf size of the stored tree: a power of two, at least 16 KiB, and the smallest such value that is at least the square root of the object size times 16 KiB. The group doubles each time the object quadruples. A hard cap of 32,768 leaves, 1 MiB of CVs, applies beyond about 32 TB, where the group grows with the size instead. The manifest records the group as a base-2 exponent. The group affects only what is stored and what granularity a client can verify at. The object digest is the same whatever group is chosen.

| Object size | Group | Leaves | Manifest bytes |
|---|---|---|---|
| 1 KiB | 16 KiB | 1 | 32 B |
| 64 KiB | 32 KiB | 2 | 64 B |
| 1 MiB | 128 KiB | 8 | 256 B |
| 4 MiB | 256 KiB | 16 | 512 B |
| 100 MiB | 2 MiB | 50 | 1.6 KiB |
| 1 GiB | 4 MiB | 256 | 8 KiB |
| 10 GiB | 16 MiB | 640 | 20 KiB |
| 100 GiB | 64 MiB | 1,600 | 50 KiB |
| 1 TiB | 128 MiB | 8,192 | 256 KiB |
| 5 TiB | 512 MiB | 10,240 | 320 KiB |
| 50 TB | 2 GiB | 23,283 | 728 KiB |

An object of 16 KiB or less is one leaf. From there the manifest cost grows with the square root of the size: a quarter of a KiB at 1 MiB, 8 KiB at 1 GiB, 256 KiB at 1 TiB. The cap applies past about 32 TB and holds a 50 TB object to 728 KiB.

The smallest range a client can verify independently is one group: 256 KiB on a 4 MiB object, 4 MiB on a 1 GiB object, 128 MiB on a 1 TiB object, 2 GiB on a 50 TB object. Two constants tune the rule. The 16 KiB floor sets where the scaling starts, and the cap bounds the manifest. An object needing finer verification than its group would want the outboard-as-blob design under [Alternatives](#alternatives-considered).

#### Leaf CVs

The manifest stores the CV of every group-aligned block of the object, in order: the **leaf CVs**. These are the nodes of the BLAKE3 tree at the group level, each 32 bytes. Every node above them is derivable with one compression per node, so nothing above the leaves is stored.

A single `PUT` computes the leaf CVs in its hashing pass, since the object length is known from `Content-Length` and so is the group.

A multipart upload does not know the object length until `CompleteMultipartUpload`, so each part computes `TreeLeaves` at its own group, chosen by the same rule from the part's own size. The object's group is never smaller than any part's group, because the rule is monotone in size and the object is at least as large as each part. At `CompleteMultipartUpload`, each object leaf is one of:

- A block lying within one part. Its CV is the merge of that part's `TreeLeaves` that it covers.
- A block straddling a part boundary. The piece on each side is a union of that part's edge subtrees, which `TreeNodes` holds, so its CV is the merge of those entries.

Either way the leaf CVs are built from recorded values with no data read.

#### The tree in `GetObjectAttributes`

A client fetches the tree with `GetObjectAttributes` (`?attributes`), which already returns a per-part checksum list for a multipart object and already takes `versionId`. This RFC adds an attribute name, `Blake3`, to the `x-amz-object-attributes` request header. Like the existing attributes it is returned only when requested, so responses to clients that do not ask for it are unchanged. The name is provisional.

```
GET /{bucket}/{key}?attributes[&versionId=...]
x-amz-object-attributes: ETag,Blake3
```

```xml
<GetObjectAttributesOutput>
  <ETag>...</ETag>
  <Blake3>
    <CID>bafkr4i...</CID>
    <Group>22</Group>
    <Leaves>
      <Leaf>base64 of 32 bytes</Leaf>
      ...
    </Leaves>
  </Blake3>
</GetObjectAttributesOutput>
```

`CID` is the object digest as returned in `x-cid`, `Group` is the group exponent and each `Leaf` is a leaf CV in order, encoded in base64 like the checksums in `ObjectParts`. The element is a few hundred bytes for a small object, about 12 KiB at 1 GiB and up to 1.6 MiB at the leaf cap. A tree belongs to one version. An overwrite can never pair an old tree with new bytes, because both live in the same manifest, and the response carries the ETag of the version the tree describes for the client's `If-Match`.

AWS rejects an attribute name it does not know with `InvalidArgument`, so accepting `Blake3` is a visible divergence. SDKs whose typed response omits unknown elements cannot read the tree, but the same SDKs could not call a custom endpoint either, and a raw HTTP client parses this XML with what it already has for every other S3 response.

#### Client procedure

1. Fetch the tree once with `GetObjectAttributes`. Compute the parents from the leaf CVs up to the root and compare with the CID. This is at most 255 compressions. When the tree holds a single leaf the object is at most one group long, and the client verifies by hashing the whole object instead.
2. Read ranges with ordinary S3 `Range` requests aligned to group boundaries, sending `If-Match` with the ETag so the bytes cannot change between the tree fetch and the reads.
3. Hash each group received as a non-root BLAKE3 subtree at its chunk offset and compare with its leaf CV.

Every byte of data travels as a plain S3 body. SDKs, caches and CDNs behave as they do today.

### Manifest changes

The body gains three fields. Sizes are the maximum for any object.

```go
type Body struct {
	// ...existing fields

	// BLAKE3 is the whole-object blake3 multihash (34 bytes).
	BLAKE3 []byte `cborgen:"b3"`
	// TreeGroup is the base-2 exponent of the leaf size of TreeLeaves (one byte).
	TreeGroup uint8 `cborgen:"tg"`
	// TreeLeaves holds the chaining values of the group-aligned blocks of the
	// body, 32 bytes each, in order (8 KiB at 1 GiB, at most 1 MiB).
	TreeLeaves []byte `cborgen:"tl"`
}
```

Objects written before this change have none of these fields. A read of such an object returns no `x-cid` header and `GetObjectAttributes` omits the `Blake3` element, so a client can tell the difference between an object without a digest and a server without the feature.

### Cost

Ingest adds one BLAKE3 pass per body or part, alongside the SHA-256 and MD5 passes already there. BLAKE3 with vector instructions is faster than SHA-256, so the added CPU cost is below the two hashes already paid.

`CompleteMultipartUpload` adds a few hundred compressions in the common path. The re-hash path reads the affected parts back through the plaintext opener, which decrypts in an encrypting deployment, and is bounded by the size of the mis-guessed parts.

The manifest grows by the leaf list, 8 KiB at 1 GiB and at most 1 MiB, plus the 34-byte multihash and a few bytes of framing. The part record grows by at most about 15 KiB for the life of the upload.

## Alternatives considered

### Whole-object SHA-256 for multipart objects

Keeping the existing SHA-256 and computing it for multipart objects means re-reading every part at `Complete`. For a 5 GiB object that is a 5 GiB read, decrypting where encryption is on, inside a request a client expects to return promptly. It also offers no range verification. BLAKE3 removes the re-read and adds range verification with the same stored bytes.

### Re-addressing blobs to BLAKE3

Making BLAKE3 the network's content address would let storage nodes serve verified ranges directly. It touches every component that handles a multihash, the proof-tree assumptions of PDP, public gateway resolvability, and every stored byte. This RFC is independent of that decision. The object digest proposed here is the same value such a migration would use, so nothing here is thrown away if it happens.

### A fixed 16 KiB group with the outboard as a blob

bao-tree's default gives 16 KiB verification granularity at any size, with the outboard stored as its own blob referenced from the manifest, about 0.4% of the object. It is the right design if fine-grained verified reads of very large objects are a requirement. It adds a blob per object to write, locate and clean up, and a second request to fetch something that is usually megabytes. The bounded manifest tree serves the common case with no new blob and can coexist with this design later: the stored leaves are the top of the finer tree.

### Returning the Bao encoded stream

iroh serves data interleaved with the parent nodes so a client verifies as bytes arrive. S3 SDKs cannot consume such a body, so it would be a separate mode rather than the S3 `GET`. At the groups this RFC uses a client must buffer a whole group before verifying anything, so interleaving saves only the one tree request. The server can expand the stored leaf CVs into a pair-form outboard on the fly, so the encoded stream can be added later as its own subresource without changing what is stored.

### Carrying the digest in an S3 checksum header

The `x-amz-checksum-*` headers have a fixed set of algorithms and SDKs validate the names, so BLAKE3 cannot ride in one.

### A dedicated subresource for the tree

A `?blake3` subresource returning a CBOR document would keep `GetObjectAttributes` identical to AWS. It costs a new endpoint, a second place that must honour `versionId`, and a body format no S3 client parses today. No SDK can call it, so it has no advantage over an attribute for SDK users, and a raw HTTP client is better served by the XML it already handles.

### Sending the leaf CVs in a header

Base64 of a 1 GiB object's 8 KiB leaf list is about 11 KiB, over the 8 KiB per-header default of common proxies, larger objects have far more, and splitting it across numbered headers is a hack. The root fits in a header; the tree does not.

## Open questions

- Names: `x-cid` and the `Blake3` attribute are placeholders.
- The scaling constants: the 16 KiB floor and the 32,768-leaf cap. The exponent is a knob too; cube-root scaling would give a 50 TB object 1,456 leaves at a 32 GiB group.
- Whether a `GET` with `?partNumber` should also return the leaf CVs covering that part, so a client that already reads by part need not align to groups.
- Whether the object CID should reach the Forge network, for example in the upload index, or remain an Ingot-level value until blob addressing is revisited.
- Whether to expose the digest on `ListObjects` responses.

## Evaluation criteria

- The S3 conformance suite passes unchanged: no ETag, checksum or body behaviour differs.
- Ingest throughput with the extra hash is within a few percent of today's.
- The share of `Complete` calls that take the re-hash path, measured against the AWS CLI, boto3, rclone, s5cmd and the MinIO client at their default part sizes, is near zero.
- Manifest size distribution matches the table above.
- A reference client verifies whole-object and ranged reads against a running Ingot, including a multipart object with non-power-of-two parts and a mis-ordered final part.

## References

- [BLAKE3 specification](https://github.com/BLAKE3-team/BLAKE3-specs/blob/master/blake3.pdf)
- [Bao]
- [bao-tree]
- [multihash] and the [multicodec table](https://github.com/multiformats/multicodec/blob/master/table.csv)
- [CID]
- [multibase]
- [S3 GetObjectAttributes](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObjectAttributes.html)
- [lukechampine.com/blake3](https://github.com/lukechampine/blake3), a Go implementation exposing the tree primitives (`guts`) and Bao with chunk groups

[Bao]: https://github.com/oconnor663/bao
[bao-tree]: https://github.com/n0-computer/bao-tree
[multihash]: https://github.com/multiformats/multihash
[CID]: https://github.com/multiformats/cid
[multibase]: https://github.com/multiformats/multibase
