# Erasure coding RFC: points to cover

Working notes for authoring the RFC. Each item needs an answer, or an explicit open question, in the RFC text.

## Direction so far

All dated 19 November 2026.

- **Coded data is stored as stripes.** A multipart part or object body is cut into chunks, the unit that is a blob today. Each chunk is cut into k data pieces, m parity pieces are computed from them, and a stripe record lists all k + m pieces. The stripe is identified by the digest of that record. Coding one chunk per stripe lets the work sit in a shared library that any client can use. The alternative was to group k whole blobs into a stripe. It keeps healthy reads as they are today, but each client would need its own logic for sizing blobs, grouping them and tracking dead stripe members.
- **A stripe is its own thing, with its own commands.** It is not a blob with a stripe attached. A blob is named by the digest of its bytes, and in a coded write no party except the client ever holds the chunk's bytes, so nothing can check such a digest without adding protocol. A stripe's digest is checkable from what the storage nodes already verify.

### How the catalog, stripe record and pieces relate

```
catalog:        stripe S at plaintext offset 0
                S = digest of the stripe record

stripe record:  scheme RS 4+2, cell size, chunk size
                slot 1  digest p1    data
                slot 2  digest p2    data
                slot 3  digest p3    data
                slot 4  digest p4    data
                slot 5  digest p5    parity
                slot 6  digest p6    parity

nodes:          p1 ... p6, on six different nodes
```

- The catalog names the stripe by S, the digest of its record. An object body is a list of stripe digests with plaintext offsets.
- A healthy read fetches the stripe record by S, checks it by hashing, fetches p1 to p4 and puts their cells back in order. A degraded read fetches any four of the six and decodes. Every piece is checked against its digest.
- Piece digests are stored in slot order. Whether a piece is data or parity follows from its slot and the scheme, so it is not stored.
- The chunk size is stored so a reader knows where the last row ends.
- The record does not contain S, and nothing in it may change after it is written. Where each piece is stored is kept apart from the record, because placement changes on repair.
- S commits to the scheme and to every piece, parity included.

### Protocol

- **Two new commands, `/stripe/add` and `/stripe/remove`.** `/blob/add` and `/blob/remove` are left as they are, and plain blobs keep using them.

  ```
  /stripe/add     scheme, chunk size, digest code, nonce
                  -> n allocations, n /http/put tasks, n /blob/accept tasks

  client streams the n pieces and concludes the n /http/put receipts

  upload service  checks the piece digests, encodes the stripe record,
                  stores it, returns its digest S

  /stripe/remove  S
                  -> removes the record and every piece
  ```

