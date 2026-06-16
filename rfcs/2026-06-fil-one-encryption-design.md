# RFC: Fil One Object Encryption

**Status:** Design  
**Author:** Petra Jaros  
**Audience:** Forge + Fil One engineers  
**Date:** 2026-06-15

---

## TL;DR

Data we store in Forge is exposed to the Filecoin network, so it must be encrypted at rest. **Encryption is entirely Ingot's job: the S3 client hands Ingot plaintext, Ingot encrypts, and Forge stores opaque blobs it can't read.** Forge itself stays unaware that encryption is happening. It stores blobs, as it already does.

Each object is encrypted per [FEE](https://github.com/filecoin-project/FIPs/discussions/1253) (chunked AES-256-GCM, STREAM construction) under a fresh per-object content-encryption key (CEK). That CEK is then wrapped twice:

- **Tenant recipient**: Lives in the FEE envelope header, travels with the blob onto the network. Wrapped to a Hilt-custodied per-tenant key. This is the _insurance_ copy: Fil One can always recover the object, and can move it to a new region if necessary without altering the blob.
- **Region wrap**: Lives only in the region's local Ingot database. Wrapped to the region provider's key. This is the _read path_: it lets a region serve GETs with no synchronous call to any central Fil One service, and because it lives in a mutable store, it allows for cryptoshredding and provider key rotation.

This RFC is the design we intend to build under [FIL-273](https://linear.app/filecoin-foundation/issue/FIL-273). It's published to get holes poked in it before we write the code.

## Motivation

The requirements that shaped this design:

1. **Forge stays oblivious.** No changes to how Piri stores or proves blobs. Encryption is solely a concern of the layer in front of Forge.
2. **Regions serve reads independently.** A GET in a region should not require a synchronous round-trip to a central Fil One service for key material. Central availability should not gate regional reads.
3. **Fil One can always recover an object.** If a region loses its keys, the object is not lost (though the recovery process may not be available as a feature yet).
4. **Deletes should be fast.** When an object is deleted, its region keys are cryptoshredded. Removing the underlying blob bytes from the network is a separate, future mechanism; this RFC is about the keys. But even when we implement it, it will be slow, due to the nature of PDP. The first step to deleting is removing the entry from the bucket. The second is deleting the region key, truly preventing access. (Note that Hilt will still have a valid key. That's okay, because it's not on the read path. We can accept the low risk that that key will be compromised _and_ the data extracted through some means other than the normal read path before we can delete the actual blob.)
5. **Drop-in appliance.** The region manager is a Docker Compose appliance (or similar). Anything this design adds to the region has to fit that model.

## Hypothesis / Goals

We believe using **two key wraps**—one wrap in the blob for recovery, one wrap in the region's DB for serving—satisfies all five requirements, and that the FEE envelope format gives us the structure to express it without inventing a bespoke format.

Success looks like:

- An S3 PUT of plaintext stores an object that is opaque to Forge, Piri, and the network.
- An S3 GET in the region decrypts and serves it as plaintext with no synchronous central dependency.
- A DELETE renders the object unreadable in the region immediately.
- Fil One can (theoretically) recover any object's plaintext from the blob alone, proven by a test.
- The whole thing ships as part of the planned Compose (or similar) appliance with one new secret (the region key).

Non-goals for v1: removing blob bytes from Forge (future "true deletion"); customer-held keys / end-to-end encryption; executing tenant-key rotation (we build the groundwork, not the flow); server-side copy (deferred).

## Design

### Cryptographic core: FEE

We use [FEE](https://github.com/filecoin-project/FIPs/discussions/1253) (Filecoin Encrypted Envelope) as the on-disk format: chunked AES-256-GCM with the STREAM construction, 256 KiB chunks. STREAM gives us three things we need: streaming encryption at constant memory (we never hold a whole object in RAM), detection of truncation/reordering/tampering, and cheap range decryption (a byte range touches only the chunks it overlaps). The envelope is a `COSE_Encrypt` structure with a detached payload. The envelope header is small and self-delimiting, the ciphertext follows it, and the two are concatenated into the stored blob.

Each object gets a CEK (Content Encryption Key). The FEE envelope carries one recipient, the tenant wrap, in its header, since everything in the header travels with the blob onto the network. The _region_ wrap is not a COSE recipient and never enters the envelope; it is stored out-of-band as our own bytes in Ingot's DB. FEE sees a single-recipient envelope; the second wrap is purely an Ingot concern.

### Key identity: `kid` is a verification method ID

Every wrapped recipient in an envelope carries a `kid` (key ID). We make the `kid` **the ID of a verification method** in the tenant's DID document. This buys several things at once:

- **Self-describing envelopes.** Any envelope records exactly which key version wrapped it. Recovery is "read the `kid`, look it up, unwrap".
- **A clean slot for the future.** If we ever offer customer-held keys (true end-to-end), _that_ recipient's `kid` correctly anchors on the _customer's_ DID, because then the identity claim is true. Custodial and customer-held envelopes are then distinguishable at a glance by `kid` namespace, and FEE's multi-recipient model lets a single envelope carry both during a migration.

### Tenant DIDs are not `did:key`s

We've discussed tenant DIDs being `did:key`s so far, possibly just by default. However, tenant identities are long-lived, and `did:key`s can't be rotated. `did:web` would allow easy access to the rotatable keys in the DID document. Other DID methods (`did:plc`, for instance) are also available. In the long run, we can mix and match supported methods, if that's valuable.

### Write path (PUT)

1. S3 PUT arrives at Ingot; authorization validated.
2. Ingot generates a fresh CEK. In parallel:
   - Ingot streams the plaintext through FEE encryption, computing the **plaintext CID** inline as the bytes flow. Only the _ciphertext_ is buffered (to disk).
   - Ingot requests the tenant public key from Hilt (through a cache).
3. Ingot builds a FEE header with the tenant as the recipient (and thus containing the tenant-wrapped CEK).
4. Envelope blob (header ‖ ciphertext) is pushed to Forge via Guppy/Sprue.
5. Ingot asks its KMS to wrap the CEK with its own region key.
6. On successful push, a manifest row (including the region-wrapped key) is committed. On any failure, the buffer is discarded and nothing is committed.

Notes:

- Plaintext is never persisted to disk and never sent to Forge.

### Multipart

Each part is encrypted separately: its own CEK, its own FEE envelope, its own blob. An object is therefore an ordered manifest of segments; a plain single-part PUT is just a one-segment object, so single-part and multipart share all read-side code.

This makes parts independent, which is what S3 multipart needs (parts arrive in parallel, out of order, and can be re-uploaded). It also bounds buffer space to one part rather than one object, lets parts be pushed as they arrive, and makes `CompleteMultipartUpload` a pure atomic manifest commit. The cost is two cleanup obligations, since parts hit Forge _before_ the upload completes:

- **Abort / abandonment** must cryptoshred orphaned parts' key rows and queue their blobs for true deletion.
- **Superseded parts** (re-uploaded part numbers; last write wins at Complete) must shred the replaced part's key row.

However, multipart itself is a separate concern, and these operations are already built on a Forge-blob-delete operation. We just need to extend the Forge-blob-delete operation in the same way we do to implement an S3-object-delete.

### Read path (GET)

1. S3 GET arrives at Ingot; authorization validated.
2. Ingot looks up the object's parts. For each part, it asks its KMS to unwrap the region-wrapped CEK.
3. Ingot streams ciphertext from the local Piri, decrypts segment-by-segment in manifest order, and streams plaintext to the client. An auth-tag (tampered data) failure mid-stream terminates the response as an error.
4. **Range GET** maps the object range to segment spans via the manifest's cumulative plaintext sizes, then to chunk spans via the stored envelope byte lengths, fetches only the affected ciphertext chunks (byte-range retrieval from Piri), and decrypts only those.
5. **HEAD/List** report plaintext Content-Length and ETag from stored metadata (not the ciphertext or envelope sizes).

Notes:

- No data (plaintext or ciphertext) is buffered to disk, for security and for performance. Everything is streamed. (This is a feature, but not a requirement. We can choose to relax this in the future if, say, caching objects will result in better performance in some cases. But we should do so with care.)

There is **no Hilt dependency on the read path.**

If the region key is ever lost, recovery is: pull the envelope, unwrap its tenant recipient with Hilt's archived private key. There is no mechanism currently planned or specified to implement this, but it's cryptographically available.

### Deletion (cryptoshredding)

DELETE destroys every segment's **region-wrap row** in Ingot's DB. Subsequent GET/HEAD return `NoSuchKey`, and the blobs are queued for the future true-deletion mechanism. That's the cryptoshred: the region can no longer read the object, immediately.

**What survives:** the **tenant recipient in the envelope** still exists wherever the blob still exists on the network, and Hilt's tenant key still exists (it's shared across the tenant's objects, so it can't be destroyed per-object). So a deleted object remains recoverable _by Fil One_ until the blob itself is truly deleted. Because this is not a normal read path, this should be acceptable.

### The keys

| Name                      | Type                  | One Per                     | Location (Unencrypted)                 | Purpose                                                              |
| ------------------------- | --------------------- | --------------------------- | -------------------------------------- | -------------------------------------------------------------------- |
| Blob CEK                  | AES-256               | Blob                        | —                                      | Encrypts the blob contents. Constant across regions when replicated. |
| Region KEK                | AES-256               | Region                      | Region secure storage[^secure-storage] | Wraps the Blob CEK for the read path.                                |
| Region-KEK(Blob-CEK)      | A256KW Result         | Region × Blob[^replication] | Region Ingot DB                        | Unwrapped by Ingot to serve read requests.                           |
| Tenant KEK                | X25519                | Tenant                      | Hilt DB (public key only)              | Wraps the Blob CEK for the region-independent decryption.            |
| Tenant-KEK(Blob-CEK)      | ECDH-ES+A256KW Result | Blob                        | FEE Header at start of blob, in Forge  | Unwrapped by tenant for region-independent decryption.               |
| Hilt Root KEK             | AES-256               | Hilt (i.e., exactly one)    | FilOne secure storage[^secure-storage] | Seals the KEKs in the Hilt DB.                                       |
| Hilt-Root-KEK(Tenant-KEK) | A256KW Result         | Tenant                      | Hilt DB.                               | Unsealed by Hilt to unwrap keys as tenant.                           |

Key types:

- **[AES-256](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard)**: Symmetric, 256-bit key.
- **[X25519](https://en.wikipedia.org/wiki/Curve25519)**: Asymmetric, 256-bit key over Curve25519, built for key agreement. Analogous to Ed25519, an EdDSA signature scheme which uses a curve derived from Curve25519.
- **A256KW**: JOSE/COSE's term for the [AES-KW key-wrapping algorithm](https://datatracker.ietf.org/doc/html/rfc3394) using a 256-bit key (AES-256).
- **ECDH-ES+A256KW**: JOSE/COSE's term for a key-wrapping algorithm which uses _asymmetric_ keys. It performs key-agreement using an elliptic curve algorithm (here, X25519) to generate an AES-256 key, and then uses A256KW to wrap.

[^secure-storage]: "Secure storage" here refers to a secure secrets manager, such as Vault. It must be able to hold a non-exported key and wrap or unwrap as requested by the local process. Ideally, it wraps an HSM, but a fully-software equivalent is acceptable. This is a policy knob we can tweak according to our relationship with regions and our appetite for precaution.

[^replication]: Replication is not a current concern, so "Region × Blob" is currently the same as "Blob".

### The Region DB

This DB schema is loose and illustrative. It will need to merge with similar schemas currently being specified with other aspects of the system in mind. For instance, the actual table holding Objects will likely have several additional fields defined by other streams of work. The basic structure outlined here should mesh with those designs.

#### Objects

| Field     | Type   | Description                                                                                                   |
| --------- | ------ | ------------------------------------------------------------------------------------------------------------- |
| Object ID | CID    | The CID of the object, and the value of its bucket entry.                                                     |
| Size      | int    | Total plaintext size of object, in bytes. Calculated as sum of parts; may not need to be denormalized at all. |
| ETag      | string |                                                                                                               |

- Has many Parts
  - A non-multipart Object has exactly 1 Part.

#### Parts

| Field                    | Type   | Description                                                      |
| ------------------------ | ------ | ---------------------------------------------------------------- |
| Object ID _[Comp. PK]_   | CID    | The CID of the object to which the part belongs.                 |
| Part Number _[Comp. PK]_ | Int    | The number of the part within the object.                        |
| Blob                     | CID    | The CID of the stored, encrypted blob.                           |
| Size                     | int    | Total plaintext size of part, in bytes.                          |
| Envelope Length          | int    | Length of the FEE envelope (and offset of ciphertext), in bytes. |
| Blob CEK                 | bytes  | CEK of the blob, encrypted with the Region KEK.                  |
| ETag                     | string |                                                                  |

- Belongs to an Object

### The Hilt DB

#### Tenant

| Field     | Type | Description |
| --------- | ---- | ----------- |
| Tenant ID | DID  |             |

- Has many Tenant Keys
  - Permits Tier 1 tenant key rotation; see below.
  - One key must be marked as "current"; there are a couple of ways to do that, and the choice is left to the implementation.

#### Tenant Keys

| Field         | Type  | Description                                               |
| ------------- | ----- | --------------------------------------------------------- |
| Key ID _[PK]_ | URI   |                                                           |
| Tenant ID     | DID   |                                                           |
| Public KEK    | bytes | Tenant's X25519 Public Key                                |
| Private KEK   | bytes | Tenant's X25519 Private Key, encrypted with Hilt Root KEK |

- Belongs to a Tenant

### Rotation

#### Tenant

There are three levels of rotation available to protect the tenant key. We define them here as "tiers".

- **Tier 0: Rotate Hilt's at-rest protection.** Tenant private keys are stored in Hilt's DB encrypted by a single Hilt key for at-rest protection. If the DB is compromised, but the Hilt key (which is stored separately) is not, the attacker has no access to the tenant keys until they also obtain the Hilt key. If we rotate the Hilt key at this point, the old key no longer exists anywhere, and the attack is defeated. This rotation is relatively cheap, and invisible to the outside world: it simply requires re-encrypting the tenant keys in Hilt's own DB. _The key Hilt uses for this encryption is not used externally_ (eg, for issuing UCAN tokens), so rotation has no external cost.
  - **When:** On a regular schedule, and immediately in response to any DB exposure. Schedule can be implemented in the future.
- **Tier 1: Rotate the wrap key for _new_ envelopes (prospective).** Hilt publishes a new tenant key (eg., `#wrap-2`) and marks it as the current key. New blobs will use the new key. Old envelopes are untouched and stay valid. They self-describe their key via `kid`, and Hilt's still holds the matching private half. This bounds how much data any single key protects. It is _not_ revocation: a leaked old key still reads old envelopes until those blobs are re-encrypted and truly deleted.
  - **When:** On a regular schedule. Schedule can be implemented in the future. _Not_ in response to exposure—this limits future exposure, but does nothing to remediate an attack.
- **Tier 2: Re-encrypt existing blobs (retroactive).** Hilt creates a new tenant key as in Tier 1. Ingot walks the tenant's segments, decrypts each with the _region_ key (so plaintext never leaves the region, and Hilt isn't needed), re-encrypts under a fresh CEK, wraps with the new tenant `kid` and with the (unchanged) region KEK, pushes the new blob, atomically swaps the manifest entry, shreds the old region row, and queues the old blob for true deletion. Because objects are content-mutable in the bucket model (a key points at a current blob, tracked in DB or MST), this is just a self-overwrite.
  - **When:** Immediately in response to key exposure.

#### Region

Region key rotation is analogous to Tier 0 tenant rotation: a new region key is generated, and all wrapped keys in the region DB are re-wrapped under the new key.

### Where things live

- **FEE library**: A Go package (initially inside Ingot's module; promotable to its own repo if it's needed beyond Ingot). Envelope encode/decode, STREAM encrypt/decrypt, X25519 recipient wrap/unwrap, range decryption. Cross-checked against the TypeScript `foc-encryption` implementation with shared test vectors.
- **Hilt**: Tenant wrap-key registry and custody, wrap-context DID documents, key distribution to regions. Recovery-unwrap-as-a-service is future; v1 proves recoverability with a library-level test.
- **Ingot**: The object/segment schema (manifest + wrapped keys + plaintext metadata), the encrypting write path, the decrypting read path, deletion, and the region-key provider interface.

## Alternatives Considered

**Single wrap (tenant only), unwrap via Hilt on every read.** No region key, Hilt unwraps on cache-miss, short-TTL CEK cache. Rejected because it puts a synchronous central dependency on the read hot path (Hilt down means region-wide read failures) and a TTL cache is a weak mitigation that also smears plaintext key material across region memory on a timer. The region wrap removes the dependency entirely and a local unwrap is microseconds, so the cache had no reason to exist.

**Single wrap (region only).** Cheapest, but no recovery: a region that loses its key loses its customers' data permanently, and Fil One has no backstop.

**Asymmetric X25519 region and Hilt keypairs instead of symmetric AES keys.** These keys are only used to wrap values for storage at rest locally: the CEKs and the tenant keys, respectively. No other party needs to encrypt with these, so we don't need a public key. Using AES gives us better compatibility with key management systems (especially hardware security modules) which can hold a non-exportable key and en/decrypt for us on demand. Conversely, the _tenant_ key is used by the region for encryption, so we want a public key it can use, and to hold the (encrypted) private key in Hilt. (But: see Open Question #2.)

**Transcode multipart into one envelope at Complete.** Buffer parts under an ephemeral key, then re-encrypt into a single canonical FEE object at `CompleteMultipartUpload`. Rejected in favor of per-part envelopes: transcoding needs up-to-object-size buffer and an extra crypto pass, while per-part envelopes are naturally independent, bound buffers to one part, and make Complete a metadata-only commit. The cost (a manifest, plus abort/supersede cleanup) is bookkeeping we need anyway.

**`did:key` tenant identity.** `did:key`s do offer an X25519 `keyAgreement` verification method. But `did:key`s, by their nature, can't rotate their keys. The spec is clear that they shouldn't be used for long-term identity for exactly this reason. Whatever DID method we use should be able to rotate its keys.

## Open Questions

1. **Plaintext CID in the envelope's `app_metadata`.** This is an optional field in the FEE proposal. Putting it in exposes what the plaintext is to anyone who's seen that plaintext somewhere before and might know its CID. It also allows someone to correlate two encrypted blobs, confirming that they hold the same data without knowing what it is. Neither of these seem like particularly worrying attacks. On the other hand, there isn't a clear argument in _favor_ of putting it in.

   The plaintext CIDs are also in the bucket entries. If those remain in MSTs and we continue to plan not to encrypt the MSTs, we have plaintext CIDs hanging around in plaintext anyway.

2. **AES vs X25519 for the region key.** There is _one_ use case for the region key being an asymmetric keypair. If Ingot's wrapped keys were lost, or if a blob were moved or replicated to another region (not currently planned, but in the realm of things we might want), we'd need to restore Ingot's wrapped CEK. Under an asymmetric scheme, Hilt could encrypt the CEK with the region's public key and send it that. Is that enough to change this decision?

3. **One region key vs many.** This proposal describes a single key for the entire region. We can further limit the blast radius by keeping a key per tenant or per bucket. This complicates the interaction with the KMS slightly, but it's a reasonable tradeoff. We also get to decide this late (or change our minds), because this doesn't impact the envelopes at all. We can migrate by re-wrapping the Ingot DB's contents, just like in region key rotation.

4. **One key per tenant vs one key per bucket.** This proposal associates the canonical wrapping key with a _tenant_. We could use a different key per _bucket_ instead. That does a few things: it limits the blast radius of exposing a single key, it makes remediation of an exposure easier by limiting what we need to Tier-2 rotate, and it makes bucket transfer between tenants easier (which has not been planned or discussed). The downside is pretty much that there are more keys to manage. This one is harder to decide later, because it does impact the envelope, but as in Tier 2 rotation, we _do_ get a hand in the bookkeeping from the `kid` in the FEE header, so we'll never lose track of which key is correct. Still, better to pick this up front.

## Evaluation Criteria

Ideally, we'll be able to write Smelt tests around these criteria.

- **Correctness.** Round-trip PUT/GET/DELETE (single and multipart) against the Compose appliance; cross-part range GETs byte-exact; tampered ciphertext rejected; max-size (5 GiB) part handled; abort/supersede cleanup verified.
- **Opacity.** A blob fetched directly from Piri is undecryptable without the keys; the envelope discloses nothing we didn't intend (modulo, notably, plaintext CID).
- **Independence.** Reads in a region succeed with Hilt unreachable.
- **Recoverability.** A library-level test recovers plaintext from an envelope using _only_ the tenant recipient and an archived private key — no region key, no Ingot DB — proving the insurance before any recovery service exists.
- **Cryptoshred.** After DELETE, the region cannot serve the object; the surviving envelope copy is documented, not silent.
- **Rotation-readiness.** Every envelope and DB row carries a versioned `kid` from day one, and Hilt's registry can archive and retain old private keys — i.e., the groundwork is provably in place even though the rotation flows ship later.

## References

- [FEE: Filecoin Encrypted Envelope (FIP discussion #1253)](https://github.com/filecoin-project/FIPs/discussions/1253)
- [RFC 9052 — CBOR Object Signing and Encryption (COSE): Structures and Process](https://datatracker.ietf.org/doc/html/rfc9052)
- [RFC 9053 — COSE: Initial Algorithms](https://datatracker.ietf.org/doc/html/rfc9053) (ECDH-ES, A256KW, OKP/X25519)
- [RFC 7748 — Elliptic Curves for Security](https://datatracker.ietf.org/doc/html/rfc7748) (X25519)
- [RFC 8032 — EdDSA](https://datatracker.ietf.org/doc/html/rfc8032) (Ed25519, for the why-not-this aside)
- FIL-273 — Implement encryption for bucket contents (Linear epic; this RFC is its design artifact)
