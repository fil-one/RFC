# Erasure coding techniques: a survey

19 November 2026

## Summary

Reed-Solomon is what almost every production storage system uses, and every other code is a way to make its repair cheaper. Rebuilding one lost Reed-Solomon piece means reading k others. The alternatives reduce that cost and pay for it in storage, loss tolerance or complexity.

- **Reed-Solomon** stores the least for a given loss tolerance and has mature, fast libraries. infectious is a Reed-Solomon library whose decoder also corrects wrong pieces, which Storj needs for untrusted nodes.
- **Locally repairable codes** add parity per small group, so a single loss is repaired from about half as many pieces. Azure runs one at 1.33×.
- **Regenerating codes** move fewer bytes per repair at Reed-Solomon's storage cost. They are complex and have little production use.
- **Fountain codes** suit transmission more than storage.
- **Red Stuff**, used by Walrus, codes in two dimensions so a node rebuilds its share by downloading only what it lost. It costs 4.5× and targets networks where a third of nodes may be dishonest.

For Forge, Reed-Solomon fits coding inside a data center. A locally repairable code is the alternative worth evaluating once stripes span data centers.

## How to compare codes

Every code here splits data into k data pieces and adds parity, for n pieces in total. Six properties separate them.

| Property | What it measures | Why it matters |
| --- | --- | --- |
| Storage overhead | Bytes stored per byte of data, n / k | The direct cost of durability |
| Fault tolerance | How many lost pieces a stripe survives | A code where any k of n pieces rebuild the data is called MDS (maximum distance separable). No code tolerates more losses for the same overhead. |
| Repair cost | Pieces read and bytes moved to rebuild one lost piece | Repair runs constantly at scale. Across a WAN it is the largest running cost. |
| Compute cost | Encode and decode throughput | Sets CPU per gigabyte on the write path and on degraded reads |
| Errors or erasures | Whether the decoder copes with wrong pieces, or only missing ones | Wrong pieces come from bit rot and from dishonest nodes. A digest per piece turns a wrong piece into a missing one. |
| Maturity | Years in production and quality of libraries | A code is a long commitment: every stripe written must stay decodable |

One more term appears below. Sub-packetization is the number of sub-chunks each piece is divided into internally. Reed-Solomon needs 1. Some repair-efficient codes need hundreds or thousands, which sets a minimum piece size and makes repair I/O more scattered.

## Reed-Solomon

Reed-Solomon (RS) is the baseline every other code is measured against. It is MDS, works for any k and n, needs no sub-packetization, and has decades of production use. Its weakness is repair: rebuilding one lost piece means reading k whole pieces.

The code treats k data pieces as defining a polynomial over a finite field and stores n points on it. Any k points recover the polynomial. In systematic form the first k pieces are the data itself, so a healthy read needs no decoding.

RS decoders come in two kinds. An erasure decoder needs to be told which pieces are missing, and rebuilds from any k good ones. An error-correcting decoder also finds pieces that are present but wrong. With r pieces received it can correct up to (r − k) / 2 wrong ones, at a higher CPU cost.

