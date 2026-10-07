# RFC: BLAKE3 object digest

Status: Experimental

## Authors

- [Alan Shaw](https://github.com/alanshaw)

## Motivation

What a client can verify about an Ingot object today depends on how the object was uploaded, and for most objects it is nothing. A single `PUT` records a whole-object SHA-256 in the manifest, but it is internal and never returned. A multipart object records no whole-object digest at all, because SHA-256 cannot be assembled from the parts and re-reading every part at `CompleteMultipartUpload` is too slow.

The one verifiable value is the per-part checksum list, and it exists only if the uploader asked for it. A multipart object uploaded with a SHA checksum algorithm, `--checksum-algorithm SHA256` or its SDK equivalent, has a checksum per part and a composite over them, and `GetObjectAttributes` returns the list after the upload completes. That is a two-level tree the client can check part-aligned ranges against, and check against the composite. No client does this by default: the AWS SDKs and CLI now send a checksum unprompted, but it is a CRC, which gives a full-object value with no per-part list; rclone, s5cmd and the MinIO client send none. The list covers nothing else: not a single `PUT`, not an upload made without a SHA algorithm, and no range finer than the uploader's parts, whose size the reader did not choose. The part ETags the object ETag is built from are not returned once the upload completes, and a per-part MD5 is available only when the uploader chose MD5 as the checksum algorithm, an opt-in like SHA-256; the ETag itself, an MD5 or an MD5 of part MD5s, identifies the object loosely and gives a reader no cryptographic assurance. The CRC family (CRC32, CRC32C, CRC64NVME) can be a full-object value, which Ingot derives at `CompleteMultipartUpload` by combining the parts' CRCs, but a CRC detects accidental corruption and nothing more.

Nothing lets a client verify a byte range the uploader did not happen to make a part.

This RFC proposes a whole-object BLAKE3 digest for every object, single `PUT` or multipart, returned to clients as a CID, plus a small tree of chaining values stored in the manifest that lets a client verify ranged reads. BLAKE3 is a Merkle tree, so the digest of a multipart object is assembled from per-part subtrees at `Complete` with no re-read of the data, and the same tree structure gives range verification for free.

Blob addressing on the Forge network stays SHA-256. This RFC adds an object-level digest alongside it and changes nothing about how blobs are stored, located or proven.

## Goals

- Every object has a whole-object BLAKE3 digest, computed in the same streaming pass that writes its blobs.
- A multipart object's digest is assembled at `Complete` from values recorded per part. Re-reading part data is a bounded fallback for the few parts whose position was assumed wrongly, never the common path, and an upload that would need more than the bound gets no digest rather than a slow `Complete`.
- A client can verify a whole-object read with the digest alone and a standard BLAKE3 implementation.
- A client can verify a ranged read with an off-the-shelf Bao library by fetching the object's outboard once: under a KiB for a small object, growing with the square root of the size.
- The S3 data path is unchanged. Bodies are raw bytes, ETags and S3 checksums behave as they do today, and a client that ignores the extension sees nothing new.
- Manifest growth stays small for ordinary objects and under 1 MiB for the largest. The block list grows with the square root of the object size and is capped.

## Background: the BLAKE3 tree

BLAKE3 hashes input as a binary Merkle tree over 1 KiB chunks. Each chunk is compressed to a 32-byte chaining value (CV). The compression of a chunk mixes in the chunk's index within the input, so the CV of a chunk depends on where in the input it sits. Pairs of CVs are compressed into parent CVs, with the index set to zero. The topmost compression is flagged as the root and its output is the hash.

The tree shape is fixed by the input length. For an input of more than one chunk, the left subtree holds the largest power-of-two number of chunks that is strictly less than the total, and the right subtree holds the rest, recursively. A consequence this RFC relies on: any aligned power-of-two block of chunks that lies within the input is a node of the tree. A block of 2^k chunks starting at a multiple of 2^k chunks has a CV that appears in the final tree, whatever the total length turns out to be.

Two further consequences follow. A contiguous byte range that starts on a chunk boundary decomposes into a short sequence of such aligned blocks, at most two per tree level, and its CVs can be computed knowing only the bytes and the starting offset. And the hash of the whole input can be computed from the CVs of any set of aligned blocks that partition it, with one parent compression per merge and the root flag applied at the last.

This is the structure that [Bao] and iroh's [bao-tree] use for verified streaming. Their "outboard" is the list of parent nodes, which lets a reader holding only the root verify any range. This RFC stores the bottom of that outboard at a coarse block size, serves the outboard itself to clients, and leaves the Bao streaming encoding for a possible later mode.

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

The body already streams through a SHA-256 hasher and an MD5 hasher as it is split into blobs. A BLAKE3 hasher joins the same pass. It produces the object digest and the block CVs described under [Range verification](#range-verification). Nothing is read twice.

### Multipart upload

Parts arrive in any order and the server learns a part's true byte offset only once every lower-numbered part exists. Each `UploadPart` therefore hashes its body at an assumed offset and records what `CompleteMultipartUpload` needs to finish the tree. `CompleteMultipartUpload` checks each assumed offset against the true one and re-hashes only the parts whose assumption was wrong, within a budget.

#### What `UploadPart` records

Each part record gains:

- `TreeOffset`: the byte offset the part was hashed at.
- `TreeNodes`: the CVs of the part's aligned subtrees, in order, covering the whole chunks between the part's first and last chunk boundaries. A span of whole chunks decomposes into at most two aligned power-of-two blocks per tree level, so this list holds a few dozen 32-byte entries at most.
- `TreeBlocks`: the part's block CVs at the part's own block size (see [Range verification](#range-verification)), a few hundred entries at most for a 5 GiB part.
- `TreeHead` and `TreeTail`: the part's bytes before its first chunk boundary and after its last one, under 1 KiB each. They are empty for a part whose offset and length are 1 KiB multiples. Every mainstream client sizes its parts in MiB, so both spans are empty for nearly every part ever uploaded. The one known exception is the AWS SDKs for Go and Java above the 10,000-part limit, objects over about 48 GiB, where they size parts as the object size divided by 10,000, an arbitrary byte count; those uploads are a small minority, and these two spans are what let them complete without re-reading anything.
- For part 1 only, `TreeRoot`: the BLAKE3 hash of the part on its own. Part 1 is always at offset 0, so its standalone hash falls out of the same pass with a root-flagged final compression.

A part cannot know whether it is the last, so it never treats a short final chunk as the object's end; those bytes go to `TreeTail`, and `CompleteMultipartUpload` compresses them once it knows what follows them.

A part that supersedes an earlier upload of the same part number replaces its record, as today.

#### Assuming the offset

- Part 1 is at offset 0.
- Part _n_ when every part below _n_ is already recorded: the sum of their sizes. This is exact.
- Part _n_ when some lower part is recorded: assume a uniform part size and use (_n_ − 1) times the size of the lowest-numbered recorded part.
- Part _n_ when no lower part is recorded: (_n_ − 1) times this part's own size.

Clients upload parts of one size in parallel, with a shorter final part. The rules above give the true offset for every part of a uniform-size upload, whatever that size is, except a final part that lands before any of its predecessors, which is re-hashed at `Complete` and is the smallest part of the upload. An assumed offset need not be a 1 KiB multiple: the part's first chunk boundary is wherever the offset puts it.

A part created by copy is hashed as its bytes are read, like an uploaded part. Where the copy does not read the bytes, the part records no tree and takes the re-hash path.

#### What `CompleteMultipartUpload` does

`CompleteMultipartUpload` already walks the requested parts in order and computes each part's true offset. A part's record is usable when its true offset equals `TreeOffset`.

The parts whose assumed offset was wrong are re-hashed at their true offsets by reading their blobs back from the local spool, and the results stand in for their records. The final part is always re-hashed when wrong: in a uniform-size upload it is the only part whose assumption can fail, when it lands before any predecessor and its own size stands in, and the 5 GiB part maximum bounds the cost. The other parts are re-hashed within a **re-hash budget**, one maximum part, 5 GiB, by default, and configurable. An upload whose wrong non-final parts add up to more than the budget commits without a digest: the object's ETag is returned as always, GET and HEAD carry no `x-cid`, and `GetObjectAttributes` omits the `Blake3` element, the state defined below for an object without a digest. The budget keeps `Complete` to seconds of local reading at most. Exceeding it is an extreme edge case. No mainstream client varies its part size within an upload, the final part aside, and a uniform-size upload assumes every offset exactly in any arrival order. A sequential upload assumes every offset exactly whatever the sizes. What is left is a hand-rolled uploader sending variable-size parts in parallel, or a parallel compose-by-copy of differently sized source objects, and even there the cost is the absence of a digest, never a failed upload.

A chunk that straddles a part boundary is rebuilt from the earlier part's `TreeTail` and the later part's `TreeHead` and compressed at its chunk index, one compression per boundary. The last part's `TreeTail` is the object's final chunk and is compressed as such. Part lengths that are not 1 KiB multiples therefore cost nothing.

With every part's nodes at their true offsets, the object digest is computed by merging adjacent aligned subtrees into parents and applying the root flag at the final merge. This is one compression per recorded subtree and reads no data. The work scales with the number of parts: a part contributes at most two subtrees per tree level, so an upload of 10,000 parts merges a few tens of thousands of nodes, each one BLAKE3 compression, which is milliseconds.

An upload completed with a single part has no merge, and its digest is that part's root as a body of its own. For part 1 that is the recorded `TreeRoot`. For any other part the recorded offset is not 0, so the offset check above re-hashes it at offset 0, and that re-hash yields the root directly. S3 requires the completed parts to be in ascending order, not to begin at part 1, so this case is legal and does occur.

The block CVs for the object are built from the parts' `TreeBlocks` and `TreeNodes` as described in the next section. The root computed by merging the block CVs up must equal the root computed from the parts' subtrees. This is a consistency check on the assembly at the cost of one compression per block, 32,767 at the cap. Should it ever fail, the object commits without a digest rather than with a wrong one; nothing re-reads the body.

### Range verification

Two terms, since BLAKE3 and Bao both have "leaves" and they are not the same thing. A **chunk** is BLAKE3's 1 KiB unit; a chunk's CV is a leaf of BLAKE3's own tree. A **block** is 2^ChunkLog chunks, the unit a client verifies; a block's CV is an interior node of BLAKE3's tree and a leaf of the Bao tree at that block size. This RFC stores and serves block CVs and never stores chunk CVs. The word "leaf" is not used below.

A client that reads ranges needs more than the root. It needs the CVs of the tree nodes covering the bytes it read, and enough of the rest of the tree to connect them to the root. The full Bao outboard provides this at 1 KiB granularity and costs about 6% of the object size. Storing it is out of the question in a manifest, and even at bao-tree's 16 KiB default it costs 4 MiB per GiB.

This RFC scales the block size with the object. The block is the geometric mean of the object size and 16 KiB, so the number of blocks grows with the square root of the size. A small object keeps a block list of a few hundred bytes, and the smallest verifiable read on a large object stays a small fraction of it.

#### The block

The **block** is the unit of the stored tree. Its target is the geometric mean of the object size and 16 KiB, the square root of their product:

```
target = sqrt(size × 16 KiB)
block  = the smallest power of two ≥ target, and at least 16 KiB
```

For a 1 MiB object the target is sqrt(1 MiB × 16 KiB) = sqrt(2^34) = 128 KiB, so the block is 128 KiB and there are 8 blocks; for a 1 GiB object it is sqrt(2^44) = 4 MiB, 256 blocks. The block doubles each time the object quadruples. A hard cap of 32,768 blocks, 1 MiB of CVs, applies beyond about 32 TB, where the block grows with the size instead. The manifest records the block size as a **chunk log**: a base-2 exponent of 1 KiB BLAKE3 chunks, the unit Bao libraries take as the block size, so 16 KiB is 4 and 4 MiB is 12. The block affects only what is stored and what granularity a client can verify at. The object digest is the same whatever block is chosen.

| Object size | Block | Chunk log | Blocks | Manifest bytes |
|---|---|---|---|---|
| 1 KiB | 16 KiB | 4 | 1 | 32 B |
| 64 KiB | 32 KiB | 5 | 2 | 64 B |
| 1 MiB | 128 KiB | 7 | 8 | 256 B |
| 4 MiB | 256 KiB | 8 | 16 | 512 B |
| 100 MiB | 2 MiB | 11 | 50 | 1.6 KiB |
| 1 GiB | 4 MiB | 12 | 256 | 8 KiB |
| 10 GiB | 16 MiB | 14 | 640 | 20 KiB |
| 100 GiB | 64 MiB | 16 | 1,600 | 50 KiB |
| 1 TiB | 128 MiB | 17 | 8,192 | 256 KiB |
| 5 TiB | 512 MiB | 19 | 10,240 | 320 KiB |
| 50 TB | 2 GiB | 21 | 23,284 | 728 KiB |

Sizes with binary prefixes are powers of two; the 50 TB row is decimal, 50 × 10¹² bytes, just under S3's current maximum object size of 48.8 TiB (10,000 parts of 5 GiB). Blocks is the size divided by the block size, rounded up, since a final partial block still has a CV. An object of 16 KiB or less is one block. From there the manifest cost grows with the square root of the size: a quarter of a KiB at 1 MiB, 8 KiB at 1 GiB, 256 KiB at 1 TiB. The cap applies past about 32 TB and holds a 50 TB object to 728 KiB.

The smallest range a client can verify independently is one block: 256 KiB on a 4 MiB object, 4 MiB on a 1 GiB object, 128 MiB on a 1 TiB object, 2 GiB on a 50 TB object. For a large object that is coarser than the per-part checksum list of a multipart upload made with SHA checksums: the block passes an 8 MiB part above 4 GiB, so on a 1 TiB object such a list verifies at 8 MiB where this tree verifies at 128 MiB. The tree's advantage is that it exists for every object, whatever the uploader did. Two constants tune the rule. The 16 KiB floor sets where the scaling starts, and the cap bounds the manifest. An object needing finer verification than its block would want the outboard-as-blob design under [Alternatives](#alternatives-considered).

#### Block CVs

The manifest stores the CV of every block of the object, in object order from offset 0: the **block CVs**, 32 bytes each. They are the nodes of BLAKE3's tree at the block level. Every node above them is derivable with one compression per node, so nothing above the blocks is stored, and nothing below them: chunk CVs are never stored.

A single `PUT` computes the block CVs in its hashing pass, since the object length is known from `Content-Length` and so is the block.

A multipart upload does not know the object length until `CompleteMultipartUpload`, so each part computes `TreeBlocks` at its own block size, chosen by the same rule from the part's own size. The object's block is never smaller than any part's, because the rule is monotone in size and the object is at least as large as each part. At `CompleteMultipartUpload`, each object block is one of:

- A block lying within one part. Its CV is the merge of that part's `TreeBlocks` that it covers.
- A block straddling a part boundary. The piece on each side is a union of that part's edge subtrees, which `TreeNodes` holds, so its CV is the merge of those entries.

Either way the block CVs are built from recorded values with no data read.

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
    <ChunkLog>12</ChunkLog>
    <Outboard>base64</Outboard>
  </Blake3>
</GetObjectAttributesOutput>
```

- `CID` is the object digest as returned in `x-cid`
- `ChunkLog` is the block size as a base-2 exponent of 1 KiB BLAKE3 chunks, the value a Bao library takes as its block size unchanged. iroh's fixed block is 4; the original Bao format is 0.
- `Outboard` is the standard pre-order Bao outboard at that block size, base64-encoded: the object size as 8 little-endian bytes, then the chaining-value pair of every parent node above the blocks, root first and left subtree before right. It is built from the stored block CVs on request, one compression per parent, and is byte-identical to what the Bao libraries produce for the same object and block size. It is an 8-byte prefix plus 64 bytes per parent, one fewer than the blocks: under 100 bytes for a small object, 16 KiB at 1 GiB, 1.4 MiB for a 50 TB object and 2 MiB at the 32,768-block cap, about 2.7 MiB after base64. A tree belongs to one version. An overwrite can never pair an old tree with new bytes, because both live in the same manifest, and the response carries the ETag of the version the tree describes for the client's `If-Match`.

AWS rejects an attribute name it does not know with `InvalidArgument`, so accepting `Blake3` is a visible divergence. SDKs whose typed response omits unknown elements cannot read the tree, but the same SDKs could not call a custom endpoint either, and a raw HTTP client parses this XML with what it already has for every other S3 response.

#### Client procedure

1. Fetch the attribute once with `GetObjectAttributes`, and load the CID's digest, the block size and the outboard into a Bao library (bao-tree in Rust, the Go BLAKE3 library's `bao` package). The library checks the outboard against the root as it goes; no merge code of our own is needed, and no BLAKE3 API beyond what a Bao library already wraps.
2. Read ranges with ordinary S3 `Range` requests aligned to block boundaries, sending `If-Match` with the ETag so the bytes cannot change between the attribute fetch and the reads. A Bao library can name the byte ranges needed for the blocks a client wants.
3. Verify each block received against the outboard. An object of one block has an empty outboard, and the client verifies by hashing the whole object against the CID.

Every byte of data travels as a plain S3 body. SDKs, caches and CDNs behave as they do today.

### Manifest changes

The body gains three fields. Sizes are the maximum for any object.

```go
type Body struct {
	// ...existing fields

	// BLAKE3 is the whole-object blake3 multihash (34 bytes).
	BLAKE3 []byte `cborgen:"b3"`
	// TreeChunkLog is the block size of TreeBlocks as a base-2 exponent of chunks (one byte).
	TreeChunkLog uint8 `cborgen:"tc"`
	// TreeBlocks holds the chaining values of the blocks of the
	// body, 32 bytes each, in order (8 KiB at 1 GiB, at most 1 MiB).
	TreeBlocks []byte `cborgen:"tb"`
}
```

Some objects have no digest: those written before this change, and a multipart object whose completion exceeded the re-hash budget. For them, GET and HEAD return no `x-cid` header and `GetObjectAttributes` returns its response without the `Blake3` element.

That makes two situations look alike from a GET or HEAD: an object without a digest, and a server that does not implement this RFC. `GetObjectAttributes` tells them apart. The request names the attributes it wants in `x-amz-object-attributes`, and a server validates the names: one that implements this RFC accepts `Blake3` and answers, with or without the element depending on the object, while one that does not, AWS included, rejects the request with `InvalidArgument`. So a client that wants to know whether verification is available at all asks for `Blake3` once; an error means the server cannot provide it for any object, and a success without the element means it cannot provide it for this one.

### Cost

Ingest adds one BLAKE3 pass per body or part, alongside the MD5 pass already there. BLAKE3 with vector instructions is faster than SHA-256, so it costs less than the whole-body SHA-256 pass it makes redundant.

`CompleteMultipartUpload` adds one compression per recorded subtree for the digest and one per block for the consistency check, a few tens of thousands at most for the largest uploads, which is milliseconds. The re-hash path reads the affected parts back through the plaintext opener, which decrypts in an encrypting deployment. The parts are read from the local spool: a part's blobs stay there until the object is deleted, since nothing evicts spool copies today, so no re-hash fetches from a storage node. The work is bounded by the final part plus the budget, about 10 GiB of local reading at most with the default, which is seconds to tens of seconds. The price of the bound is that a variable-size parallel upload beyond it gets no digest.

The manifest grows by the block list, 8 KiB at 1 GiB and at most 1 MiB, plus the 34-byte multihash and a few bytes of framing. The part record grows by at most about 15 KiB for the life of the upload.

## Alternatives considered

### Whole-object SHA-256 for multipart objects

Keeping the existing SHA-256 and computing it for multipart objects means re-reading every part at `Complete`. For a 5 GiB object that is a 5 GiB read, decrypting where encryption is on, inside a request a client expects to return promptly. It also offers no range verification. BLAKE3 removes the re-read and adds range verification with the same stored bytes.

### A root SHA-256 over per-part SHA-256 hashes

S3 already defines this for uploads made with `--checksum-algorithm SHA256`: a SHA-256 per part and a composite over the list, which `GetObjectAttributes` returns. The alternative is to compute that for every multipart upload, server side, whatever the client asked for, and treat the composite as the object's digest. It needs no new primitive, and for a client that already understands S3 composite checksums it is familiar. It falls short of this RFC in several ways.

- **The root is not a hash of the bytes.** A composite depends on where the part boundaries fell, so the same bytes uploaded with 8 MiB parts, with 16 MiB parts, or as a single `PUT` get three different roots. It identifies an upload, not an object: two copies of the same data cannot be recognised as equal, nothing can be addressed by it, and a single-`PUT` object has a plain SHA-256 while a multipart object has a hash of hashes, so a client needs two code paths and two meanings for "the digest".
- **Verifying a download takes a list, not a value.** The verifier must fetch the per-part hash list and the part sizes, up to 10,000 of each, keep them for the whole download, and apply the right one to the right byte range. The BLAKE3 digest is one 32-byte value that a standard library verifies against a stream. Walking a 10,000-entry list is custom code in every client, and the list itself is hundreds of KiB of XML on every fetch, with no compact form.
- **The granularity is the uploader's, and often too coarse.** Verification happens per part, so a client cannot check anything until it has a whole part, 5 GiB on large uploads, and a single-`PUT` object cannot be range-verified at all. The Bao tree verifies 16 KiB on small objects, and a server-chosen block on large ones, with proofs that are logarithmic in the object rather than linear in the part count.
- **No standard verifies it.** There is no library that takes a composite SHA-256 and a part list and verifies a stream or a range, so every consumer writes its own; the Bao outboard is loadable by existing libraries in Rust and Go.
- **It costs about the same.** A SHA-256 pass per part is no cheaper than the BLAKE3 pass, which is faster per byte, and the composite still has to be assembled at `Complete` from recorded values, as the tree is.

What it does offer over the current state is universality within multipart uploads: every multipart object would have a verifiable per-part list rather than only those uploaded with SHA checksums. BLAKE3 gives that and the rest.

### Re-addressing blobs to BLAKE3

Making BLAKE3 the network's content address would let storage nodes serve verified ranges directly. It touches every component that handles a multihash, the proof-tree assumptions of PDP, public gateway resolvability, and every stored byte. This RFC is independent of that decision. The object digest proposed here is the same value such a migration would use, so nothing here is thrown away if it happens.

### A fixed 16 KiB block with the outboard as a blob

bao-tree's default gives 16 KiB verification granularity at any size, with the outboard stored as its own blob referenced from the manifest, about 0.4% of the object. It is the right design if fine-grained verified reads of very large objects are a requirement. It adds a blob per object to write, locate and clean up, and a second request to fetch something that is usually megabytes. The bounded manifest tree serves the common case with no new blob and can coexist with this design later: the stored block CVs are the top of the finer tree.

### Returning the Bao encoded stream

iroh serves data interleaved with the parent nodes so a client verifies as bytes arrive. S3 SDKs cannot consume such a body, so it would be a separate mode rather than the S3 `GET`. At the block sizes this RFC uses a client must buffer a whole block before verifying anything, so interleaving saves only the one tree request. The outboard served by `GetObjectAttributes` is already the Bao form, so the encoded stream can be added later as its own subresource without changing what is stored.

### Carrying the digest in an S3 checksum header

The `x-amz-checksum-*` headers have a fixed set of algorithms and SDKs validate the names, so BLAKE3 cannot ride in one.

### A dedicated subresource for the tree

A `?blake3` subresource returning a CBOR document would keep `GetObjectAttributes` identical to AWS. It costs a new endpoint, a second place that must honour `versionId`, and a body format no S3 client parses today. No SDK can call it, so it has no advantage over an attribute for SDK users, and a raw HTTP client is better served by the XML it already handles.

### Returning the block list instead of the outboard

The stored block CVs are half the size of the outboard, and a client could merge them to the root itself. But checking a block against its CV means hashing the block as a non-root subtree at its chunk offset, which standard BLAKE3 APIs do not expose, so every client would carry custom tree code. Serving the Bao outboard costs twice the bytes and lets any Bao library do the verification.

### Sending the block CVs in a header

Base64 of a 1 GiB object's 16 KiB outboard is about 22 KiB, over the 8 KiB per-header default of common proxies, larger objects have far more, and splitting it across numbered headers is a hack. The root fits in a header; the tree does not.

## Open questions

- The re-hash budget's default for non-final parts. The final part is exempt, so the budget only governs variable-size parallel uploads; a larger default trades `Complete` latency for digests on more of them.

- Names: `x-cid` and the `Blake3` attribute are placeholders.
- The scaling constants: the 16 KiB floor and the 32,768-block cap. The exponent is a knob too; cube-root scaling would give a 50 TB object 1,456 blocks at a 32 GiB block.
- Whether a `GET` with `?partNumber` should also return the block CVs covering that part, so a client that already reads by part need not align to blocks.
- Whether the object CID should reach the Forge network, for example in the upload index, or remain an Ingot-level value until blob addressing is revisited.
- Whether to expose the digest on `ListObjects` responses.

## Evaluation criteria

- The S3 conformance suite passes unchanged: no ETag, checksum or body behaviour differs.
- Ingest throughput with the extra hash is within a few percent of today's.
- The share of `Complete` calls that take the re-hash path, measured against the AWS CLI, boto3, the Go SDK's uploader above the 10,000-part threshold, rclone, s5cmd and the MinIO client at their default part sizes, is near zero, and none exceeds the budget.
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