- **Storage nodes need no changes.** Each piece is an ordinary blob with an ordinary digest, handled through the existing allocate and accept calls.
- **Pieces stream to storage nodes before any digest is known**, using digest-less allocations from the [trailer hash RFC](https://github.com/fil-one/RFC/blob/main/rfcs/2026-09-trailer-hash.md). The client hashes each piece as it sends, each `/http/put` receipt reports that piece's digest, and each node checks its own computed digest at `/blob/accept`. Piece count and piece sizes follow from the chunk size and scheme.
- **The upload service computes S.** It has every piece digest verified by the nodes, so it encodes the record and hashes it. The client computes the same S locally and writes it into the catalog. Nothing rests on the client's word, and there is no closing message beyond today's `/ucan/conclude`. The encoding must be deterministic, and the hash covers the keyed union wrapper.
- **Each `/http/put` principal includes the slot number** in its derivation from the `/stripe/add` task link, so the n puts of one add have n keys.
- **The layout is interleaved.** All n pieces stream for the whole upload and the encoder holds one row of k + m cells. Contiguous pieces would hold parity until the last data byte.
- **A failed piece upload fails the request.** The S3 client retries the PUT or the part. A write needs all n pieces accepted, and the client does no rebuilding or repair of its own. One slow or failed node out of n fails the write, which is acceptable while operators are trusted and close together.
- **A client cannot remove a single piece.** Pieces are not the caller's blobs: they are not listed in the space and not counted one by one. A piece is removed only when its stripe is removed, or when repair or re-encoding replaces it. Degraded reads and repair are tested by an operator taking a piece or a node away, never by a client command.
- **S changes if the chunk is re-encoded under another scheme.** A scheme change needs a catalog update per stripe or a redirect from the old S to the new one.

Open:

- [ ] How the upload service keeps pieces apart from the space's blobs, so that `/blob/remove` with a piece digest does nothing to the stripe. This includes a caller adding a plain blob with the same bytes as a piece and then removing it.
- [ ] How a catalog entry tells a stripe digest from a blob digest, since both kinds of entry exist side by side.
- [ ] Cell size. It sets the memory held per upload (one row) and how often a range read stays within one piece. It is a field of the scheme arm and can vary per chunk: for example the largest power of two that still gives the chunk a full row, up to a cap. A chunk gets full coding efficiency only when it fills a row of k cells.
- [ ] The short last row. Data cells keep their real length, missing ones count as zeros, and parity cells are as long as the row's longest data cell, so no padding is stored.
- [ ] Whether the client keeps a plain hash of the chunk's bytes in its own catalog entry, for its own checks. The protocol does not need one.
- [ ] Which digest function pieces use.
- [ ] How `/stripe/add` combines with the multi-blob add: several stripes in one call, and all pieces of a stripe placed in separate failure domains.
- [ ] The Hilt grants for the two new commands, and which S3 actions need them.
- [ ] Whether repair needs a command of its own. Add one only if the design requires it.

Set aside:

- **Coding on `/blob/add`, with a blob digest D over the chunk's bytes.** D can be reported by the client and checked by nobody, which also needs a closing invocation under the client's own authority, because the `/http/put` keys are public. Or D can be derived from per-cell values reported by the storage nodes: BLAKE3 chaining values, the cell-level nodes of the piece CID tree, or a set hash such as LtHash. Each of those changes the allocation, the storage node's hashing or the accept receipt.

### Stripe record encoding

- **Encode the stripe record as a keyed union:** a single-entry CBOR map whose key is a type and version string naming the payload's format. This is the pattern Ingot uses for catalog value blocks, for example `/objectmanifest/1` and `/blobextent/0` (`ValueUnion` and `BlobExtentUnion` in Ingot's `bucket` package). A format revision adds an arm under a new key and old records keep decoding.
- Decode through a wrapper that fails when no known key is present. The decoder cbor-gen generates skips unknown keys silently, which would turn a record in a newer format into an empty one.
- **One arm per scheme.** The union key names the scheme and its version, so each scheme carries its own fields and the union is the list of schemes a reader supports. Copies is an arm like any other.
- [ ] Choose the key names, and say which fields every arm must carry (chunk size, piece digests in slot order).

## Accounting

- [ ] How stored data is recorded: with parity bytes or without.
- [ ] How egress is recorded: with parity bytes or without.

## Write path

- [x] Whether piece uploads are raced, as Storj does: send more pieces than needed and stop once enough have landed. No. A write needs all n pieces, so there is nothing to race.
- [x] Whether erasure coding fits the existing blob protocol or needs a different one. It needs two new commands at the upload service, and storage nodes keep the blob protocol: see [Protocol](#protocol).
- [x] Piece layout and how the encoder streams. Interleaved, one row at a time. Pieces can stream to nodes before their digests are known: see [Protocol](#protocol). See also [Streaming](./how-reed-solomon-works.md#streaming).
- [ ] Where the coding code lives, so that clients other than Ingot can reuse it. A shared library is the likely home.
- [x] How a read finds the stripe record. The catalog holds the record's digest: see [Protocol](#protocol).
- [ ] Where stripe records are stored and served from. Every read needs the record, healthy or degraded.
- [ ] Multipart: each part is coded alone, as one or more stripes. Parts smaller than k times the minimum piece size need copies or a smaller k. This overlaps with the small data question below.
- [x] Memory held while encoding one chunk. One row of k + m cells, set by the cell size.
- [ ] Read cost of the interleaved layout. A read longer than one cell holds up to k fetches open at once and runs at the pace of the slowest node. Measure it against today's read path for the cell sizes under consideration.
- [ ] The changes to Ingot's read path to resolve stripes.
- [x] When a PUT is acknowledged relative to piece durability. After all n pieces are accepted: see [Protocol](#protocol).

## Metadata durability

- [ ] How the stripe record is protected. Losing it loses the data, because the pieces cannot be reassembled without it.

## Coexistence and upgrades

- [x] Whether plain blobs and erasure-coded data are both supported at once. Yes. Plain blobs stay on `/blob/add` and coded data uses `/stripe/add`.
- [ ] Upgradability: a later move to a different coding technique should be possible. A new scheme is a new arm of the stripe record. Re-encoding a chunk changes its stripe digest, so say whether a move updates the catalog or leaves a redirect from the old digest.

## Carried over from the staged plan RFC

- [ ] The open questions in Hannah's [staged plan](https://github.com/fil-one/RFC/pull/39), including how to handle small data.

## Research

- [x] Survey existing erasure coding techniques: [erasure-coding-survey.md](./erasure-coding-survey.md).