| Library | Language | Decoder | Piece limit | Notes |
| --- | --- | --- | --- | --- |
| [storj.io/infectious](https://github.com/storj/infectious) | Go, amd64 assembly kernel | Error-correcting (Berlekamp-Welch) | Not documented | Fork of [vivint/infectious](https://github.com/vivint/infectious). Storj's [uplink](https://raw.githubusercontent.com/storj/uplink/main/go.mod) pins v1.0.1. The README example uses 14 shares with 8 required, and recovers with 2 dropped and 1 corrupted. No published throughput figures. |
| [klauspost/reedsolomon](https://github.com/klauspost/reedsolomon) | Go, with SSSE3, AVX2, GFNI and ARM64 paths | Erasure only, plus a Verify call | 256, or 65,536 in Leopard mode | Over 1 GB/s per core in pure Go and several times that with SIMD. Streaming and progressive encoding. The README says to keep a hash per shard, because the decoder cannot tell which shard is wrong. |
| [catid/leopard](https://github.com/catid/leopard) | C++ | Erasure only | 65,536 | FFT-based, O(n log n) in the number of pieces, so it stays fast at high piece counts. The basis of the high-count mode in klauspost/reedsolomon. |
| [Backblaze JavaReedSolomon](https://www.backblaze.com/blog/reed-solomon/) | Java | Erasure only | 256 | MIT-licensed, written for Backblaze Vaults. klauspost/reedsolomon began as a Go port of it. |
| Intel ISA-L | C and assembly | Erasure only | 256 | The [default Ceph plugin](https://docs.ceph.com/en/latest/rados/operations/erasure-code/) for new pools from the Tentacle release. |

infectious earns its place where pieces cannot be checked one by one. Storj nodes are untrusted, and the decoder can discard a bad piece without a stored digest for it. Where a digest per piece already exists, a bad piece is rejected before decoding and an erasure decoder is enough.

## Locally repairable codes

A locally repairable code (LRC) adds parity pieces that each cover a small group of data pieces, so one lost piece is rebuilt from its group. Repair reads fewer pieces than Reed-Solomon. The price is that LRCs are not MDS: for the same overhead they survive fewer worst-case losses.

| Code | Layout | Overhead | Reads to repair one data piece | Losses survived |
| --- | --- | --- | --- | --- |
| RS(12,4), for reference | 12 data, 4 parity | 1.33× | 12 | Any 4 |
| [Azure LRC(12,2,2)](https://www.usenix.org/conference/atc12/technical-sessions/presentation/huang) | 12 data in 2 groups of 6, 1 local parity per group, 2 global parities | 1.33× | 6 | Any 3, and most patterns of 4 |
| RS(10,4), for reference | 10 data, 4 parity | 1.4× | 10 | Any 4 |
| [Xorbas LRC(10,6,5)](https://arxiv.org/abs/1301.3791) | RS(10,4) plus 2 local XOR parities | 1.6× | 5 | Any 4 |

The two designs spend the saving differently. Azure kept the overhead of RS(12,4) and halved repair reads, giving up the guarantee on a fourth loss. Xorbas kept all four losses and paid 14% more storage for about half the repair disk I/O and network traffic.

Azure's code is "maximally recoverable": it decodes every loss pattern that is decodable in principle for its layout. In the paper's smaller (6,2,2) example that is 86% of four-loss patterns.

LRCs suit multi-site layouts. Place each local group inside one site and a failed disk is repaired from pieces on the same site, while the global parities cover the loss of a whole site. Local repair helps only when a group has a single loss. Two losses in one group fall back to a global decode that reads k pieces.

Ceph ships an [LRC plugin](https://docs.ceph.com/en/latest/rados/operations/erasure-code-lrc/). A web search found no maintained pure-Go LRC library. The local parities can be built with any RS library, and the global parity coefficients need care to reach the maximally recoverable property.

## Regenerating codes

Regenerating codes reduce the bytes moved during repair. Each helper node sends a computed fraction of its piece, and the repairing node contacts more than k helpers. LRCs reduce the number of pieces read; regenerating codes reduce the bytes read from each.

[Dimakis and co-authors](https://arxiv.org/abs/cs/0702015) showed that storage per node and repair traffic trade against each other. The two ends of that trade have names:

- **MSR (minimum storage regenerating):** the same storage as an MDS code, with the least repair traffic possible at that storage. These are the codes of interest for object storage.
- **MBR (minimum bandwidth regenerating):** repair moves only as many bytes as the lost piece holds, and each piece is stored larger than MDS requires.

| Code | Kind | Result | Cost |
| --- | --- | --- | --- |
| [Clay](https://www.usenix.org/conference/fast18/presentation/vajha) | MSR, built by coupling layers of any MDS code | At 1.25× overhead, 2.9× less repair network traffic than RS. In the [Ceph example](https://docs.ceph.com/en/latest/rados/operations/erasure-code-clay/) with k = 8, m = 4, repairing a 1 GiB piece downloads 2.75 GiB from 11 helpers where RS downloads 8 GiB. | High sub-packetization: 1,024 sub-chunks for k = 16, m = 4 with 19 helpers |
| [Hitchhiker](https://research.facebook.com/file/237476094961438/publication00049_download0001.pdf) | RS(10,4) with extra parity functions "piggybacked" onto it | About 25% to 45% less network and disk I/O during reconstruction at the same storage as RS, measured in HDFS on a Facebook warehouse cluster | Sub-packetization of 2, and a smaller saving than MSR |

Three limits apply to the family. The saving is for single-piece repair, and several losses in one stripe fall back to an ordinary decode. Repair needs more than k helpers online at once. Sub-packetization sets a floor on piece size and spreads repair reads across many small ranges.

Production experience is thin. Clay's implementation is a C++ plugin inside Ceph, and current Ceph documentation marks that plugin deprecated and scheduled for removal.

## Fountain codes and other non-MDS codes

These codes give up the exact any-k-of-n guarantee for speed or simplicity. They were designed mainly for transmission and for disk arrays, and object stores rarely choose them.

| Code | How it works | Decoding guarantee | Where it is used |
| --- | --- | --- | --- |
| [LT and Raptor](https://en.wikipedia.org/wiki/Raptor_code) | Each piece is the XOR of a pseudo-random subset of the data. Raptor adds a pre-code to reach linear-time encoding and decoding. | Slightly more than k pieces, with high probability | Broadcast and streaming delivery |
| [RaptorQ (RFC 6330)](https://www.rfc-editor.org/rfc/rfc6330) | A systematic fountain code with up to 56,403 source symbols per block | Over 99% success with k pieces, over 99.99% with k + 1, over 99.9999% with k + 2 | Transport standards. Qualcomm has filed patent statements against the RFC. |
| [Tornado](https://en.wikipedia.org/wiki/Tornado_code) and [LDPC](https://en.wikipedia.org/wiki/Low-density_parity-check_code) | Sparse XOR equations in layers | Tornado needs a constant factor more pieces than RS | LDPC is the error-correcting code inside most SSDs |
| [RAID-6, EVENODD, RDP](https://en.wikipedia.org/wiki/Standard_RAID_levels#RAID_6) | Two parity pieces per stripe. The first is a plain XOR. | Any two losses | Disk arrays, including ZFS RAID-Z2 |

A fountain code is "rateless": the encoder can keep producing new pieces without limit. That helps a sender that does not know how many pieces will be lost in transit. A storage system fixes n when it writes, so the property goes unused, and the small chance of a failed decode is a durability risk RS does not carry.

Fountain codes stay fast when k runs to tens of thousands of pieces. FFT-based RS such as Leopard now covers that range too.

## Two-dimensional codes: Red Stuff

Red Stuff, the code in [Walrus](https://arxiv.org/abs/2505.05370), encodes a blob in two directions so that a node can rebuild its own share by downloading only as much as it lost. It stores 4.5× the blob to do so. It is designed for a network where up to a third of nodes may be dishonest.

Walrus runs n = 3f + 1 storage nodes and tolerates f dishonest ones. Encoding works as follows:

1. The blob is laid out as a matrix of symbols with f + 1 rows and 2f + 1 columns.
2. Each column is extended with Reed-Solomon from f + 1 symbols to n. The n rows that result are the primary slivers.
3. Each row of the original matrix is extended from 2f + 1 symbols to n. The n columns that result are the secondary slivers.
4. Node i stores primary sliver i and secondary sliver i, together called a sliver pair.

The layout for f = 1, which gives four nodes and a data matrix of 2 rows by 3 columns. `D` is a symbol of the blob itself and `p` is Reed-Solomon parity. Rows are primary slivers P1 to P4 and columns are secondary slivers S1 to S4. Node 2 stores row P2 and column S2.

```
                    S1   S2   S3   S4
Primary sliver P1    D    D    D    p     rows extended from 2f + 1 to n symbols
Primary sliver P2    D    D    D    p
Primary sliver P3    p    p    p
Primary sliver P4    p    p    p
                    columns extended from f + 1 to n symbols
```

A primary sliver is a row of 2f + 1 symbols and a secondary sliver is a column of f + 1 symbols. Each node holds one of each.

The two directions have different thresholds. Any f + 1 primary slivers rebuild the blob, which is the read path. A node missing its secondary sliver asks f + 1 other nodes for one symbol each. A node missing its primary sliver asks 2f + 1 nodes for one symbol each.

That last property is what the second dimension pays for. The paper puts single-node recovery at O(blob / n) bytes, where one-dimensional codes and full replication need O(blob). In a network with constant node churn, repair traffic stays proportional to the data lost.

| Property | Red Stuff in Walrus |
| --- | --- |
| Storage overhead | About 3× for primary slivers plus 1.5× for secondary, 4.5× in total. The [docs](https://docs.wal.app/docs/system-overview/red-stuff) add a fixed metadata term of 64 MB per blob at 1,000 shards. |
| Underlying code | Reed-Solomon over GF(2^16), using the Rust reed-solomon-simd library. The paper says the choice is not fundamental. |
| Integrity | A Merkle commitment per sliver and a blob commitment over those. A node that receives inconsistent slivers can prove the encoding was wrong. |
| Scale | 1,000 shards on mainnet |
| Paper's comparison | Full replication at 25×, one-dimensional erasure coding at 3× with whole-blob recovery, Walrus at 4.5× with proportional recovery |

Red Stuff's 4.5× follows from the trust model. Tolerating f dishonest nodes out of 3f + 1 caps k near n / 3, so even plain RS costs 3× there. Among operators who are trusted, RS or an LRC reaches the same failure tolerance at 1.3× to 1.5×.

[Celestia](https://docs.celestia.org/learn/how-celestia-works/data-availability-layer) also uses two-dimensional Reed-Solomon, for a different purpose. Block data in a k × k square is extended to 2k × 2k, with a Merkle root per row and column. Light clients sample random cells to check the data was published, and a wrongly encoded row or column can be proven with a small proof.

## Comparison

Reed-Solomon is the cheapest in storage and the most mature. Every alternative buys cheaper repair with extra storage, a weaker loss guarantee, or implementation risk.

| Code | MDS | Typical overhead | Repair of one piece | Sub-packetization | Go library | Production use |
| --- | --- | --- | --- | --- | --- | --- |
| Reed-Solomon, erasure decoding | Yes | 1.2× to 1.5× in data centers | Reads k pieces | 1 | klauspost/reedsolomon | Backblaze, HDFS, Ceph, MinIO, Facebook f4 |
| Reed-Solomon, error-correcting decoder | Yes | 2.8× to 3.8× in Storj | Reads k pieces | 1 | storj.io/infectious | Storj |
| LRC | No | 1.33× (Azure), 1.6× (Xorbas) | Reads one local group, about half of k | 1 | None found | Azure Storage, Ceph plugin |
| MSR regenerating (Clay) | Yes | 1.25× to 1.5× | Contacts more than k helpers, moves about a third of the bytes RS moves | Hundreds to thousands | None | Ceph plugin, now deprecated |
| Piggybacked RS (Hitchhiker) | Yes | 1.4× | 25% to 45% less traffic than RS | 2 | None | Evaluated at Facebook on HDFS |
| Fountain (RaptorQ) | No, needs slightly more than k | Any | Reads about k pieces | 1 | None in wide use | Transport protocols |
| XOR array codes (RAID-6 family) | Yes, for two parities | 1.1× to 1.5× | Reads the stripe | 1 | Not applicable | Disk arrays, ZFS |
| Two-dimensional RS (Red Stuff) | Per dimension | 4.5× | Moves about blob / n bytes, the size of what was lost | 1 | None, Rust only | Walrus |

Storj's overhead range comes from its default of 29 pieces required, 80 stored on a successful upload and up to 110 attempted.

## What production systems use

Almost every production system runs plain Reed-Solomon. Azure is the main exception inside data centers and Walrus the main one among decentralized networks.

| System | Code | Parameters | Overhead |
| --- | --- | --- | --- |
| [Storj](https://raw.githubusercontent.com/storj/storj/main/satellite/metainfo/config.go) | RS with error correction (infectious) | 29 required, repair at 35, 80 for a successful upload, 110 attempted | 2.8× at 80 pieces |
| [Sia](https://devs.sia.storage/docs/core-concepts/erasure-coding) | 10 data and 20 parity shards per slab | 10 of 30 | 3× |
| [Walrus](https://docs.wal.app/docs/system-overview/red-stuff) | Red Stuff (two-dimensional RS) | 1,000 shards | 4.5× |
| [Azure Storage](https://www.usenix.org/conference/atc12/technical-sessions/presentation/huang) | LRC | 12 data, 2 local, 2 global | 1.33× |
| [Backblaze Vault](https://www.backblaze.com/blog/vault-cloud-storage-architecture/) | RS | 17 data, 3 parity | 1.18× |
| [Facebook f4](https://www.usenix.org/conference/osdi14/technical-sessions/presentation/muralidhar) | RS(10,4) inside a data center, XOR of paired volumes in a third data center | 10 data, 4 parity | 1.4× per site, 2.1× with geo-redundancy |
| [HDFS](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSErasureCoding.html) | RS, with an XOR option | RS-6-3 by default; RS-3-2, RS-10-4 and XOR-2-1 built in | 1.5× by default |
| [Ceph](https://docs.ceph.com/en/latest/rados/operations/erasure-code/) | RS through ISA-L by default; LRC, SHEC and Clay plugins | Default profile k = 2, m = 2 | 2× by default |
| [MinIO](https://docs.min.io/community/minio-object-store/operations/concepts/erasure-coding.html) | RS | Erasure sets of 2 to 16 drives; 4 parity drives by default on a 16-drive set | 1.33× at 12 + 4 |

The decentralized networks pay 2.8× to 4.5× because they plan for nodes that leave or misbehave. Operators who run their own hardware sit between 1.18× and 1.5×.

Facebook f4 is the nearest published precedent for coding across sites cheaply. It keeps RS local to each data center, so disk repairs never cross the WAN, and adds one XOR volume in a third site to survive losing a data center.

## Fit to Forge's staged plan

Systematic Reed-Solomon with an erasure decoder fits the stages inside one data center. The choice of code becomes a real question at the stage that spreads a stripe across data centers, where repair first crosses a WAN.

| Stage in the [erasure coding plan](https://github.com/fil-one/RFC/pull/39) | Where repair traffic flows | Code that fits | Reason |
| --- | --- | --- | --- |
| Across enclosures in one data center | Inside the data center | Systematic RS, klauspost/reedsolomon | Reading k pieces is cheap on a local network. The stripe record already holds a digest per piece, so an erasure decoder is enough. |
| Across data centers in a region | Over the WAN for every disk failure under plain RS | RS, or an LRC with one local group per site | An LRC keeps single-disk repair on site. It adds local parity on top of the minimum overhead for surviving a site, and needs an implementation we would own. |
| Across regions | Over long-haul links | The same choice at larger scale | The f4 layout, RS per site plus one cross-site parity, is the simplest design with published production use, at about 2.1×. |
| Untrusted operators | Between parties who may misbehave | The same code, with stronger verification around it | Piece digests already turn wrong pieces into missing ones. |

Three codes look like poor fits. MSR codes such as Clay carry high sub-packetization, have no Go implementation, and Ceph is retiring its plugin. Fountain codes add a decode-failure probability for a property storage does not use. Red Stuff's 4.5× pays for tolerating a dishonest third of a large, churning network, which is a different problem from surviving a data center.

Two ideas from the decentralized systems carry over without their codes. Walrus commits to every sliver and lets a node prove an encoding was inconsistent, which matters once writers are untrusted. Storj repairs lazily, only when a segment drops to a threshold, which cuts repair traffic for any code.

Recording the scheme in every stripe keeps this choice open. Stripes written with RS stay readable if later stripes use an LRC.

## Sources

- [storj/infectious](https://github.com/storj/infectious), [vivint/infectious](https://github.com/vivint/infectious) and the [package documentation](https://pkg.go.dev/storj.io/infectious)
- [klauspost/reedsolomon](https://github.com/klauspost/reedsolomon) and [catid/leopard](https://github.com/catid/leopard)
- Storj [uplink go.mod](https://raw.githubusercontent.com/storj/uplink/main/go.mod), [satellite redundancy defaults](https://raw.githubusercontent.com/storj/storj/main/satellite/metainfo/config.go) and [file redundancy concepts](https://storj.dev/learn/concepts/file-redundancy)
- Walrus: [paper](https://arxiv.org/abs/2505.05370), [Red Stuff documentation](https://docs.wal.app/docs/system-overview/red-stuff) and [blog post](https://www.walrus.xyz/blog/how-walrus-red-stuff-encoding-works)
- Huang et al., [Erasure Coding in Windows Azure Storage](https://www.usenix.org/conference/atc12/technical-sessions/presentation/huang), USENIX ATC 2012
- Sathiamoorthy et al., [XORing Elephants: Novel Erasure Codes for Big Data](https://arxiv.org/abs/1301.3791), VLDB 2013
- Dimakis et al., [Network Coding for Distributed Storage Systems](https://arxiv.org/abs/cs/0702015)
- Vajha et al., [Clay Codes](https://www.usenix.org/conference/fast18/presentation/vajha), FAST 2018, and the [Ceph Clay plugin documentation](https://docs.ceph.com/en/latest/rados/operations/erasure-code-clay/)
- Rashmi et al., [A Hitchhiker's Guide to Fast and Efficient Data Reconstruction](https://research.facebook.com/file/237476094961438/publication00049_download0001.pdf), SIGCOMM 2014. The paper could not be opened; its figures here come from search summaries of it.
- Muralidhar et al., [f4: Facebook's Warm BLOB Storage System](https://www.usenix.org/conference/osdi14/technical-sessions/presentation/muralidhar), OSDI 2014
- [RFC 6330, RaptorQ](https://www.rfc-editor.org/rfc/rfc6330), and Wikipedia on [Raptor](https://en.wikipedia.org/wiki/Raptor_code), [Tornado](https://en.wikipedia.org/wiki/Tornado_code), [LDPC](https://en.wikipedia.org/wiki/Low-density_parity-check_code) codes and [RAID levels](https://en.wikipedia.org/wiki/Standard_RAID_levels#RAID_6)
- [Ceph erasure code documentation](https://docs.ceph.com/en/latest/rados/operations/erasure-code/), [HDFS erasure coding](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSErasureCoding.html), [MinIO erasure coding](https://docs.min.io/community/minio-object-store/operations/concepts/erasure-coding.html)
- Backblaze on [Reed-Solomon](https://www.backblaze.com/blog/reed-solomon/) and the [Vault architecture](https://www.backblaze.com/blog/vault-cloud-storage-architecture/)
- [Sia erasure coding](https://devs.sia.storage/docs/core-concepts/erasure-coding) and the [Celestia data availability layer](https://docs.celestia.org/learn/how-celestia-works/data-availability-layer)
