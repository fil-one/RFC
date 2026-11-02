# RFC: Erasure coding in Forge, a staged plan

Status: Experimental

## Authors

- [Hannah Howard](https://github.com/hannahhoward)

## Summary

Forge protects stored data today with ZFS inside a single disk enclosure. The products we want to sell next need more: data that survives losing an enclosure, then a data center, then a region, without paying for a full copy at every level. Erasure coding is how storage systems get that durability at a fraction of the cost of copies, and building it into Forge is a multi-quarter initiative.

We will build it in five stages, each adding a failure domain the data can survive: stage 1 ships by mid-November 2026, stage 2 by the end of 2026, Sovereign in Q1 2027, Global by the end of Q2 2027, and untrusted operators by the end of 2027. Stage 1 is broken down to user stories, stage 2 has a broad strategy, and later stages are named units of work. Many decisions are still open, and each has a stage by which it has to be settled. Please push on the order and the boundaries of the stages, and help pick off the open questions.

## Where we want to end up

Let's imagine the ultimate vision of this project as a multi-faceted user story:

> "As a user of FilOne, I want to be able to upload data to FilOne from anywhere in the world through a single endpoint, so I'm always uploading through the closest and fastest gateway no matter where my data ends up. I want my data to be stored durably in the places I choose, so that I can maintain sovereignty over my data. I want to be able to increase the resiliency of my data to disaster without paying for multiple full copies of data, so that my costs stay reasonable even when I need global resiliency. I want my data to be repaired any time a hard drive fails, so that ultimately a durable copy is never lost. I want to be able to retrieve my data from anywhere in the world through a single endpoint, so that I'm always downloading from the closest and fastest copies of my data stored. Ultimately, I want to be able to upload my data once and download it from anywhere in the world at comparable performance, so my business can operate anywhere."

We get there through three product tiers, rolled out in this order:

| Tier | Survives the loss of | Who it is for |
|---|---|---|
| **Local** | Hardware inside one data center | Neoclouds (newer GPU and AI cloud providers), and anyone who wants inexpensive storage close to their compute. This is effectively what we sell today. |
| **Sovereign** | A whole data center, with data kept inside one region or country | Buyers who need their data to stay in-country or in-region. Costs less than full copies only where a region has three or more data centers. |
| **Global** | A whole region | Media and other high-bandwidth streaming workloads. For them it works much like a CDN. |

Not every line of the user story applies to every tier. "Upload once, download anywhere at comparable performance" is the Global tier, and even there only where we run data centers. A single global endpoint is where every tier ends up, but for a long time Local and Sovereign customers will use a regional endpoint.

## Erasure coding in one page

Erasure coding splits a blob into **k** data pieces and computes **m** parity pieces from them, for **n = k + m** pieces in total. Any k of the n pieces are enough to rebuild the blob. Store the pieces on different disks and the blob survives the loss of any m of them.

The terms the rest of this RFC uses:

- **Piece:** one of the n chunks. Data pieces are the blob's own bytes; parity pieces are computed from them. Each piece is stored as an ordinary blob on a storage node.
- **Stripe:** one set of n pieces coded together, from one blob or one contiguous run of data. Any k pieces of a stripe rebuild it.
- **Scheme:** the recipe a stripe was coded with: the code (for example Reed-Solomon), k and m, and the exact details needed to produce the same pieces again. Keeping full copies is a scheme too.
- **Systematic code:** a code whose data pieces are the original bytes, unchanged, so a healthy read needs no decoding.
- **Stripe record:** the write-once record of a stripe's scheme and its n piece digests.
- **Placement:** which node holds each piece. It changes whenever repair moves a piece.
- **Failure domain:** hardware that can fail together: a disk, an enclosure, a data center, a region.
- **Expansion:** bytes stored divided by bytes of data, n / k.
- **Repair:** rebuilding lost pieces from k surviving ones and placing them on healthy nodes.

Take k = 8 and m = 4 as illustrative numbers. The blob is stored as 12 pieces at 1.5× its size, and any 4 pieces can be lost. Three full copies survive only 2 losses and cost 3×. Spread the 12 pieces across four enclosures, three to an enclosure, and losing a whole enclosure still leaves 9 pieces, one more than the 8 needed.

The same arithmetic sets the cost of each tier. To survive losing one of **s** failure domains, the data has to be spread so that the remaining s − 1 hold at least k pieces, which costs at least s / (s − 1). Across three data centers that is 1.5×; across two it is 2×, no better than keeping two copies. So the number of data centers in a region shapes the economics of the Sovereign tier directly.

Erasure coding has three costs:

- **Repair traffic.** Rebuilding one lost piece means reading k others. Inside a data center that is cheap. Across a WAN it is the dominant cost, which is why codes with local repair groups exist.
- **Small data.** Every piece is a separately stored, separately proven object. Cut a small blob into 12 pieces and the per-piece overhead swamps the savings. Small objects need a different path.
- **Bookkeeping.** Every stripe needs a stripe record and a placement. The stripe record is as important as the data, because without it the pieces cannot be reassembled.

The code we expect to start from is systematic Reed-Solomon. That choice is still open (see [Decisions to lock, and when](#decisions-to-lock-and-when)).

## The stages

The Forge components involved: ingot is the S3 gateway, which encrypts and will encode data; sprue is the upload service that chooses storage nodes; piri is the storage node, which proves possession on chain through PDP (proof of data possession); Hilt holds tenants and their keys. A head is the server that disk enclosures attach to.

Each stage adds one failure domain. Stage 5 adds a different axis, trust in the operators.

| Stage | Target | Survives loss of | Tier | Where redundancy lives | Repair | Location index |
|---|---|---|---|---|---|---|
| 1 | Mid-November 2026 | Disks within an enclosure | Local | ZFS, one pool per enclosure | ZFS | Ingot's Postgres |
| 2 | End of 2026 | An enclosure | Local | Erasure coding in ingot, across enclosures | Trusted, simplest possible | Ingot's Postgres |
| 3 | Q1 2027, aiming for mid-quarter | A data center | Sovereign | Erasure coding across data centers in a region | A separate service, across sites | Replicated across the region's sites |
| 4 | End of Q2 2027 | A region | Global | Erasure coding across regions | Global | Global (design open) |
| 5 | End of 2027 | An untrusted operator | Open network | As stage 4 | Plus reputation and audits | As stage 4 |

A tier is a placement policy chosen per bucket, and the hardware does not change with it. After stage 3 ships, the same sites serve Local buckets, whose pieces stay in one data center, and Sovereign buckets, whose pieces spread across data centers. Stripes written at stage 2 stay where they are, because Local is the only product we sell until stage 3. What carries forward from stage 2 is the software, the schema, and the placement hierarchy.

We build iteratively wherever a door is two-way, or "one and a half way": reversible with a data migration. "One-way door" in this RFC means a choice no migration can undo.

## Stage 1: Local, resilient to disk failure

Stage 1 involves no erasure coding. One ingot serves each data center, one piri serves each disk enclosure, and each enclosure runs a ZFS pool, likely with all of a site's piri processes on the same head. ZFS is the only durability: losing a disk is routine, and losing a whole enclosure loses the data on it. The difference from today is that a site runs several enclosures.

Much of this exists. Dev and staging already run three piri behind one ingot, each with its own key, wallet, data set and database ([FIL-1503](https://linear.app/filecoin-foundation/issue/FIL-1503)), and sprue already spreads writes across a site's piri. What production lacks is the operational shape.

**Story: Run a production site with several enclosures.** As FilOne operations, I want to bring up a site with one head and several ZFS enclosures, each served by its own piri, so that a site's capacity grows past one enclosure.

- A reference design documents the head, the enclosures, and the ZFS layout, and staging mirrors it.
- A platform deploy completes at the production piri count.
- Writes spread across a site's piri by available capacity, and a full piri stops taking writes.
- Every piri record carries its site and enclosure.

Tasks: the reference hardware and ZFS design; staging on the same layout; deploys that do not wait for every piri to be in a safe proving window at once; database and resource budgets per piri; a free-space admission check; fixing an off-by-one in the weighted node selector, which skews how writes spread at equal weights; flushing blob writes to disk before the database commits ([FIL-1611](https://linear.app/filecoin-foundation/issue/FIL-1611)); failure-domain labels on node records.

**Story: Add and retire enclosures without a reset.** As FilOne operations, I want to add an enclosure to a running site and to drain and retire one, so that sites grow and age without downtime.

- A new piri joins a running site, funded and registered, without resetting the site or central services.
- A piri can stop taking writes, hand its data off, and leave its data set.
- Wallet funding and balance alerts are automatic.

Tasks: add-a-piri tooling; drain-and-retire tooling; signed on-chain piece removal, which reverts today ([FIL-1657](https://linear.app/filecoin-foundation/issue/FIL-1657)) and blocks retiring anything that has been proven; automated wallet funding.

**Story: Know when an enclosure is in trouble.** As on-call, I want to be alerted by enclosure when a pool degrades, a disk fails, space runs low, or proofs fail, so that we act before customer data is at risk.

- Alerts fire for pool health, disk failure, free space ([FIL-1209](https://linear.app/filecoin-foundation/issue/FIL-1209)) and proving faults, labelled by enclosure.
- A runbook and a written durability statement cover losing a disk and losing an enclosure.

Stage 1 also carries groundwork that stage 2 depends on:

- **Measure object and part sizes** in production, by count and by bytes. Every small-data decision in stage 2 depends on this distribution.
- **Lower piri's aggregation threshold to 8 MiB** ([FIL-1656](https://linear.app/filecoin-foundation/issue/FIL-1656)). Piri folds blobs at or below the threshold into shared on-chain aggregates; the design intent is 8 MiB, but the shipped default is still 128 MiB. Erasure-coded pieces are meant to stand alone, so this plan assumes the 8 MiB floor and keeps pieces above it.
- **Design stage 3 and 4 location indexing.** Ingot's Postgres is currently the permanent record of where data lives, and it can serve stages 1 and 2. How that record spans sites and regions is a design question for later stages, and it should be started early.

## Stage 2: Local, resilient to enclosure loss

At stage 2, erasure coding moves into ingot above piri, and placement spreads each stripe's pieces across enclosures so the site survives losing a whole one. How piri maps to disks is open. One option runs enclosures as plain JBODs (disks presented individually, with no RAID or ZFS pool) with one piri per disk, grouped by enclosure: one layer of parity, finer failure domains, and dozens of piri per head. The other keeps one piri per enclosure on its ZFS pool and codes across enclosures on top: today's node shape and fewer piri, paying for ZFS parity underneath the code's. Repair is trusted and as simple as it can be. All traffic stays inside one data center, and there is no single global endpoint yet. Existing ZFS data stays on ZFS, so whole-blob ZFS piri and erasure-coded piri run side by side in a region and routing tells them apart. Whether that data ever moves is open.

The major units of work, with a broad strategy for each:

- **Encode and decode in ingot.** Ingot already encrypts each blob. The encoder works on that ciphertext, below the blob digest, so the catalog, key wraps and exit format stay unchanged. Healthy reads fetch the data pieces directly; reads that hit a missing piece decode from any k.
- **The stripe record.** A write-once descriptor names the code, its parameters and the n piece digests. Placement (which node holds which piece) is a separate, mutable table, because repair changes it. Both live in ingot's Postgres, currently the system of record, so the catalog database replication and backups now being built have to cover them.
- **Stripe registration.** Sprue registers stripes, never individual pieces, through the [multi-blob add protocol](https://github.com/fil-one/RFC/pull/29). That design needs three additions for erasure coding: a way to mark which blobs form one stripe so sprue spreads them across enclosures, replacement that avoids a failed piece's whole enclosure, and acceptance of the same digest at more than one node.
- **Failure-domain-aware placement.** Sprue spreads a stripe's pieces across enclosures using the labels stage 1 records.
- **If one piri per disk, sharing a wallet.** Today each piri has its own wallet and provider registration, and each registration burns 5 FIL. The contracts allow one registered provider to own many data sets, so a site's piri can share one wallet and registration while each keeps its own data set and database. What piri lacks is nonce coordination: each process tracks the wallet's transaction nonces in its own database, so two piri sending from one address can collide, and a dropped proving transaction stalls proving for that data set. The work is a nonce lock shared across a site's piri processes, with one piri settling payment rails for all of them. Running dozens of piri on one head also needs resource sizing.
- **Small objects.** Blobs below the coding threshold need their own path. With pieces held above 8 MiB and an illustrative k = 8, that threshold is around 64 MiB, so many objects fall below it. The simplest candidate is to store a small blob as several full copies across enclosures, recorded as a stripe with a "copies" scheme in the same stripe record. The approach is open until the size measurements and the research item below are in.
- **Multipart uploads.** Multipart is a special case: parts are small because of how they are uploaded, and the rest of the object is on its way. Instead of coding each part, ingot waits for a contiguous run of parts (64, 128 or 256 MiB), lays them end to end, and codes the run as one stripe. Each part keeps its own encrypted envelope. The tail left at Complete takes the small-object path. Open: how parts are protected while they wait (S3 clients treat a successful part upload as durable), how a stalled upload stops waiting, and the run size.
- **Detecting lost pieces.** On-chain PDP proves possession once a day on a few random leaves. It notices a dead node within about a day, and a single lost piece practically never. We need an off-chain challenge system, reporting to trusted verifiers, that checks pieces far more often and feeds repair. This is an open design.
- **Repair.** The simplest loop: find the affected stripes, read k surviving pieces, rebuild the missing ones, place them, update placement. It needs an index from node to the pieces it holds.

Stage 2 survives losing an enclosure. Ingot and its Postgres still run on the head, so losing the head is outside what stage 2 protects against, and the stripe record depends on that database's replication and backups.

## Stage 3: Sovereign, resilient to a data center

Stage 3 codes across data centers within a region. Each data center keeps its own ingot, and a regional endpoint routes customers to the closest one through geo-DNS or anycast. Placement gains a second level above the enclosure. Pieces for sites other than the writer's cross a WAN, and repair becomes its own service that tracks pieces across sites. Everything is still operated by us; there is no reputation system.

Units of work, all open designs at this point:

- **Key distribution across sites.** Ingot wraps each blob's content encryption key (CEK) with a key encryption key (KEK) held in the site's local OpenBao, as a non-exportable key. Another site cannot read that wrap. One possible flow: a site that lacks its own wrap sends the tenant-wrapped CEK to Hilt, which unwraps it with the tenant key, and the site wraps the CEK under its own KEK and stores that. Hilt has no unwrap service yet. Worth exploring: asymmetric site KEKs, like the tenant KEK, with the private half held in the site's OpenBao and the public half published. The writing site could then wrap each CEK for every site in the bucket's placement at write time, with no calls to anyone, and Hilt would re-encrypt to a site's public key on the rare recovery path, so an unwrapped CEK never crosses the wire. That amends the encryption RFC's choice of symmetric region keys, and depends on key-store support for a non-exportable asymmetric key and on the read-path cost of an asymmetric unwrap. Open: whether each data center or each region has its own KEK; whether wraps are made on first read, at write time, or in the background (a failover read after losing a site is a cold read, which argues for having wraps in place before it is needed); crypto-shred across every site's wraps; and whether Sovereign customers need tenant keys held in-region.
- **More than one ingot writing a bucket.** Ingot allows one writer per bucket today, through an in-process lock. With an ingot in each data center behind a regional endpoint, a bucket needs either multi-writer coordination or a home ingot that other sites forward to.
- **Multi-site regions in Hilt.** Today a region means one provider in Hilt, and each tenant is bound to one provider. Customer-facing names and ingot's identity are already region-scoped, so this is a schema and modeling change.
- **The regional location index.** Possibly Postgres replicated across the region's sites, one writer and two readers.
- **Cross-site repair.** Rebuilding one disk's pieces under a plain code reads mostly over the WAN. Codes with local repair groups keep disk repairs on site; this is part of the code research below.
- **Writes when central services are unreachable.** Every blob write goes to central sprue today. A region that should survive losing a data center should not depend on reaching another continent to write.
- **Possibly on-chain PDP for detection across sites,** alongside the off-chain system from stage 2.

## Stage 4: Global, resilient to a region

Stage 4 codes across regions and delivers the full user story: write once, read anywhere, through one global endpoint. It needs a global location index; whether that is a redesigned indexing service, a globally distributed SQL database, or something else is open. Global read-after-write is the hard part, and the IPNI-based indexing we inherited never solved it. Repair works across regions. Performance is comparable everywhere we run a data center. Everything is still trusted.

## Stage 5: Untrusted operators

Stage 5 opens storage to nodes we do not operate, which brings reputation, audits, admission and the economic machinery that comes with them. The earlier stages should not close it off.

## Decisions to lock, and when

| Decision | Door | Settle by |
|---|---|---|
| Core erasure code. Research production practice and the literature before assuming Reed-Solomon. Codes with local repair groups matter most at stage 3. The bar for anything other than systematic RS is a compelling gain plus production-grade library support usable from Go. | Two-way for new data if every stripe records its scheme | Stage 2 design |
| Codec output pinned per scheme: repair must reproduce byte-identical pieces, so the code, matrix and padding are fixed per scheme. | One-way for each stripe written | Stage 2 design |
| The stripe descriptor can name existing, independent blobs as its pieces, and full copies as a scheme. This keeps multipart runs and other grouping strategies open. | 1.5-way | Stage 2 design |
| Failure domains recorded as an ordered path (region, site, enclosure, disk), so a later stage adds a level instead of migrating. | Two-way while there is little data | Stage 1 |
| Small-object approach and threshold. Research the literature and industry practice, then set the threshold from stage 1 measurements. | 1.5-way | Stage 2 design |
| Multipart run size. How waiting parts are protected is a correctness requirement, settled before multipart runs ship. | Two-way | Stage 2 design |
| One piri per disk on JBODs, or one per enclosure on ZFS. With one per disk, a site's piri share a wallet and provider registration. | 1.5-way: a data set cannot change provider, so switching recreates data sets | Stage 2 design |
| Off-chain detection of lost pieces. | Two-way | Stage 2 design |
| Location index across a region's sites. | 1.5-way | Stage 3 design |
| KEK scope and distribution across sites. | 1.5-way: a rewrap of key rows | Stage 3 design |
| Multi-site regions in Hilt. | Two-way | Stage 3 design |
| Global location index. | 1.5-way | Stage 4 design |
| Durability target for each tier. | Hard to walk back once published | Before each tier is sold |
| Whether a Local bucket can move to Sovereign, by re-encoding it under the wider placement. | Two-way | Stage 3 design |

Customer-facing endpoints and ingot's identity stay named for regions, never for sites. They already are, and keeping them that way keeps stage 3 a two-way door.

## Alternatives considered

**Replicate everything.** Copies are simpler and need no stripe record. Surviving the loss of one data center takes two full copies, 2×, wherever they sit. Coding across two data centers costs the same 2×, but across three it costs 1.5× and across four about 1.33×, and the Sovereign and Global tiers depend on that saving.

**Erasure coding inside piri, or in a storage layer under it.** A layer under piri, such as ZFS, protects only the disks attached to one machine. Coding above the storage nodes, with ingot encoding and sprue placing pieces, spans every failure domain the stages add.

**Keep ZFS and add copies across enclosures at stage 2.** This survives an enclosure, but every byte pays for two full copies on top of ZFS's own parity, and it does not extend to stages 3 and 4.

## Coordination

We will share where we are headed on erasure coding, durability and repair with FilOz, the team behind the PDP and Filecoin Warm Storage Service contracts.

## References

- [Spike: How should Forge add erasure coding, and on what timeline?](https://linear.app/filecoin-foundation/issue/FIL-1556)
- [RFC: Forge multi-blob add](https://github.com/fil-one/RFC/pull/29)
- [RFC: FilOne encryption design](./2026-06-fil-one-encryption-design.md)
- [RFC: Regional security principles and key management](./2026-08-regional-security-and-key-management.md)
- [RFC: Credible exit under encryption](./2026-08-credible-exit-under-encryption.md)
- [RFC: Routing policies](./2026-08-routing-policies.md)
- [RFC: Keep Piri's state in Postgres and blob store](./2026-10-piri-state-in-postgres.md)
- [RFC: BLAKE3 object digest](./2026-10-blake3-object-digest.md)
