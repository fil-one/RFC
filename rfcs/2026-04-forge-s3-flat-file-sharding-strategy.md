# RFC: Forge S3 Facade sharding strategy

Status: Experimental

## Authors

- [Alan Shaw](https://github.com/alanshaw)

## Motivation

The Forge S3 Facade must support Multipart uploads via `UploadPart` as well as single part uploads via `PutObject`.

This is a proposal that will allow Forge to support this with minimal changes to the network, leaning on ideas previously proposed by the team:

* https://github.com/storacha/RFC/pull/65
* https://github.com/storacha/RFC/pull/66

## Design

The Sharded DAG Index will gain a `nodes` property - a list of UnixFS nodes "inlined" in the index (renamed from `blocks` as proposed in [#66](https://github.com/storacha/RFC/pull/66)).

For a given upload (a `PutObject` or `UploadPart` request) the S3 facade will shard data at the current threshold (256MB) and keep track of the CIDs of the shards that have been created as well as the order.

The order is important for retrieval, since we need to serve data from the shards in the order it appears in the object that is being uploaded.

When the `PutObject` request ends or the multipart `CompleteMultipartUpload` request is received, a UnixFS File root node is created that links to ALL shards in order.

A Sharded DAG Index is created, that contains entries for each shard. Each shard entry has a single slice entry which is the slice of the entire shard from byte 0 to (up to) 256MB.

The order in which the shards should be served is encoded in the root UnixFS node, which is added to the Sharded DAG Index `nodes` property.

To enable retrieval Guppy will need to be updated to consider the `nodes` property, and load the node(s) into it's own cache. Retrieval can then proceed as usual.

## References

* https://github.com/storacha/RFC/pull/65
* https://github.com/storacha/RFC/pull/66
