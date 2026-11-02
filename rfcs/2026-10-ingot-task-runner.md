# RFC: Ingot task runner

Status: Experimental

## Authors

- [Alan Shaw](https://github.com/alanshaw)

The idea:

A set of durable task queues, each serialized on the property whose actions must happen in serial. Aims to reduce the complexity in code, replace the bespoke database locks with a single exclusivity mechanism, and replace the special-purpose sweepers with delayed tasks.

Durable - task submission is recorded to postgres so that tasks may be resumed on crash/restart. Delayed work (a release after the reader grace period, an abort a week later) survives a restart, and a task's effects commit with the tasks it spawns, so a crash between the two cannot leak blobs. Every task is durable, sync ones included, so no handler has to decide whether it needs to be.

Awaitable - async tasks can be awaited by queue and task ID.

### Locks this replaces

Ingot keeps concurrent writes consistent today with 10 separate mechanisms of 7 kinds: 3 Postgres advisory locks, 1 `FOR SHARE` row lock, 2 in-process mutexes, in-memory spool holds, in-process ownership of the multipart acceptors, 1 lease table renewed by live requests, and 1 release protocol with its own ordering rules. They are spread across 4 packages, and each has its own rules about what must be taken first and what must be checked under it. With the runner, 1 locking mechanism remains: a lease per subject. The full list is in [Appendix: Locks this replaces](#appendix-locks-this-replaces).

## Task Definition

```go
type TaskID = uuid.ID

type Task struct {
  ID        TaskID
  Subject   string     // the thing this task pertains to and is serialized on
  Action    string     // different actions that might be done with this subject
  Created   time.Time  // temporal ordering
  NotBefore *time.Time // when this task may be run after
  Expires   *time.Time // the latest time the task may be run
  Node      *string    // the instance that must run this task; nil runs on any
}
```

### One table per queue

Each queue has its own table of tasks and leases, so its indexes, vacuuming and contention follow its own workload. A spawn into another queue still commits in the spawner's transaction, since the tables share a database. Awaiting a task names its queue and ID.

### Arguments and proofs

The runner reads only the fields of `Task`. Everything else on a task type is that action's arguments: the runner stores them as opaque bytes (Ingot encodes them with cbor-gen), hands them to the handler registered for the (queue, action), and never interprets them. Nothing in the runner knows about Ingot, so it could be released as a library.

Actions that invoke the network, directly or through a task they spawn, carry a `Proofs` argument: the delegation chain the invocation is issued with. A task passes its proofs on to the children that need them.

### Expiry

`Expires` is the latest time a task may run. After it, the task fails without running and is recorded as expired. Ingot uses it for authority: a task carries the proofs of the request that scheduled it, and those must last past its `NotBefore` (a cleanup 24 hours after an upload, the abort of a multipart upload abandoned for a week). See [Trade-offs and future work](#trade-offs-and-future-work) for how long they last today.

### Pending tasks

#### Replacing

A task with a `NotBefore` can be submitted as a replacement: if a task with the same subject and action is pending, the one with the later `NotBefore` is kept and the other is dropped. A task with no `NotBefore` is never a replacement and leaves a pending one alone. Ingot uses this to extend a cached blob's lifetime: each read or new copy of the blob pushes its pending eviction later.

#### Cancelling

A task can cancel a pending task by queue and ID. The cancel joins the caller's transaction, so it commits or fails with the caller's own writes. Cancelling a task that has already started, finished or been cancelled does nothing, so a handler whose task can be cancelled still checks its own preconditions when it runs. Ingot uses this when a write commits, to cancel the cleanup an upload scheduled in case the write never happened.

### Batching

A handler can be registered as batching, with a limit. One execution then takes the oldest due task on a subject plus the due tasks after it with the same action, stopping at the first with a different action so the subject's order holds. A batch commits or fails together. Only async tasks batch; extending it to sync tasks is deferred (see [Trade-offs and future work](#trade-offs-and-future-work)). Ingot uses it to send a space's queued object registrations to the network in one request.

### Retries

An action registers with a retry policy for when it runs async: a backoff (first delay, maximum delay, multiplier) and an optional attempt limit. A sync task is never retried: it fails its caller, and the S3 client retries the request. A handler can mark an error as permanent (an abort refused because the blob was accepted, an expired proof) to stop the retries. A task that fails permanently or runs out of attempts is recorded as failed and logged, and monitoring shows it by queue and action.

### Placement

A task that touches one instance's local state is placed there with `Node`, and only that instance runs it. Ingot runs as one instance, so this changes nothing yet; it is there for a future multi-instance design (see [Trade-offs and future work](#trade-offs-and-future-work)).

## Runner rules

### The runner guarantees

1. **One consumer per subject.** At most one task per (queue, subject) runs at a time. Among a subject's tasks that are due (no `NotBefore`, or one that has passed), the oldest by `Created` runs first. A task waiting on its `NotBefore` never holds up the tasks behind it. Exclusivity is a lease row per (queue, subject) in Postgres (owner, expiry, renewed while running), so it holds across processes.
2. **Effects and spawns commit together.** A task's Postgres writes, its result, its completion, the rows of the tasks it spawns and its sync children's writes all commit in one transaction. A task and its sync children succeed or fail together, so a resumed task re-runs its children from scratch. The transaction opens only after every handler in the tree has run, so none stays open across a network call. Until then, handlers add their writes to an ordered list, which runs in that transaction in the order the handlers ran; a write that only makes sense if an earlier one held (claiming a reference, say) is a conditional statement, and one that affects no rows fails the whole tree. Every subject in the tree stays leased until the commit, so reads made before it are safe. A top-level task's row commits when the task is submitted, before it runs, so a crash while it runs leaves a task to resume. A sync child's row commits with its parent. If the final transaction itself fails, the task is not marked failed: it stays as submitted and is resumed when its lease expires, as after a crash. Only an error a handler returns marks a task failed.
3. **Sync children run inline.** A sync child runs in its parent's goroutine once its subject's lease is taken. It does not wait for a worker. A sync task submitted by an S3 handler runs in the handler's goroutine the same way. Worker pools only run async tasks and tasks taken over after a lease expiry.

### Handlers must

4. **Be idempotent.** Effects outside Postgres (file renames, log fsyncs, network invocations) happen before the transaction in rule 2 commits, so a crash re-runs them. Every handler must tolerate running again after an effect already happened.
5. **Sync-spawn only down the queue order, never up or across.** A task may sync-spawn only into a queue below its own in the order below. A task never sync-spawns onto a subject of its own queue. Async spawns may go anywhere.
6. **Not recurse.** No action spawns, directly or through children, a task with its own action. This is documented, not enforced.
7. **Not read their own tree's writes.** A tree's writes reach Postgres only when it commits ([rule 2](#the-runner-guarantees)), so a handler decides from what it read before the tree wrote anything and from its children's results, and never re-reads a row the tree has written. The runner cannot detect a breach, which shows up as a stale read, so a test helper runs each tree twice, once with writes held to the end and once with them applied straight away, and checks both give the same result.
8. **Fail before acting.** A handler makes its checks, and returns any error it is going to return, before its first effect outside Postgres. After that, it fails only for an error a retry could never get past, such as an expired proof. A task then either fails having changed nothing outside Postgres or ends up committed, since a failed commit is resumed ([rule 2](#the-runner-guarantees)).

Queue order (sync spawns only go down):

1. [`queue-multipart-upload`](#multipart-upload-queue-queue-multipart-upload)
2. [`queue-multipart-upload-shard`](#multipart-upload-shard-queue-queue-multipart-upload-shard)
3. [`queue-bucket`](#bucket-queue-queue-bucket)
4. [`queue-space-upload`](#space-upload-queue-queue-space-upload)
5. [`queue-space-shard`](#space-shard-queue-queue-space-shard)
6. [`queue-shard`](#shard-queue-queue-shard)

Rule 5 is the lock-ordering discipline Ingot already follows (commits take their claim-key locks sorted by digest so two commits sharing digests cannot deadlock). The difference is that it is now stated once for every queue, which matters most where porting today's code would break it: see the release check under [`queue-space-shard.release`](#queue-space-shardrelease).

---

## How S3 requests use the queues

What happens in the handlers in terms of interactions with the queues. Indented tasks are spawned by the task above them; a delayed async task says when it runs.

* **sync**: the caller waits for the task and gets its result. The task runs in the caller's goroutine once its subject's lease is free ([rule 3](#the-runner-guarantees)), its writes commit with the caller's ([rule 2](#the-runner-guarantees)), and if it fails, the caller fails.
* **async**: the caller records the task and carries on. Its row commits with the caller's writes (or straight away, when the caller is an S3 handler), so it is never lost, and a worker runs it later: as soon as its subject is free, or once its `NotBefore` passes. The caller does not wait for it, and its failure does not fail the caller; the runner retries it under the action's retry policy.

### `CreateBucket`

1. (sync) [`queue-bucket.create`][bucket-create]

### `DeleteBucket`

1. (sync) [`queue-bucket.delete`][bucket-delete]
    * (sync) [`queue-space-shard.release`][space-shard-release] - each of the space's pending releases, ignoring `NotBefore`
        * (async) [`queue-shard.evict`][shard-evict]

### `PutBucketVersioning`

1. (sync) [`queue-bucket.set-versioning`][bucket-set-versioning]

### `PutObjectLockConfiguration`

1. (sync) [`queue-bucket.set-object-lock-config`][bucket-set-object-lock-config]

### `PutBucketTagging`, `DeleteBucketTagging`, `PutBucketCors`, `DeleteBucketCors`

1. One update of the bucket's row, with no task. These settings constrain nothing else in the bucket, so nothing needs to serialize with them.

### `PutBucketPolicy`, `DeleteBucketPolicy`

1. Sent to Hilt, which stores the policy. No task.

### `PutObject`

1. Stream the body, split into shards, uploading each to the provider as its bytes arrive and, with the cache on, writing it to a temporary file. If a stream fails, abort its allocation, delete its temporary file and fail the request. For each shard, once its digest is known and its body is sent:
    * (sync) [`queue-space-shard.add`][space-shard-add] with `Accept: true` and the temporary file as `Path`
        * (sync) [`queue-shard.cache`][shard-cache] - once accepted, with the cache on
            * (async, at the TTL) [`queue-shard.evict`][shard-evict]
        * (async, at `Deadline`) [`queue-space-shard.expire`][space-shard-expire] - cancelled when the reference is claimed
            * (async, after the reader grace period) [`queue-space-shard.release`][space-shard-release] - when the space's references reach zero
                * (async) [`queue-shard.evict`][shard-evict]
1. (sync) [`queue-bucket.commit`][bucket-commit] with a `Put` change - claims the shards' references
    * (async) [`queue-bucket.ship`][bucket-ship] - when the log's open segment is sealed
    * (async) [`queue-space-shard.remove`][space-shard-remove] - shards of a superseded version (unversioned bucket)
        * (async, after the reader grace period) [`queue-space-shard.release`][space-shard-release] - when the space's references reach zero
            * (async) [`queue-shard.evict`][shard-evict]
    * (async) [`queue-space-upload.add`][space-upload-add]
    * (async) [`queue-space-upload.remove`][space-upload-remove] - root of a superseded version

### `GetObject`

1. For each shard, read the cached file if there is one; with less than half its TTL left:
    * (async) [`queue-shard.cache`][shard-cache] with an empty `Path`
        * (async, at the TTL) [`queue-shard.evict`][shard-evict] - replacing the pending one
1. Otherwise stream the shard from the network. With the cache on and the whole blob read, write it to a temporary file as it streams and verify its digest, then:
    * (async) [`queue-shard.cache`][shard-cache]
        * (async, at the TTL) [`queue-shard.evict`][shard-evict] - replacing any pending one

### `CopyObject`

1. Within the space: (sync) [`queue-bucket.commit`][bucket-commit] with a `Copy` change
    * (sync) [`queue-space-shard.add`][space-shard-add] - each source shard, with no `Path`
        * (async, at `Deadline`) [`queue-space-shard.expire`][space-shard-expire] - cancelled when this commit claims the reference
            * (async, after the reader grace period) [`queue-space-shard.release`][space-shard-release] - when the space's references reach zero
                * (async) [`queue-shard.evict`][shard-evict]
    * (async) [`queue-bucket.ship`][bucket-ship] - when the log's open segment is sealed
    * (async) [`queue-space-shard.remove`][space-shard-remove] - shards of a superseded version (unversioned bucket)
        * (async, after the reader grace period) [`queue-space-shard.release`][space-shard-release] - when the space's references reach zero
            * (async) [`queue-shard.evict`][shard-evict]
    * (async) [`queue-space-upload.add`][space-upload-add]
    * (async) [`queue-space-upload.remove`][space-upload-remove] - root of a superseded version
1. Across spaces: handle as `PutObject`, with the source object's read stream as the body

### `DeleteObject`

1. (sync) [`queue-bucket.commit`][bucket-commit] with a `Delete` change
    * (async) [`queue-bucket.ship`][bucket-ship] - when the log's open segment is sealed
    * (async) [`queue-space-shard.remove`][space-shard-remove] - shards of the removed version (none for a new delete marker)
        * (async, after the reader grace period) [`queue-space-shard.release`][space-shard-release] - when the space's references reach zero
            * (async) [`queue-shard.evict`][shard-evict]
    * (async) [`queue-space-upload.remove`][space-upload-remove] - root of the removed version

### `DeleteObjects`

1. (sync) [`queue-bucket.commit`][bucket-commit] with one `Delete` change per key, each answered with its own result. Today's code commits each key on its own.
    * (async) [`queue-bucket.ship`][bucket-ship] - when the log's open segment is sealed
    * (async) [`queue-space-shard.remove`][space-shard-remove] - shards of each removed version
        * (async, after the reader grace period) [`queue-space-shard.release`][space-shard-release] - when the space's references reach zero
            * (async) [`queue-shard.evict`][shard-evict]
    * (async) [`queue-space-upload.remove`][space-upload-remove] - root of each removed version

### `PutObjectRetention`, `PutObjectLegalHold`, `PutObjectTagging`

1. (sync) [`queue-bucket.commit`][bucket-commit] with a `SetAttributes` change
    * (async) [`queue-bucket.ship`][bucket-ship] - when the log's open segment is sealed

### `CreateMultipartUpload`

1. (sync) [`queue-multipart-upload.create`][multipart-upload-create]
    * (async, after 1 week) [`queue-multipart-upload.abort`][multipart-upload-abort] - ignored once the upload is complete or aborted
        * (async) [`queue-space-shard.remove`][space-shard-remove] - each part shard, with `NoGrace`
            * (async) [`queue-space-shard.release`][space-shard-release] - when the space's references reach zero, with no grace period
                * (async) [`queue-shard.evict`][shard-evict]

### `UploadPart`

1. Stream the body, split into shards, uploading each to the provider as its bytes arrive. Nothing is written to disk. If a stream fails, abort its allocation and fail the request. For each shard, once its digest is known and its body is sent:
    * (sync) [`queue-space-shard.add`][space-shard-add] with `Accept: false`
        * (async, at `Deadline`) [`queue-space-shard.expire`][space-shard-expire] - cancelled when the reference is claimed
            * (async, after the reader grace period) [`queue-space-shard.release`][space-shard-release] - when the space's references reach zero
                * (async) [`queue-shard.evict`][shard-evict]
1. (sync) [`queue-multipart-upload.add-part`][multipart-upload-add-part] - claims the shards' references
    * (async) [`queue-multipart-upload-shard.accept`][multipart-upload-shard-accept] - submitted as a replacement
    * (async) [`queue-space-shard.remove`][space-shard-remove] - shards of a replaced part, with `NoGrace`
        * (async) [`queue-space-shard.release`][space-shard-release] - when the space's references reach zero, with no grace period
            * (async) [`queue-shard.evict`][shard-evict]

### `UploadPartCopy`

1. As `UploadPart`, with the source object's range, read through the decrypting read path, as the body. The source may be in any bucket of the tenant, and nothing is shared with it.

### `CompleteMultipartUpload`

1. (sync) [`queue-multipart-upload-shard.accept`][multipart-upload-shard-accept] - an ordinary task, never a replacement: concludes every shard still parked, and fails the request if one cannot be accepted
1. (sync) [`queue-multipart-upload.complete`][multipart-upload-complete] - checks every listed part is accepted
    * (sync) [`queue-bucket.commit`][bucket-commit] with a `Put` change - the parts already own their shards' references
        * (async) [`queue-bucket.ship`][bucket-ship] - when the log's open segment is sealed
        * (async) [`queue-space-shard.remove`][space-shard-remove] - shards of a superseded version (unversioned bucket)
            * (async, after the reader grace period) [`queue-space-shard.release`][space-shard-release] - when the space's references reach zero
                * (async) [`queue-shard.evict`][shard-evict]
        * (async) [`queue-space-upload.add`][space-upload-add]
        * (async) [`queue-space-upload.remove`][space-upload-remove] - root of a superseded version
    * (async) [`queue-space-shard.remove`][space-shard-remove] - shards of recorded parts not listed in the request, with `NoGrace`
        * (async) [`queue-space-shard.release`][space-shard-release] - when the space's references reach zero, with no grace period
            * (async) [`queue-shard.evict`][shard-evict]

### `AbortMultipartUpload`

1. (sync) [`queue-multipart-upload.abort`][multipart-upload-abort]
    * (async) [`queue-space-shard.remove`][space-shard-remove] - each part shard, with `NoGrace`
        * (async) [`queue-space-shard.release`][space-shard-release] - when the space's references reach zero, with no grace period
            * (async) [`queue-shard.evict`][shard-evict]

[bucket-commit]: #queue-bucketcommit
[bucket-create]: #queue-bucketcreate
[bucket-delete]: #queue-bucketdelete
[bucket-ship]: #queue-bucketship
[bucket-set-versioning]: #queue-bucketset-versioning
[bucket-set-object-lock-config]: #queue-bucketset-object-lock-config
[multipart-upload-shard-accept]: #queue-multipart-upload-shardaccept
[multipart-upload-abort]: #queue-multipart-uploadabort
[multipart-upload-add-part]: #queue-multipart-uploadadd-part
[multipart-upload-complete]: #queue-multipart-uploadcomplete
[multipart-upload-create]: #queue-multipart-uploadcreate
[shard-cache]: #queue-shardcache
[shard-evict]: #queue-shardevict
[space-shard-add]: #queue-space-shardadd
[space-shard-expire]: #queue-space-shardexpire
[space-shard-release]: #queue-space-shardrelease
[space-shard-remove]: #queue-space-shardremove
[space-upload-add]: #queue-space-uploadadd
[space-upload-remove]: #queue-space-uploadremove

---

## Queues

### Multipart Upload Queue (`queue-multipart-upload`)

Serialized on: `upload ID`

i.e. multiple runners but only one consumer per upload ID.

Parts are accepted on their own queue, `queue-multipart-upload-shard`, so an accept pass's network calls never hold up a `queue-multipart-upload.add-part`.

Actions:

#### `queue-multipart-upload.create`

```go
type CreateTask struct {
  Task
  Space    did.DID
  Key      string
  UploadID uuid.ID
  Proofs   []byte // passed to the spawned auto-abort
}

type CreateResult struct {
  AutoAbort TaskID // the spawned abort task that won't execute for 1 week
}
```

Spawns:

* (async) `queue-multipart-upload.abort` - with a `NotBefore` value that aborts the upload after 1 week when not completed. Error is ignored if upload no longer exists in an abortable state.

#### `queue-multipart-upload.add-part`

Adds or replaces a part for a multipart upload, and claims its shards' references. Ensures upload is not in complete/aborted state. The part's shards were added to the space (parked, referenced) before this runs, so it only writes the part row, and parts of one upload barely wait for each other. Of two uploads of the same part number, the later replaces the earlier and removes its shards.

```go
type AddPartTask struct {
  Task
  UploadID   uuid.ID
  PartNumber int
  Part       PartValue // etag, size, checksum, shard digests, span lengths
  Proofs     []byte    // passed to the spawned accept and removes
}

type AddPartResult struct {
  Replaced []multihash.Multihash // shards of the part this one replaced
}
```

Spawns:

* (async) `queue-multipart-upload-shard.accept` - with `NotBefore` set to now, submitted as a replacement, so an upload has at most one accept pending
* (async) `queue-space-shard.remove` - shards of a replaced part, with `NoGrace`: a part is never readable

#### `queue-multipart-upload.complete`

Assembles the object from the listed parts and puts it. CompleteMultipartUpload runs `queue-multipart-upload-shard.accept` as its own task first, so every part's acceptance is committed before this reads the parts ([rule 7](#handlers-must)). If a listed part is still parked, because it was uploaded again after that accept, Complete fails and the client retries it. Parts recorded but not listed in the request are dropped.

```go
type CompleteTask struct {
  Task
  UploadID uuid.ID
  Parts    []CompletedPart // part number + etag, from the request
  Proofs   []byte          // passed to the spawned commit and removes
}

type CompleteResult struct {
  VersionID string
}
```

Spawns:

* (sync) `queue-bucket.commit` - a `Put` change that adds the object to the bucket
* (async) `queue-space-shard.remove` - shards of recorded parts not listed in the request, with `NoGrace`: a part is never readable

#### `queue-multipart-upload.abort`

Marks the upload aborted and drops its parts. Error is ignored if the upload is already complete or aborted (the auto-abort).

```go
type AbortTask struct {
  Task
  UploadID uuid.ID
  Proofs   []byte // passed to the spawned removes; the auto-abort carries the create request's proofs
}

type AbortResult struct {
  ShardRemovals []TaskID
}
```

Spawns:

* (async) `queue-space-shard.remove` - each part shard, with `NoGrace`: an aborted part was never readable

Retries: no attempt limit, backing off from 1 second, doubling, up to 5 minutes. Dropping the auto-abort would leave an abandoned upload open, holding its parts' blobs.

### Multipart Upload Shard Queue (`queue-multipart-upload-shard`)

Serialized on: `upload ID`

i.e. one acceptor per upload, running apart from the upload's own queue.

Accepts a multipart upload's parked shards while the client is still uploading, so CompleteMultipartUpload has only the last few left to accept. A pass makes network calls and can run for minutes, so it has its own subject: on `queue-multipart-upload` it would hold up every `queue-multipart-upload.add-part` behind it. It stays consistent with that queue through the part rows, as the acceptor does today.

Actions:

#### `queue-multipart-upload-shard.accept`

Concludes every parked shard of the upload, in part order and in batches of 1,000 (one `/ucan/conclude`), until none is left, including shards parked while it runs. A part is marked accepted only if its row is unchanged since the batch started, so a part replaced or dropped meanwhile is left alone. Does nothing when nothing is parked, or when the upload is no longer open or completing. Replaces the per-upload acceptor goroutine and the sweeper that nudges it.

A failed batch fails the task, which records nothing ([rule 2](#the-runner-guarantees)). Run async, it is retried; run sync by CompleteMultipartUpload, it fails the request. Concluding a shard again counts as a repeat, so a retry only repeats round trips.

CompleteMultipartUpload runs this action as a top-level sync task before `queue-multipart-upload.complete`, so its writes are committed when Complete reads the parts. Submitted as an ordinary task, never a replacement, it runs after any accept already pending and concludes whatever is left.

```go
type AcceptTask struct {
  Task
  UploadID uuid.ID
  Proofs   []byte // for /ucan/conclude
}

type AcceptResult struct {
  Accepted int // shards accepted by this execution
}
```

Spawns: NONE

Retries: 5 attempts, backing off from 5 seconds, doubling. A later part's accept, or the accept CompleteMultipartUpload runs, concludes whatever is left; a few retries cover the case where neither comes before the provider expires the parked allocations.

### Bucket Queue (`queue-bucket`)

Serialized on: `space`

Serializes every change to the bucket's objects, its settings and its lifecycle. Every object write rewrites the bucket's one MST root, and what a write does depends on bucket state (the version ID and what it supersedes depend on versioning), so all of it is decided here, as today's code decides it under the per-bucket lock. The slow part of a write (streaming the body, uploading its shards, `queue-space-shard.add`) happens in the handler beforehand, so this subject is held for the commit itself. Commits and ships touch the bucket's catalog log on local disk, so they are placed on the instance that holds it.

Actions:

#### `queue-bucket.create`

Creates the bucket and its empty root.

```go
type CreateTask struct {
  Task
  Space did.DID
  Name  string
}

type CreateResult = Unit
```

Spawns: NONE

#### `queue-bucket.commit`

Applies a list of key changes in order: decides each one's effect from the bucket's state, splices them into the MST, appends the new catalog blocks to the bucket's log in one fsynced batch and swaps the root once. Every handler passes one change except `DeleteObjects`, which passes every key of its request, so up to 1,000 deletes cost one fsync and one root swap. Replaces the per-bucket in-process lock and the root CAS. Everything below is decided under this subject, so a concurrent `queue-bucket.set-versioning` lands entirely before or after a commit:

* the new version's sequence number and ID: a minted ID with versioning Enabled, the null version otherwise
* what it supersedes: nothing with versioning Enabled, otherwise the key's null version, whose references are removed and whose upload is retracted
* the write's preconditions (If-Match, If-None-Match), checked against the version it would supersede
* for a delete, whether it adds a delete marker or removes the version, and the refusal of a version under retention or legal hold
* for a put or copy, claiming the new version's pending references and cancelling their `queue-space-shard.expire` tasks

Each change gets its own result. A change that fails on its own terms (a failed precondition, a version under retention or legal hold) fails only that change, and the rest still apply. A failure of the commit itself, such as the fsync or the Postgres transaction, fails every change. A within-space copy reads its source under this subject too, so the source cannot change or disappear while the copy takes references to its shards.

```go
type CommitTask struct {
  Task
  Space   did.DID
  Changes []KeyChange // applied in order; one per key
  Proofs  []byte      // passed to the spawned adds, removes, upload registrations and ship
}

// KeyChange holds a key and exactly one change to it.
type KeyChange struct {
  Key           string
  Put           *PutChange
  Copy          *CopyChange
  Delete        *DeleteChange
  SetAttributes *SetAttributesChange
}

// PutChange adds a version from an upload or a completed multipart upload.
type PutChange struct {
  Value         ObjectValue
  Refs          []TaskID // pending references to claim; empty for shards a multipart part already owns
  Preconditions Preconditions
}

// CopyChange copies another key of the bucket. Metadata only: the new
// version references the source's shards. A copy across spaces re-ingests
// and is a PutChange.
type CopyChange struct {
  SourceKey     string
  VersionID     string // source version, empty for current
  Preconditions Preconditions
}

type DeleteChange struct {
  VersionID     string // empty for current
  Preconditions Preconditions
}

// SetAttributesChange sets a version's retention, legal hold or tags.
type SetAttributesChange struct {
  VersionID string
  Retention *Retention
  LegalHold *bool
  Tags      map[string]string
}

type CommitResult struct {
  Root    cid.Cid
  Results []ChangeResult // one per change, in order
}

type ChangeResult struct {
  VersionID    string // the new or affected version's ID
  DeleteMarker bool   // a delete added a marker
  Err          error  // this change's own failure; nil when it applied
}
```

Spawns:

* (sync) `queue-space-shard.add` - for a copy, each source shard, with no `Path`
* (async) `queue-space-shard.remove` - shards of a superseded or removed version
* (async) `queue-space-upload.add` - a new version's root, for a put or copy
* (async) `queue-space-upload.remove` - root of a superseded or removed version
* (async) `queue-bucket.ship` - when the log's open segment is sealed

#### `queue-bucket.ship`

Ships sealed catalog segments to the network.

```go
type ShipTask struct {
  Task
  Space  did.DID
  Proofs []byte // for the segment upload
}

type ShipResult struct {
  Segments []cid.Cid
}
```

Spawns: NONE

Retries: no attempt limit, backing off from 1 second, doubling, up to 5 minutes. An expired proof is permanent: the bucket's next write spawns a ship with fresh proofs. Dropping a ship any other way would leave an idle bucket's catalog only on local disk.

#### `queue-bucket.set-versioning`

Sets the bucket's versioning state to Enabled or Suspended. Refuses Suspended while the bucket has an object lock configuration. On this subject, the check cannot race a `queue-bucket.set-object-lock-config`, and no commit runs with the state half-changed. Today the two writes each check the other's setting and then update without a lock, so two concurrent requests can leave a lock configuration on a suspended bucket.

```go
type SetVersioningTask struct {
  Task
  Space did.DID
  State VersioningState // Enabled or Suspended
}

type SetVersioningResult = Unit
```

Spawns: NONE

#### `queue-bucket.set-object-lock-config`

Stores the bucket's object lock configuration. Refuses unless versioning is Enabled.

```go
type SetObjectLockConfigTask struct {
  Task
  Space  did.DID
  Config []byte // validated by the S3 layer
}

type SetObjectLockConfigResult = Unit
```

Spawns: NONE

#### `queue-bucket.delete`

Deletes an empty bucket, and fails if it holds objects or in-flight multipart uploads. It releases the space's blobs immediately, since an explicit delete owes readers no grace (the delayed releases then find nothing to do), then asks Hilt to delete the space. A re-run after Hilt already deleted it gets `UnknownBucket`, which counts as done.

Hilt authorizes the delete with the signed S3 request itself, so the task stores that request exactly as received, headers and signature included, and clears it once the task completes. A signed request can be replayed for that one operation while it is inside Hilt's 15-minute clock-skew window, so it is not kept longer than the task needs it. A delete resumed within the window finishes on its own. One resumed later is refused by Hilt and recorded as failed; the client, whose request failed with the crash, retries with a freshly signed request, and every step before the Hilt call is safe to repeat.

```go
type DeleteTask struct {
  Task
  Space   did.DID
  Request []byte // the signed S3 request, as Hilt verifies it; cleared when the task completes
  Proofs  []byte // passed to the spawned queue-space-shard.release tasks
}

type DeleteResult = Unit
```

Spawns:

* (sync) `queue-space-shard.release` - each of the space's pending releases, ignoring `NotBefore`

### Space Upload Queue (`queue-space-upload`)

Serialized on: `space`

Registers and retracts object roots with the network (`/upload/add`, `/upload/remove`), which is where the network's object count comes from. An add and a remove of one root must not reorder. Serializing on the space keeps them in order and stops one busy space starving another. Both actions batch, so queued registrations go out in one request, as the registration sweep does today.

Actions:

#### `queue-space-upload.add`

```go
type AddTask struct {
  Task
  Space  did.DID
  Root   cid.Cid
  Proofs []byte // for /upload/add
}

type AddResult = Unit
```

Spawns: NONE

Retries: no attempt limit, backing off from 1 second, doubling, up to 5 minutes. Dropping it would leave the space's object count wrong.

#### `queue-space-upload.remove`

```go
type RemoveTask struct {
  Task
  Space  did.DID
  Root   cid.Cid
  Proofs []byte // for /upload/remove
}

type RemoveResult = Unit
```

Spawns: NONE

Retries: no attempt limit, backing off from 1 second, doubling, up to 5 minutes. Dropping it would leave the space's object count wrong.

### Space Shard Queue (`queue-space-shard`)

Serialized on: `space + digest`

i.e. multiple runners but only one consumer per space + digest.

Serializes operations on a given space (bucket) and digest combination, ensuring that concurrent add/removes do not result in inconsistent network state. The reference count here is per space and covers object versions and multipart parts alike, so `queue-space-shard.release` never has to ask another queue whether something still uses the blob.

A reference starts out pending, identified by the `queue-space-shard.add` that took it, with a deadline. The write it is for (a `queue-bucket.commit` adding a version, or a `queue-multipart-upload.add-part`) claims it, which makes it owned, and cancels the reference's `queue-space-shard.expire` in the same transaction; the reference row records that task's ID. At the deadline, `queue-space-shard.expire` drops it if it is still pending, which cleans up after a request that died before its write. Both are conditional updates of one row, so whichever commits first wins without a lock, and a write that finds its reference expired fails.

The deadline is 24 hours after the add. Neither S3 nor Ingot limits how long a request may take, so the deadline only has to be long enough that a real upload never reaches it. 24 hours matches how long the provider keeps an unconcluded allocation, so an abandoned upload's pending references and its allocations are both gone within about a day.

Actions:

#### `queue-space-shard.add`

Adds a streamed shard to the space and takes a pending reference. With `Accept: true` (PutObject), it sends `/ucan/conclude` with the `/http/put` receipt, which triggers `/blob/accept`, and records the location. With `Accept: false` (UploadPart), it only records the shard as parked (with the provider, not yet accepted), and `queue-multipart-upload-shard.accept` concludes it later in a batch. For a within-space copy, which uploads nothing, it only takes the reference.

If a `queue-space-shard.release` of the same space and digest removed the blob after this upload's `Started` time, the add fails before it concludes anything ([rule 8](#handlers-must)), and so does the write; the release records when it removed the blob. This only happens without encryption: identical bytes then share a digest, the provider keeps one blob per digest in a space, and the upload cannot take its reference until the body has streamed and the digest is known. The client's retry then uploads normally.

`queue-space-shard.add` is the durable step of an upload. Before it, nothing records the allocation: a stream that fails while Ingot runs aborts its allocation inline, and one abandoned by a crash is never concluded, so the provider expires it.

```go
type AddTask struct {
  Task
  Space    did.DID
  Digest   multihash.Multihash
  Started  time.Time // when the upload's allocation was made; zero for a within-space copy
  Path     string    // temporary file to cache once accepted; empty with the cache off, for a part or a copy
  Accept   bool      // accept now (PutObject) or leave parked (UploadPart)
  Deadline time.Time // when the reference expires unless claimed: 24 hours after the add
  Proofs   []byte    // for /ucan/conclude when Accept is set, and passed to the spawned expire
}

type AddResult struct {
  Refs uint64 // references in the space, pending and owned
}
```

Spawns:

* (sync) `queue-shard.cache` - once accepted, when the add has a `Path`
* (async) `queue-space-shard.expire` - with `NotBefore` set to `Deadline`

A failed add deletes its temporary file.

#### `queue-space-shard.expire`

Drops the pending reference a `queue-space-shard.add` took, if it is still pending, then continues as `queue-space-shard.remove`. Its claim normally cancels it; if the claim commits while it is already running, it finds the reference owned and does nothing.

```go
type ExpireTask struct {
  Task
  Space  did.DID
  Digest multihash.Multihash
  Ref    TaskID // the queue-space-shard.add that took the reference
  Proofs []byte // passed to the spawned queue-space-shard.release
}

type ExpireResult = RemoveResult
```

Spawns:

* (async) `queue-space-shard.release` - as `queue-space-shard.remove`

Retries: no attempt limit, backing off from 1 second, doubling, up to 5 minutes. Dropping it would leave a reference nothing removes.

#### `queue-space-shard.remove`

Removes an owned reference. When the count reaches zero, spawns `queue-space-shard.release` of the blob from the network after the reader grace period. The release is submitted as a replacement: if the count reaches zero again while one is pending, the later grace period wins, so a reader of the newer version gets its full grace. Today's release queue keeps the later `not_before` for the same reason. A remove with `NoGrace` set spawns the release straight away instead, for a blob nobody could ever have read: a multipart part that was aborted, replaced, or left out of its Complete. A pending delayed release for the same blob then finds it gone and does nothing.

```go
type RemoveTask struct {
  Task
  Space  did.DID
  Digest  multihash.Multihash
  NoGrace bool   // release as soon as the count reaches zero: the blob was never readable
  Proofs  []byte // passed to the spawned queue-space-shard.release
}

type RemoveResult struct {
  Refs    uint64  // references remaining in the space
  Release *TaskID // set when Refs reached zero
}
```

Spawns:

* (async) `queue-space-shard.release` - when refs reach zero: with `NotBefore` set to the reader grace period, submitted as a replacement, or straight away with `NoGrace`

Retries: no attempt limit, backing off from 1 second, doubling, up to 5 minutes. Dropping it would leave a reference nothing removes.

#### `queue-space-shard.release`

Removes the blob from the space on the network: `/blob/abort` if it was parked and never accepted, `/blob/remove` if it was accepted. If a `queue-multipart-upload-shard.accept` races it, the provider decides: a released allocation refuses the accept, and an accepted blob refuses the abort, so the release removes it instead. It then deletes the blob's encryption params (crypto-shred) and location rows, records when it removed the blob (see `queue-space-shard.add`), and evicts the cached copy.

It does nothing if the space's references are above zero (a write reused the blob, so no cancellation is needed) or the space no longer holds the blob (a `queue-bucket.delete` got there first, or this is a re-run).

Today's release also checks whether an in-flight multipart part names the blob. Here a part holds a reference (see `queue-space-shard.add`), so that check is the reference count. Porting the check as a sync query into `queue-multipart-upload` would be a sync spawn up the queue order ([rule 5](#handlers-must)) and could deadlock against a `queue-multipart-upload.add-part` waiting on this subject.

```go
type ReleaseTask struct {
  Task
  Space  did.DID
  Digest multihash.Multihash
  Proofs []byte // for /blob/abort or /blob/remove
}

type ReleaseResult struct {
  Released bool
}
```

Spawns:

* (async) `queue-shard.evict` - with no `NotBefore`

Retries: no attempt limit, backing off from 1 second, doubling, up to 5 minutes. Dropping it would leave a blob nothing references on the network.

### Shard Queue (`queue-shard`)

Serialized on: `digest`

i.e. multiple runners but only one consumer per unique digest.

Serializes the cache's copy of a digest. Every task on this queue is placed on the instance whose disk holds the file. Ingot keeps a blob on disk only to cache it: an upload streams each shard to the provider as its bytes arrive (the trailer hash design) and never reads it back from disk. That makes streaming required:

* Sprue must support adding a blob by digest code. There is no store-and-forward path.
* A streamed PUT that fails fails the request, and the client retries it. There is no spool copy to upload again.

With the cache off, Ingot writes no blob to disk. With it on, `PutObject` writes each shard to a temporary file in the cache directory as it streams, and `queue-space-shard.add` moves it into the cache once the provider accepts it. A cache miss that reads a whole blob from the network fills the cache the same way, after verifying the digest. Multipart parts are not cached, as today.

A cached file has a TTL, held as a pending `queue-shard.evict`. Caching a blob again (another upload of the same bytes, a cache miss filled, a cache hit) extends it. A removed shard's `queue-space-shard.release` evicts the file straight away. In a non-encrypting bucket, that can remove a copy another bucket's object shares, which costs that bucket a cache miss and nothing else.

Actions:

#### `queue-shard.cache`

Moves a temporary file into the cache under its digest. If the cache already has the digest, the temporary file is deleted. No-op if the temporary file is gone (a re-run after the move). An empty `Path` only extends the TTL, for a cache hit.

```go
type CacheTask struct {
  Task
  Path string        // the temporary file, empty to only extend the TTL
  TTL  time.Duration // how long the file stays cached from now
}

type CacheResult = Unit
```

Spawns:

* (async) `queue-shard.evict` - with `NotBefore` set to now + `TTL`, submitted as a replacement, so a later eviction replaces a pending earlier one

A cache hit extends the TTL only when less than half of it remains, tracked in memory as the read retention is today, so most reads write no task.

Retries: none. A failed cache costs only a later cache miss.

#### `queue-shard.evict`

Deletes the cached file. No-op if there is none. A reader that already opened the file keeps reading it.

```go
type EvictTask = Task

type EvictResult struct {
  Evicted bool
}
```

Spawns: NONE

Retries: 3 attempts, backing off from 1 second, doubling. The disk budget sweeper takes whatever is left.

---

## Operations

### Sweepers

| Today | With the runner |
|---|---|
| Release sweeper | `queue-space-shard.release` with `NotBefore` |
| Stream sweeper (aborts a stream whose request died) | Not needed. A failed stream aborts its allocation inline; one abandoned by a crash is never concluded and the provider expires it within 24 hours. An unaccepted blob counts toward no space usage and is never served, so nothing needs to abort it sooner. References an abandoned request took are removed by `queue-space-shard.expire` |
| Multipart acceptor, and the accept sweeper that nudges it | `queue-multipart-upload-shard.accept` |
| Session expiry | `queue-multipart-upload.abort` with `NotBefore` |
| Upload registration sweep | `queue-space-upload` |
| Cache residency and read retention windows | The TTL on each cached file: a pending `queue-shard.evict`, extended by `queue-shard.cache` |
| Disk budget eviction | Stays a sweeper: a TTL does not bound the disk, and a burst of writes can fill it before anything expires. Over budget, it spawns `queue-shard.evict` with no `NotBefore` for the oldest cached files first |
| Orphan file scan | Stays a sweeper. Unfinished `.tmp-*` files belong to no subject |

The runner itself polls for due tasks (`NotBefore` passed, subject free) and for tasks whose lease expired. That poller is generic and is the only one besides the two above.

### Monitoring

Pool and queue sizes are deployment settings, and the right values depend on the workload. The runner reports enough to tune them, through OpenTelemetry, labelled by queue and action, so the library stays agnostic to what the actions do:

* queue depth: tasks ready to run, and tasks waiting on `NotBefore`
* age of the oldest ready task
* time a task waits for its subject's lease
* task duration and outcome (succeeded, failed, retried, expired)
* pool utilization: busy and idle workers per pool
* lease expiries, i.e. tasks taken over from a consumer that stopped renewing

---

## Trade-offs and future work

1. **Group commit (deferred).** Each object write holds `queue-bucket` for its own commit, so a bucket commits one write at a time, the same ceiling as today's per-bucket lock. Batching commits (one splice, one fsync, one root swap for many writes) is possible later by extending batching to top-level sync tasks, which `queue-bucket.commit` already is for every write except Complete's. A commit already carries a list of changes with a result for each, so a batch would apply every task's changes and hand each task its own results; its limit should count changes, not tasks. It adds real complexity to the runner, so revisit only if a bucket hits the ceiling.
2. **Multiple instances.** Leases make subject ownership work across processes, but two things stay on one instance's disk. The cache is the smaller one: another instance only misses it, and its tasks are placed on the instance holding the file. The bucket's catalog log is the blocker. Until a segment ships, the catalog blocks a bucket's root depends on exist only on the disk of the instance that committed them, so no other instance can commit to the bucket or read its newest objects, and losing that disk loses those blocks. Several instances need bucket ownership (a lease per bucket, with requests routed to its owner), catalog blocks made durable beyond one disk before a commit is acknowledged, and a way to wake a sync caller waiting on a task that runs in another process. Placement is in the runner already, so commits and ships can follow the bucket's owner.
3. **Write volume.** A `PutObject` of N shards writes about 4N + 2 task rows before it returns with the cache on (2N + 2 with it off): per shard a `queue-space-shard.add`, `queue-space-shard.expire`, `queue-shard.cache` and `queue-shard.evict`, plus a `queue-bucket.commit` and a `queue-space-upload.add`. Superseding a version or sealing a log segment adds a few more. Every task is durable on purpose (see the intro). Letting a request's inline sync tasks, other than `queue-space-shard.add`, skip their rows would roughly halve this, at the cost of a per-action durability decision that fails silently when wrong. Revisit only if row volume shows up in profiling.
4. **Authority for delayed cleanup.** A cleanup task invokes the network with the proofs of the request that scheduled it, and two things cut those short. Hilt's delegations last until the next UTC midnight plus a 4-hour overhang for async work, which covers a release after the reader grace period but not a `queue-space-shard.expire` 24 hours after an upload, nor an auto-abort a week after its upload was created. And they are signed by the caller's access key, so they also end when that key expires, is deleted or is rotated by a policy change; a longer lifetime alone would not help. The proposed fix ([fil-forge/hilt#113](https://github.com/fil-forge/hilt/issues/113)) is for Hilt to issue the cleanup commands (`/blob/remove`, `/blob/abort`, `/upload/remove`) per request, signed by its own issuer, with a lifetime past one week. Until then, a cleanup task whose proofs have lapsed fails permanently and is logged, so its blob or upload stays until something else cleans it up, as cleanup that outlives its authority does today.

---

## Appendix: Locks this replaces

| Today | Where | Replaced by |
|---|---|---|
| Advisory lock per (space, digest), the claim key, taken sorted by digest at every root commit | `registry/stores_postgres.go` (`lockClaimKeys`, `lockClaimKey`) | `queue-space-shard` |
| Release protocol run under the claim-key lock: `StartRelease`, `CancelRelease`, `GetRelease` and their ordering rules | `registry/stores.go` (`PendingReleaseStore`) | `queue-space-shard.release`, which re-checks the reference count and needs no cancellation |
| Advisory lock per digest, so exactly one space removes a shared local file | `registry/stores_postgres.go` (`DeleteIntentAndRelease`) | `queue-shard` |
| Advisory lock per (upload ID, part number) on a part write | `registry/stores_postgres.go` (put part) | `queue-multipart-upload` |
| `FOR SHARE` read of the multipart session row, against a concurrent Complete or Abort | `registry/stores_postgres.go` (put part) | `queue-multipart-upload` |
| In-process mutex per bucket, plus the compare-and-swap of the bucket root | `bucketop` (`Coordinator`) | `queue-bucket` |
| In-process ownership of each upload's acceptor goroutine | `s3frontend/acceptor.go` | `queue-multipart-upload-shard.accept` |
| In-process mutex per (space, digest), so two streams of the same bytes in one space never both record a location | `s3frontend` (`Backend.blobKeys`, taken in `concludeStreamed`) | `queue-space-shard` |
| In-memory spool holds per (space, digest), which stop a file a write is still uploading from being moved or removed | `blockstore/spool.go` (`Spool.holds`) | Nothing: an upload reads no local file, so there is no spool (see `queue-shard`) |
| Lease on `blob_streams` rows, renewed by the live request | `s3frontend/stream.go` | Nothing: an unconcluded allocation expires on the provider, and `queue-space-shard.add` is the upload's first durable step |

The migration lock in `migrations/migrations.go` stays: it serializes schema migrations at startup, not data.
