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
  Queue     string
  Subject   string     // the thing this task pertains to and is serialized on
  Action    string     // different actions that might be done with this subject
  Created   time.Time  // temporal ordering
  NotBefore *time.Time // when this task may be run after; async tasks only
  Expires   *time.Time // the latest time the task may start
  Args      []byte     // the action's arguments, never read by the runner
  Attempt   int        // how many earlier runs failed and were retried
  Node      *string    // the instance that must run this task; nil runs on any
}
```

### One table per queue

Each queue has its own table of tasks and leases, so its indexes, vacuuming and contention follow its own workload. A spawn into another queue still commits in the spawner's transaction, since the tables share a database. Awaiting a task names its queue and ID.

### Arguments and proofs

The runner reads only the fields of `Task`. Everything else on a task type is that action's arguments: the runner stores them as opaque bytes (Ingot encodes them with cbor-gen), hands them to the handler registered for the (queue, action), and never interprets them. Nothing in the runner knows about Ingot, so it could be released as a library.

Actions that invoke the network after their request has gone carry the authority they need in their arguments, captured from the request that queued them: a release carries `ReleaseAuthority`, the request's `/blob/remove` and `/blob/abort` chains, and an object registration its `/upload/add` or `/upload/remove` chain. When the handler runs, it reads three sources in turn: a store a waiting request has lent for the space (a bucket delete lends its own while it drains the space's releases), the chains in its arguments, then the authority the space's latest writes captured. Lends and captures are in memory on one instance; the chains in the arguments are what survive a restart.

### Expiry

`Expires` is the latest time a task may start. A task found after it is recorded as expired without running. A sync task's `Expires` bounds its caller's whole wait, and defaults to the caller's deadline. Ingot uses it for authority: a task carries the proofs of the request that scheduled it, and those must last past its `NotBefore` (a cleanup 24 hours after an upload, the abort of a multipart upload abandoned for a week). See [Trade-offs and future work](#trade-offs-and-future-work) for how long they last today.

### Pending tasks

#### Replacing

A task with a `NotBefore` can be submitted as a replacement: if a replacement with the same subject and action is pending and has not started, the one with the later `NotBefore` is kept and the other is dropped. A task with no `NotBefore` is never a replacement and leaves a pending one alone. A replacement that has run, whether it is running or failed and waiting to retry, can no longer be replaced, so a new one is added beside it: its attempt may have acted, so it keeps its place and finishes what it started. Ingot uses this to extend a cached blob's lifetime: each read or new copy of the blob pushes its pending eviction later.

#### Cancelling

A task can cancel a pending task by queue and ID. The cancel joins the caller's transaction, so it commits or fails with the caller's own writes. Cancelling a task that has already started, finished or been cancelled does nothing, so a handler whose task can be cancelled still checks its own preconditions when it runs. Ingot uses this when a write commits, to cancel the cleanup an upload scheduled in case the write never happened.

#### Running delayed work now

A caller can make a subject's pending tasks of an action due now, whatever their `NotBefore` or retry backoff, and get them back by subject to wait on. A subject ending in `*` stands for every subject that starts with what comes before it, which is how a bucket delete runs its space's pending releases. No task brought forward can be replaced any more, so nothing pushes it back again. The same can be done for given task IDs, so a caller cleaning up after itself brings forward only its own tasks. A sync no-op on a subject runs once the tasks due before it there have run, so a caller can wait behind the work it brought forward.

#### Submitting many

A handler can submit many tasks at once, in one statement for each queue they go to, committed with its own writes. Of two replacements in one submission with the same subject and action, the one with the later `NotBefore` is kept, as it would be had they been submitted one by one. Ingot uses it when one task queues work for many subjects: an upload's accept queuing one per shard, or a commit queuing its releases and registrations.

### Batching

A handler can be registered as batching, with a limit. One execution then takes the oldest due task on a subject plus the due tasks after it with the same action, stopping at the first with a different action so the subject's order holds. A batch commits or fails together. Only async tasks batch; extending it to sync tasks is deferred (see [Trade-offs and future work](#trade-offs-and-future-work)). Ingot uses it to send a space's queued object registrations to the network in one request.

### Retries

An action registers with a retry policy for when it runs async: a backoff (first delay, maximum delay, multiplier) and an optional attempt limit. A sync task is never retried: it fails its caller, and the S3 client retries the request. A handler can mark an error as permanent (an abort refused because the blob was accepted, an expired proof) to stop the retries. A task that fails permanently or runs out of attempts is recorded as failed and logged, and monitoring shows it by queue and action. By default the due tasks behind a task waiting to retry run during its backoff. A policy can hold the subject instead, so the tasks behind wait for the retry and the subject's order survives a failure. That holds up the subject's sync tasks too, so only actions whose order matters more than that use it. A task waiting to retry can still be cancelled, since its failed attempt changed nothing outside Postgres ([rule 8](#handlers-must)).

### Failed tasks

A task that fails for good, with a permanent error or out of attempts, stays in its queue with its last error. It no longer holds up its subject, and it is kept until it is resubmitted or its queue's failed-task retention passes, so the work it stood for can be inspected and replayed. A resubmission is a new pending task on the same subject and action, behind the tasks already queued there, and can take new arguments, such as fresh authority. A task that ran sync is listed but not resubmitted: its caller has had the failure. A bucket delete uses this to resend its space's failed retractions.

### Retention

Finished tasks are deleted once they are older than their queue's retention, 24 hours by default. Failed tasks are kept until they are resubmitted, unless the queue sets a retention for them too.

### Placement

A task that touches one instance's local state is placed there with `Node`, and only that instance runs it. Ingot runs as one instance, so this changes nothing yet; it is there for a future multi-instance design (see [Trade-offs and future work](#trade-offs-and-future-work)).

## Runner rules

### The runner guarantees

1. **One consumer per subject.** At most one task per (queue, subject) runs at a time.
    * A lease row per (queue, subject) in Postgres enforces it, so it holds across processes. The lease records its owner and expiry, and is renewed three times per lease period (30 seconds by default) while the task runs. A lease left unrenewed past its period is taken over by the next consumer to ask, which runs the interrupted task again; the old holder's commit then fails and writes nothing.
    * A worker makes at most a set number of handler runs on one subject (100 by default, a batch counting as one) before it releases the lease, so one busy subject cannot hold a worker for good.
    * A subject's due tasks run oldest first. A task is due when it has no `NotBefore` or its `NotBefore` has passed, so a delayed task never holds up the ones behind it.
    * A sync child runs as soon as it has its subject's lease, without waiting for older due tasks there. Running another tree while its parent holds leases could deadlock. The handlers' own checks keep this safe.
2. **Effects and spawns commit together.** A task's Postgres writes, its result, its completion, the rows of the tasks it spawns and its sync children's writes all commit in one transaction.
    * Handlers add their writes to an ordered list. The transaction opens only after every handler in the tree has run, and runs the list in the order the handlers ran it, so no transaction stays open across a network call.
    * A write that only makes sense if an earlier read still holds (claiming a reference, say) is conditional. One that affects no rows fails the commit, and the whole task runs again from scratch, up to three times before it is left to resume when its lease expires.
    * Every subject in the tree stays leased until the commit, so reads made before it are safe.
    * A task and its sync children succeed or fail together. A task that runs again re-runs its children from scratch.
    * A top-level task's row commits when it is submitted, before it runs, so a crash leaves it to resume once its lease expires. A sync child's row commits with its parent.
    * A failed commit does not mark the task failed. A sync task's caller keeps retrying it until it commits, its handler fails or its `Expires` passes, which defaults to the caller's deadline, so the caller always learns the outcome and an abandoned task never lands later. An async task is left to resume when its lease expires, as after a crash.
    * Only an error a handler returns marks a task failed.
3. **Sync children run inline.**
    * A sync child runs in its parent's goroutine once its subject's lease is taken, without waiting for a worker.
    * A sync task submitted by an S3 handler runs in the handler's goroutine the same way, once it is its subject's oldest due task. It runs no other task: those due ahead of it are left to the workers, so its `Expires` bounds the caller's whole wait.
    * Worker pools run only async tasks, and tasks taken over after a lease expiry.

### Handlers must

4. **Be idempotent.** Effects outside Postgres (file renames, log fsyncs, network invocations) happen before the transaction in rule 2 commits, so a crash re-runs them. Every handler must tolerate running again after an effect already happened.
5. **Sync-spawn only down the queue order, never up or across.** A task may sync-spawn only into a queue below its own in the order below. A task never sync-spawns onto a subject of its own queue. Async spawns may go anywhere.
6. **Not recurse.** No action spawns, directly or through children, a task with its own action. The runner refuses such a spawn within a tree; an async task that later spawns its own action is not caught.
7. **Not read their own tree's writes.** A tree's writes reach Postgres only when it commits ([rule 2](#the-runner-guarantees)), so a handler decides from what it read before the tree wrote anything and from its children's results, and never re-reads a row the tree has written. The runner cannot detect a breach, which shows up as a stale read, so a test helper runs each tree twice, once with writes held to the end and once with them applied straight away, and checks both give the same result.
8. **Fail before acting.** A handler makes its checks, and returns any error it is going to return, before its first effect outside Postgres. After that, it fails only for an error a retry could never get past, such as an expired proof. A task then either fails having changed nothing outside Postgres or ends up committed, since a failed commit is retried ([rule 2](#the-runner-guarantees)). The one exception is a sync task whose commit keeps failing until its deadline: it expires, and any effect it already made outside Postgres stays, so it must be one that is safe to leave behind, such as an upload the provider expires on its own.

Queue order (sync spawns only go down):

1. [`queue-multipart-upload`](#multipart-upload-queue-queue-multipart-upload)
2. [`queue-multipart-upload-shard`](#multipart-upload-shard-queue-queue-multipart-upload-shard)
3. [`queue-bucket`](#bucket-queue-queue-bucket)
4. [`queue-space-upload`](#space-upload-queue-queue-space-upload)
5. [`queue-space-shard`](#space-shard-queue-queue-space-shard)
6. [`queue-shard`](#shard-queue-queue-shard)

Rule 5 is the lock-ordering discipline Ingot already follows (commits take their claim-key locks sorted by digest so two commits sharing digests cannot deadlock). The difference is that it is now stated once for every queue, which matters most where porting today's code would break it: see the release check under [`queue-space-shard.release`](#queue-space-shardrelease).

Rule 5 orders a parent and its children, but not siblings. Two trees whose sync children take the same subjects in different orders, such as two `CopyObject` commits whose `queue-space-shard.add` children share shards, could each wait on a lease the other holds. The runner settles it by the age of each tree's root task: a tree that asks for a lease held by an older tree releases every lease it holds and starts again, keeping its age, so it eventually becomes the oldest and waits instead. A restart re-runs the tree's handlers, which rule 4 makes safe.

---

## How S3 requests use the queues

What happens in the handlers in terms of interactions with the queues. Indented tasks are spawned by the task above them; a delayed async task says when it runs.

* **sync**: the caller waits for the task and gets its result. The task runs in the caller's goroutine once its subject's lease is free ([rule 3](#the-runner-guarantees)), its writes commit with the caller's ([rule 2](#the-runner-guarantees)), and if it fails, the caller fails.
* **async**: the caller records the task and carries on. Its row commits with the caller's writes (or straight away, when the caller is an S3 handler), so it is never lost, and a worker runs it later: as soon as its subject is free, or once its `NotBefore` passes. The caller does not wait for it, and its failure does not fail the caller; the runner retries it under the action's retry policy.

### `CreateBucket`

1. (sync) [`queue-bucket.create`][bucket-create]

### `DeleteBucket`

1. (sync) [`queue-bucket.close`][bucket-close] - refuses a bucket that holds objects, and marks it closing, so no commit or new multipart upload gets in
1. For each of the bucket's multipart uploads, as AWS deletes a bucket with its uploads. The request runs them: the multipart queue is above the bucket queue, so a bucket task cannot
    * an open upload: (sync) [`queue-multipart-upload.abort`][multipart-upload-abort]
        * (async) [`queue-space-shard.release`][space-shard-release] - each part shard, due now
    * a completed one: (sync) [`queue-multipart-upload.reap`][multipart-upload-reap]
1. (sync) [`queue-bucket.delete`][bucket-delete]
    * [`queue-space-shard.release`][space-shard-release] - each of the space's pending releases, run at once, ignoring `NotBefore`
        * (async) [`queue-shard.evict`][shard-evict]
    * `queue-space-upload` - the space's pending registrations and retractions, run at once

### `PutBucketVersioning`

1. (sync) [`queue-bucket.set-versioning`][bucket-set-versioning]

### `PutObjectLockConfiguration`

1. (sync) [`queue-bucket.set-object-lock-config`][bucket-set-object-lock-config]

### `PutBucketTagging`, `DeleteBucketTagging`

1. (sync) [`queue-bucket.set-tagging`][bucket-set-tagging]

### `PutBucketCors`, `DeleteBucketCors`

1. (sync) [`queue-bucket.set-cors`][bucket-set-cors]

### `PutBucketEncryption`, `DeleteBucketEncryption`

1. (sync) [`queue-bucket.set-encryption`][bucket-set-encryption]

### `PutBucketPolicy`, `DeleteBucketPolicy`

1. Sent to Hilt, which stores the policy. No task.

### `PutObject`

1. Stream the body, split into shards, uploading each to the provider as its bytes arrive and, with the cache on, writing it to a temporary file. If a stream fails, abort its allocation, delete its temporary file and fail the request. For each shard, once its digest is known and its body is sent:
    * (sync) [`queue-space-shard.add`][space-shard-add] - concludes the shard, unless the space holds it already
        * (async, at 24 hours) [`queue-space-shard.expire`][space-shard-expire] - cancelled when the reference is claimed
            * (async) [`queue-space-shard.release`][space-shard-release] - due now, when nothing else in the space references the shard
                * (async) [`queue-shard.evict`][shard-evict]
    * (sync) [`queue-shard.cache`][shard-cache] - with the cache on, once the add has run: moves the temporary file into the cache
        * (async, at the TTL) [`queue-shard.evict`][shard-evict]
1. (sync) [`queue-bucket.commit`][bucket-commit] with a `Put` change - claims the shards' references
    * (async, after the reader grace period) [`queue-space-shard.release`][space-shard-release] - each shard of a superseded version (unversioned bucket) that nothing else in the space claims
        * (async) [`queue-shard.evict`][shard-evict]
    * (async) [`queue-space-upload.add`][space-upload-add]
    * (async) [`queue-space-upload.remove`][space-upload-remove] - root of a superseded version

The catalog log queues a [`queue-bucket.ship`][bucket-ship] whenever it seals a segment, by size or by age.

### `GetObject`

1. For each shard, read the cached file if there is one; with less than half its TTL left:
    * (async) [`queue-shard.cache`][shard-cache]
        * (async, at the TTL) [`queue-shard.evict`][shard-evict] - replacing the pending one
1. Otherwise read the shard from the network. A read does not fill the cache.

### `CopyObject`

1. Within the bucket, which is its space: read the source, then
    * (sync) [`queue-space-shard.add`][space-shard-add] - each source shard, taking a reference and sending nothing. The reference holds the shard whatever becomes of the source, so a copy whose source is deleted before its commit still has its shards; a shard the space no longer holds fails the copy
        * (async, at 24 hours) [`queue-space-shard.expire`][space-shard-expire] - cancelled when the commit claims the reference
1. Within the bucket: (sync) [`queue-bucket.commit`][bucket-commit] with a `Put` change that claims those references, spawning as for `PutObject`
1. Across buckets: handle as `PutObject`, with the source object's read stream as the body

### `DeleteObject`

1. (sync) [`queue-bucket.commit`][bucket-commit] with a `Delete` change
    * (async, after the reader grace period) [`queue-space-shard.release`][space-shard-release] - each shard of the removed version that nothing else in the space claims (none for a new delete marker)
        * (async) [`queue-shard.evict`][shard-evict]
    * (async) [`queue-space-upload.remove`][space-upload-remove] - root of the removed version

### `DeleteObjects`

1. (sync) [`queue-bucket.commit`][bucket-commit] with one `Delete` change per key, each answered with its own result, spawning as for `DeleteObject`

### `PutObjectRetention`, `PutObjectLegalHold`, `PutObjectTagging`

1. (sync) [`queue-bucket.commit`][bucket-commit] with a `SetAttributes` change

### `CreateMultipartUpload`

1. (sync) [`queue-multipart-upload.create`][multipart-upload-create]
    * (sync) [`queue-bucket.admit-upload`][bucket-admit-upload] - refuses a closing bucket
    * (async, after the session TTL, 1 week by default) [`queue-multipart-upload.abort`][multipart-upload-abort] - does nothing once the upload is complete or gone
        * (async) [`queue-space-shard.release`][space-shard-release] - each part shard, due now
            * (async) [`queue-shard.evict`][shard-evict]

### `UploadPart`

1. Stream the body, split into shards, uploading each to the provider as its bytes arrive. Nothing is written to disk. If a stream fails, abort its allocation and fail the request. For each shard, once its digest is known and its body is sent:
    * (sync) [`queue-space-shard.add`][space-shard-add] - records the shard as parked, unless the space has it parked or accepted already
        * (async, at 24 hours) [`queue-space-shard.expire`][space-shard-expire] - cancelled when the part row takes the reference over
1. (sync) [`queue-multipart-upload.add-part`][multipart-upload-add-part] - takes the shards' references over
    * (async) [`queue-multipart-upload-shard.accept`][multipart-upload-shard-accept] - submitted as a replacement
        * (async) [`queue-space-shard.accept`][space-shard-accept] - each parked shard, submitted as a replacement
    * (async) [`queue-space-shard.release`][space-shard-release] - each shard of a replaced part this one does not name, due now: a part is never readable

### `UploadPartCopy`

1. As `UploadPart`, with the source object's range, read through the decrypting read path, as the body. The source may be in any bucket of the tenant, and nothing is shared with it.

### `CompleteMultipartUpload`

1. (sync) [`queue-multipart-upload.complete`][multipart-upload-complete]
    * (sync, as a task of its own) [`queue-multipart-upload-shard.accept`][multipart-upload-shard-accept] - an ordinary task, never a replacement, for the listed parts
        * (async) [`queue-space-shard.accept`][space-shard-accept] - each of their shards still parked. Complete brings these forward and waits for them, and fails the request if one cannot be accepted
    * (sync) [`queue-bucket.commit`][bucket-commit] with a `Put` change - the parts' references become the version's claims, spawning as for `PutObject`
    * (async) [`queue-space-shard.release`][space-shard-release] - each shard of recorded parts not listed in the request, due now
        * (async) [`queue-shard.evict`][shard-evict]
    * (async, after the session TTL) [`queue-multipart-upload.reap`][multipart-upload-reap] - deletes the completed upload, kept until then so a repeated Complete gets the same answer

### `AbortMultipartUpload`

1. (sync) [`queue-multipart-upload.abort`][multipart-upload-abort]
    * (async) [`queue-space-shard.release`][space-shard-release] - each part shard, due now, or after the reader grace period for a shard an object also claims
        * (async) [`queue-shard.evict`][shard-evict]

[bucket-admit-upload]: #queue-bucketadmit-upload
[bucket-close]: #queue-bucketclose
[bucket-commit]: #queue-bucketcommit
[bucket-create]: #queue-bucketcreate
[bucket-delete]: #queue-bucketdelete
[bucket-ship]: #queue-bucketship
[bucket-set-versioning]: #queue-bucketset-versioning
[bucket-set-object-lock-config]: #queue-bucketset-object-lock-config
[bucket-set-tagging]: #queue-bucketset-tagging
[bucket-set-cors]: #queue-bucketset-cors
[bucket-set-encryption]: #queue-bucketset-encryption
[multipart-upload-shard-accept]: #queue-multipart-upload-shardaccept
[multipart-upload-abort]: #queue-multipart-uploadabort
[multipart-upload-add-part]: #queue-multipart-uploadadd-part
[multipart-upload-complete]: #queue-multipart-uploadcomplete
[multipart-upload-create]: #queue-multipart-uploadcreate
[multipart-upload-reap]: #queue-multipart-uploadreap
[shard-cache]: #queue-shardcache
[shard-evict]: #queue-shardevict
[space-shard-accept]: #queue-space-shardaccept
[space-shard-add]: #queue-space-shardadd
[space-shard-expire]: #queue-space-shardexpire
[space-shard-release]: #queue-space-shardrelease
[space-upload-add]: #queue-space-uploadadd
[space-upload-remove]: #queue-space-uploadremove

---

## Queues

### Multipart Upload Queue (`queue-multipart-upload`)

Serialized on: `upload ID`

i.e. multiple runners but only one consumer per upload ID.

Every change to an upload runs here, one at a time: its creation, its parts, its Complete and its Abort. A part therefore never lands beside a Complete or an Abort, and a Complete and an Abort never overlap, with no lock or latch on the upload's row. The slow part of an UploadPart, streaming its shards and taking their references, happens in the request beforehand.

Create, add-part and Complete run sync, and take what they write from their request, through its context, rather than from stored arguments: a task its caller abandoned can answer nobody, so it fails without acting. The auto-abort a create schedules, and the reap a Complete schedules, carry their own arguments.

An upload's accept is scheduled on its own queue, `queue-multipart-upload-shard`, and each shard is concluded on `queue-space-shard`, so no network call made to accept a part holds up a `queue-multipart-upload.add-part`.

Actions:

#### `queue-multipart-upload.create`

Records the upload. Its headers, checksum algorithm and object-lock settings come from the request.

```go
type CreateTask struct {
  Task
  Space    did.DID
  Key      string
  UploadID uuid.ID
}

type CreateResult = Unit
```

Spawns:

* (sync) `queue-bucket.admit-upload` - the bucket must not be closing. The create holds the bucket's subject until it commits, so a `queue-bucket.close` either runs after the upload exists, and the bucket's delete aborts it, or refuses this create
* (async) `queue-multipart-upload.abort` - the auto-abort, with a `NotBefore` one session TTL out (1 week by default), carrying the create request's authority to release the parts. It does nothing once the upload is completed or gone.

#### `queue-multipart-upload.add-part`

Adds or replaces a part for an open upload, and takes its shards' pending references over, cancelling their expiries. The part's shards were added to the space (parked, referenced) before this runs, so it only writes the part row, and parts of one upload barely wait for each other. Of two uploads of the same part number, the later replaces the earlier and releases the shards only the earlier named. An upload not open is NoSuchUpload, and the references the request took expire at once. A request that waits more than 30 seconds for the upload's other changes gets SlowDown.

```go
type AddPartTask struct {
  Task
  UploadID   uuid.ID
  PartNumber int
  Part       PartValue // etag, size, checksum, shard digests, span lengths
  Refs       []TaskID  // the pending references to take over
  Release    ReleaseAuthority
}

type AddPartResult struct {
  Released []multihash.Multihash // shards of the replaced part, released
}
```

Spawns:

* (async) `queue-multipart-upload-shard.accept` - with `NotBefore` set to now, submitted as a replacement, so an upload has at most one accept pending
* (async) `queue-space-shard.release` - each shard of a replaced part this one does not name, due now: a part is never readable

#### `queue-multipart-upload.complete`

Assembles the object from the listed parts and puts it. After the response's headers have gone out, so a long accept is covered by the keep-alive, it runs `queue-multipart-upload-shard.accept` for the parts it lists as a task of its own, not a child, then brings forward the `queue-space-shard.accept` of each of their shards still pending and waits for them, within 30 seconds in all. Their writes commit before Complete reads the parts again and before it assembles the object's tree, which reads accepted shards back through their locations ([rule 7](#handlers-must)). A listed part still parked after the wait has a shard whose accept failed, and Complete fails for the client to retry. Parts recorded but not listed in the request are dropped. A completed upload is kept, so a repeated Complete answers with the same version, until its `queue-multipart-upload.reap`.

```go
type CompleteTask struct {
  Task
  UploadID uuid.ID
  Parts    []CompletedPart // part number + etag, from the request
  Release  ReleaseAuthority
}

type CompleteResult struct {
  VersionID string
}
```

Spawns:

* (sync, as a task of its own, committed before Complete reads on) `queue-multipart-upload-shard.accept` - the listed parts, whose `queue-space-shard.accept` tasks Complete then waits for
* (sync) `queue-bucket.commit` - a `Put` change that adds the object to the bucket
* (async) `queue-space-shard.release` - each shard of recorded parts not listed in the request, due now: a part is never readable
* (async) `queue-multipart-upload.reap` - with a `NotBefore` one session TTL out

A Complete that waits more than 10 seconds behind the upload's other changes, another Complete included, answers OperationAborted; one that lost to another replays the winner's result.

#### `queue-multipart-upload.abort`

Deletes the upload and its parts, and releases their shards. An upload already completed or gone is NoSuchUpload for a client's abort, and nothing for the auto-abort.

```go
type AbortTask struct {
  Task
  UploadID uuid.ID
  Release  ReleaseAuthority // the request's, or the create request's for the auto-abort
  Auto     bool             // the auto-abort: leaves an upload completed or gone alone
}

type AbortResult = Unit
```

Spawns:

* (async) `queue-space-shard.release` - each part shard still there to release, due now, since an aborted part was never readable; after the reader grace period for a shard an object also claims, which a reader may hold

Retries: no attempt limit, backing off from 1 second, doubling, up to 5 minutes. Dropping the auto-abort would leave an abandoned upload open, holding its parts' blobs.

#### `queue-multipart-upload.reap`

Deletes a completed upload and its part rows, one session TTL after its Complete, or at once when its bucket is deleted. It releases nothing: the Complete queued the releases of the parts it left out with its own commit, and the object's references hold the rest. A separate action from the abort, which only ever aborts an open upload.

```go
type ReapTask struct {
  Task
  UploadID uuid.ID
}

type ReapResult = Unit
```

Spawns: NONE

### Multipart Upload Shard Queue (`queue-multipart-upload-shard`)

Serialized on: `upload ID`

i.e. one accept per upload at a time, running apart from the upload's own queue.

Schedules the acceptance of a multipart upload's parked shards while the client is still uploading, so CompleteMultipartUpload has only the last few left to accept. The shards are concluded by `queue-space-shard.accept`, under each shard's own subject, so a shard's conclude never runs beside its release or a write's add of it. This queue makes no network calls.

Actions:

#### `queue-multipart-upload-shard.accept`

Queues a `queue-space-shard.accept` for each shard of the upload's parked parts, including parts parked while it runs, in one submission ([Submitting many](#submitting-many)). A part with no shards is marked accepted here, since nothing else will. Does nothing when nothing is parked or the upload is no longer open. Replaces the per-upload acceptor goroutine and the sweeper that nudges it.

`queue-multipart-upload.complete` runs this action for the parts it lists, as a task of its own. Submitted as an ordinary task, never a replacement, it runs after any accept already pending.

```go
type AcceptTask struct {
  Task
  UploadID uuid.ID
  Parts    []int // limits the accept to these parts; empty for every parked part
}

type AcceptResult struct {
  Queued int // shards whose accept this execution queued
}
```

Spawns:

* (async) `queue-space-shard.accept` - each shard of the parked parts, as a replacement, so a shard has at most one accept pending however many parts or uploads name it

Retries: 5 attempts, backing off from 5 seconds, doubling. A later part's accept, or the accept CompleteMultipartUpload runs, queues whatever is left.

### Bucket Queue (`queue-bucket`)

Serialized on: `space`

Serializes every change to the bucket's objects, its settings and its lifecycle. Every object write rewrites the bucket's one MST root, and what a write does depends on bucket state (the version ID and what it supersedes depend on versioning), so all of it is decided here, as today's code decides it under the per-bucket lock. The slow part of a write (streaming the body, uploading its shards, `queue-space-shard.add`) happens in the handler beforehand, so this subject is held for the commit itself. Commits and ships touch the bucket's catalog log on local disk, so every task here is placed on the instance that holds it.

Every action acts only on the bucket that still lives in the task's space. A bucket deleted and recreated under the same name lives in another space, so a task queued for the first is NoSuchBucket rather than a change to the second.

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

Applies a list of key changes in order, and refuses a closing bucket (`queue-bucket.close`): decides each one's effect from the bucket's state, splices them into the MST, appends the new catalog blocks to the bucket's log in one fsynced batch and swaps the root once. Every handler passes one change except `DeleteObjects`, which passes every key of its request, so up to 1,000 deletes cost one fsync and one root swap. Replaces the per-bucket in-process lock and the root CAS. Everything below is decided under this subject, so a concurrent `queue-bucket.set-versioning` lands entirely before or after a commit:

* the new version's sequence number and ID: a minted ID with versioning Enabled, the null version otherwise
* what it supersedes: nothing with versioning Enabled, otherwise the key's null version, whose references are removed and whose upload is retracted
* the write's preconditions (If-Match, If-None-Match), checked against the version it would supersede
* for a delete, whether it adds a delete marker or removes the version, and the refusal of a version under retention or legal hold
* for a put or copy, claiming the new version's pending references and cancelling their `queue-space-shard.expire` tasks
* for the versions it supersedes or removes, dropping their claims, and queuing a release of each shard nothing else in the space claims

Each change gets its own result. A change that fails on its own terms (a failed precondition, a version under retention or legal hold) fails only that change, and the rest still apply. A failure of the commit itself, such as the fsync or the Postgres transaction, fails every change. A within-space copy is a `Put`: it reads its source and takes references to its shards before the commit, as an upload does, and the references keep the shards whatever becomes of the source.

```go
type CommitTask struct {
  Task
  Space   did.DID
  Changes []KeyChange       // applied in order; one per key
  Release ReleaseAuthority  // passed to the releases it queues
}

// KeyChange holds a key and exactly one change to it.
type KeyChange struct {
  Key           string
  Put           *PutChange
  Delete        *DeleteChange
  SetAttributes *SetAttributesChange
}

// PutChange adds a version from an upload, a copy or a completed multipart
// upload.
type PutChange struct {
  Value         ObjectValue
  Refs          []TaskID // pending references to claim; empty for shards a multipart part already holds
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

* (async) `queue-space-shard.release` - each shard of a superseded or removed version that nothing else in the space claims, with `NotBefore` set to the reader grace period (60 seconds by default), submitted as a replacement, so a reader of the newer version gets its full grace
* (async) `queue-space-upload.add` - a new version's root, for a put or copy
* (async) `queue-space-upload.remove` - root of a superseded or removed version
* (async) `queue-bucket.ship` - when the bucket's last ship found no authority, which this commit's request supplies

#### `queue-bucket.ship`

Ships sealed catalog segments to the network.

It ships every sealed segment still unshipped, oldest first, and stops at the first that fails. The log seals on its own trigger, by size or by age, and queues a ship when it does, as a replacement due at once, so a bucket has at most one pending; a seal found unshipped at startup queues one too. A seal by age has no request behind it, so the ship signs with the authority the bucket's latest write captured, as today's ship does, and carries no proofs of its own. A ship that finds none fails for good, and the bucket's next commit queues it again.

```go
type ShipTask struct {
  Task
  Space did.DID
}

type ShipResult struct {
  Segments []cid.Cid
}
```

Spawns: NONE

Retries: no attempt limit, backing off from 1 second, doubling, up to 5 minutes. Missing authority is permanent, as above. Dropping a ship any other way would leave an idle bucket's catalog only on local disk.

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

#### `queue-bucket.set-tagging`

Stores the bucket's tag set; an empty one clears it. Like every action here, it acts only on the bucket that still lives in this space, so a request meant for a bucket deleted and recreated under the same name is NoSuchBucket rather than a change to the new bucket.

```go
type SetTaggingTask struct {
  Task
  Space did.DID
  Tags  []byte // validated by the S3 layer
}

type SetTaggingResult = Unit
```

Spawns: NONE

#### `queue-bucket.set-cors`

Stores the bucket's own CORS configuration; an empty one clears it, and the gateway's applies.

```go
type SetCORSTask struct {
  Task
  Space  did.DID
  Config []byte // validated by the S3 layer
}

type SetCORSResult = Unit
```

Spawns: NONE

#### `queue-bucket.set-encryption`

Stores the bucket's default encryption; an empty one clears it.

```go
type SetEncryptionTask struct {
  Task
  Space     did.DID
  Algorithm string // AES256, or none where the deployment allows it
}

type SetEncryptionResult = Unit
```

Spawns: NONE

#### `queue-bucket.close`

The first step of DeleteBucket. Refuses a bucket that holds objects (BucketNotEmpty), and otherwise marks it closing: from then on `queue-bucket.commit` and `queue-bucket.admit-upload` refuse it, so it stays empty and gains no multipart upload while the request aborts the ones in flight. A delete that stops after this leaves the bucket closed, and the client's retry carries on from there.

```go
type CloseTask struct {
  Task
  Space did.DID
}

type CloseResult = Unit
```

Spawns: NONE

#### `queue-bucket.admit-upload`

Run by `queue-multipart-upload.create` as a child: refuses a closing bucket. The create holds this subject until it commits, so its upload either exists before a `queue-bucket.close`, and the delete aborts it, or is refused. No lock is needed.

```go
type AdmitUploadTask struct {
  Task
  Space did.DID
}

type AdmitUploadResult = Unit
```

Spawns: NONE

#### `queue-bucket.delete`

The last step of DeleteBucket, after `queue-bucket.close` and the aborts and reaps of the bucket's multipart uploads. It fails if the bucket holds objects, which the close makes impossible. In order, it:

1. commits the bucket's root as the empty tree, ahead of its own commit, so nothing the bucket reads is released below;
2. quiesces the catalog log, and removes the blobs of its retired and shipped segments from the space;
3. lends the request's authority for the space and runs the space's pending releases at once, since an explicit delete owes readers no grace (a release still waiting fails the delete, for the client to retry);
4. sends the space's pending registrations and retractions, and resends failed retractions with the request's authority, so the objects the emptying removed are not left counted;
5. asks Hilt to delete the space. Not found means an earlier delete removed it, so this one finishes. An explicit refusal clears the closing mark, and the bucket works again. A call that failed on its way leaves the bucket closing until a retry settles it;
6. removes the bucket's local catalog log, then deletes the bucket's row. The log is keyed by the bucket's name, and a bucket created under the name once the row goes runs under its own space, apart from this task, so the log goes while the name is still held.

Hilt authorizes the delete with the signed S3 request itself. A signed request can be replayed for that one operation while it is inside Hilt's 15-minute clock-skew window, so it is never stored: it reaches the task through its caller's context. A delete resumed without its caller fails without acting; the client, whose request failed with it, retries with a freshly signed request, and every step before the Hilt call is safe to repeat.

```go
type DeleteTask struct {
  Task
  Space did.DID
  Name  string
}

type DeleteResult = Unit
```

Runs, as tasks of their own rather than children, since it waits on them only to drain the space:

* `queue-space-shard.release` - each of the space's pending releases, ignoring `NotBefore`
* `queue-space-upload` - the space's pending registrations and retractions

### Space Upload Queue (`queue-space-upload`)

Serialized on: `space`

Registers and retracts object roots with the network (`/upload/add`, `/upload/remove`), which is where the network's object count comes from. An add and a remove of one root must not reorder. Serializing on the space keeps them in order and stops one busy space starving another. Both actions batch, up to 256 changes of one kind in one request, as the registration sweep does today. A batch takes only consecutive changes of one kind, so adds and removes stay in sequence.

A change whose authority has been spent first tries to renew it from what the space captured since, and keeps the renewal in its arguments, so a restart does not leave it with the spent chain. One that still cannot be sent is retried 10 times and then left as a failed task, so it no longer holds back the changes behind it. A bucket delete resends its space's failed retractions with its own request's authority.

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

Retries: no attempt limit while the change has usable authority, backing off from 1 second, doubling, up to 5 minutes, holding the space so an add and a remove of one root keep their order. Dropping it would leave the space's object count wrong.

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

Retries: no attempt limit while the change has usable authority, backing off from 1 second, doubling, up to 5 minutes, holding the space so an add and a remove of one root keep their order. Dropping it would leave the space's object count wrong.

### Space Shard Queue (`queue-space-shard`)

Serialized on: `space + digest`

i.e. multiple runners but only one consumer per space + digest.

Serializes everything done to a shard in a space (bucket): a write's add, its accept, its references' expiry and its release, so no two of them change the shard's network state at once.

Nothing stores a count of a shard's references. A reference is any of:

* a **claim**: a row per object version that holds the shard, written and dropped by `queue-bucket.commit`;
* a **pending reference**: a row per `queue-space-shard.add`, identified by the add's task ID, which lasts until the write takes it over or it expires;
* a **part** of an open multipart upload that names the shard.

A release counts all three in one statement, so a row that takes another's place in one transaction (a claim or a part row taking a pending reference over) is counted before or after, never neither. Whatever drops the last reference it knows of queues a release, and the release decides when it runs whether the shard is in fact free: a release that finds a reference does nothing.

A write takes its pending reference over in the same transaction that records it: a `queue-bucket.commit` claims it for the new version, and a `queue-multipart-upload.add-part` replaces it with the part row. Either cancels the reference's `queue-space-shard.expire`. The takeover is conditional on the pending reference still being there, so a takeover and an expiry that run together settle without a lock: a write that finds its reference expired fails, and the client retries.

The deadline is 24 hours after the add. Neither S3 nor Ingot limits how long a request may take, so the deadline only has to be long enough that a real upload never reaches it. 24 hours matches how long the provider keeps an unconcluded allocation, so an abandoned upload's pending references and its allocations are both gone within about a day.

Actions:

#### `queue-space-shard.add`

Makes a written shard durable for the space and takes the write's pending reference to it. It runs in one of three modes:

* **streamed** (PutObject): concludes the shard's allocation, sending `/ucan/conclude` with the `/http/put` receipt, which triggers `/blob/accept`, and records the location;
* **part** (UploadPart): records the shard as parked, with the provider but not yet accepted, for `queue-space-shard.accept` to conclude later;
* **copy** (a copy within the bucket): sends nothing, and fails if the space no longer holds the shard.

A space that holds the shard already, parked or accepted, keeps it, and a streamed write's own allocation is given up. The allocation's `/http/put` invocation carries derived signer keys, so it reaches the add through its caller's context and is never stored.

A `queue-space-shard.release` of the same space and digest cannot leave the upload without its blob. The body streams into an allocation of its own, which the release does not touch, and the add runs under the same lease as the release. So the add either finds the blob still there and takes a reference that keeps it, or finds it removed and concludes its own allocation in its place. That relies on the protocol's guarantee that a space can take a digest again once a `/blob/remove` of it has a receipt. An add that waits more than 30 seconds behind a release, running or waiting to retry, is refused with SlowDown, which the client retries.

`queue-space-shard.add` is the durable step of an upload. Before it, nothing records the allocation: a stream that fails while Ingot runs aborts its allocation inline, and one abandoned by a crash is never concluded, so the provider expires it.

```go
type AddTask struct {
  Task
  Space   did.DID
  Digest  multihash.Multihash
  Mode    string // streamed, part or copy
  Bucket  string // recorded with the shard's upload intent
  Size    int64
  Release ReleaseAuthority // passed to the release an expiry queues
}

type AddResult struct {
  Ref    TaskID // the pending reference: this task's ID
  Reused bool   // the space held the shard already, so the write's allocation is surplus
}
```

Spawns:

* (async) `queue-space-shard.expire` - with `NotBefore` 24 hours out

#### `queue-space-shard.accept`

Concludes a parked shard: sends `/ucan/conclude` with the shard's `/http/put` receipt, which triggers `/blob/accept`, records the location, and marks accepted every parked part of the space's open uploads that names the shard and now has all its shards located. A part's shards are accepted under different subjects, so the part rows are locked first, in a fixed order: whichever accept commits second sees the first's location and marks the part. A shard already located needs nothing concluded. A shard nothing in the space references any more, as a dropped part's is, is left for its release, which aborts the allocation.

Because it runs under the shard's subject, a `queue-space-shard.release` of the same shard runs before it, and the accept then finds no park and does nothing, or after it, and removes the accepted blob. The two never overlap.

```go
type AcceptTask struct {
  Task
  Space  did.DID
  Digest multihash.Multihash
}

type AcceptResult = Unit
```

Spawns: NONE

Retries: 5 attempts, backing off from 5 seconds, doubling. A failed conclude records nothing ([rule 2](#the-runner-guarantees)); concluding a shard again counts as a repeat.

#### `queue-space-shard.expire`

Drops the pending reference a `queue-space-shard.add` took, if nothing took it over. Its takeover normally cancels it; if the takeover commits while it is already running, it finds the reference gone and does nothing. When nothing else in the space references the shard, it queues the shard's release due now: a shard nothing claimed was never readable, so no reader grace is owed.

```go
type ExpireTask struct {
  Task
  Space   did.DID
  Digest  multihash.Multihash
  Release ReleaseAuthority // passed to the release it queues
}

type ExpireResult = Unit
```

Spawns:

* (async) `queue-space-shard.release` - due now, when the space's references reach zero

Retries: no attempt limit, backing off from 1 second, doubling, up to 5 minutes. Dropping it would leave the shard referenced for good.

#### `queue-space-shard.release`

Removes the shard from the space on the network: `/blob/remove` if it was accepted, `/blob/abort` if it was parked and never accepted, and `/blob/remove` anyway if the provider answers that the space accepted it. A shard with neither a location nor a park may still be accepted, if a conclude was accepted and its location never recorded, so the release sends `/blob/remove` for it too, which costs one round trip and closes that leak. It deletes the shard's encryption params first (crypto-shred), so a shard nothing references becomes unreadable at once, and then its rows, with its commit. It evicts the cached copy unless another space's upload intent names the digest.

Releases are queued by whatever drops a reference: a commit that drops a version's last claim, after the reader grace period and as a replacement, so a reader of the newer version gets its full grace; and a teardown, a replaced part or an expiry, due now. It does nothing when the space still references the shard (a write reused it, so no cancellation is needed) or when the shard's upload intent is gone, which only a release's commit removes (an earlier release has run).

It shares its subject with the shard's `queue-space-shard.accept`, so it never runs while the shard is being concluded. A conclude Ingot gave up on (a timeout, or a crash that lets its lease lapse) may still be running at the upload service; ordering a remove against that is the upload service's to get right, and Ingot treats a `/blob/remove` receipt as final.

Today's release also checks whether an in-flight multipart part names the blob. Here a part is one of the references the release counts. Porting the check as a sync query into `queue-multipart-upload` would be a sync spawn up the queue order ([rule 5](#handlers-must)) and could deadlock against a `queue-multipart-upload.add-part` waiting on this subject.

```go
type ReleaseTask struct {
  Task
  Space  did.DID
  Digest multihash.Multihash
  ReleaseAuthority // the /blob/remove and /blob/abort chains
}

type ReleaseResult struct {
  Released bool
}
```

Spawns:

* (async) `queue-shard.evict` - with no `NotBefore`

Retries: no attempt limit, backing off from 1 second, doubling, up to 5 minutes, holding the subject: a failed attempt may already have removed the blob, so an add of the same shard must wait for the retry and then find it gone rather than reuse it. Dropping it would leave a blob nothing references on the network.

### Shard Queue (`queue-shard`)

Serialized on: `digest`

i.e. multiple runners but only one consumer per unique digest.

Serializes the cache's copy of a digest. Every task on this queue is placed on the instance whose disk holds the file. Ingot keeps a blob on disk only to cache it: an upload streams each shard to the provider as its bytes arrive (the trailer hash design) and never reads it back from disk. That makes streaming required:

* Sprue must support adding a blob by digest code. There is no store-and-forward path.
* A streamed PUT that fails fails the request, and the client retries it. There is no spool copy to upload again.

With the cache off, Ingot writes no blob to disk. With it on, `PutObject` writes each shard to a temporary file in the cache directory as it streams, and once the shard's `queue-space-shard.add` has run, the request runs a sync `queue-shard.cache` that moves the file into the cache and schedules its eviction in the same task, so an eviction of an earlier copy cannot run between the two. A read from the network does not fill the cache. Multipart parts are not cached, as today.

A cached file has a TTL, 1 hour by default, held as a pending `queue-shard.evict`. Caching a blob again (another upload of the same bytes, a cache hit) extends it. A removed shard's `queue-space-shard.release` evicts the file straight away, unless another space's upload intent names the digest. A copy both spaces' releases leave at once is reclaimed by its TTL.

Actions:

#### `queue-shard.cache`

Run sync by a write, it first moves the write's temporary file into the cache under its digest; the file reaches it through the caller's context. Then, if the cache holds the blob, it dates the file now, which orders the budget pass, and schedules its eviction. Run async for a cache hit, it only does the second part.

```go
type CacheTask struct {
  Task
  TTL time.Duration // how long the file stays cached from now
}

type CacheResult = Unit
```

Spawns:

* (async) `queue-shard.evict` - with `NotBefore` set to now + `TTL`, submitted as a replacement, so a later eviction replaces a pending earlier one

A cache hit extends the TTL only when less than half of it remains, tracked in memory as the read retention is today, so most reads write no task.

Retries: 14 attempts, backing off from 1 second, doubling, up to 5 minutes, about half an hour in all, which rides out a passing fault such as a database failover. A failed cache costs only a later cache miss.

#### `queue-shard.evict`

Deletes the cached file. No-op if there is none. A reader that already opened the file keeps reading it.

```go
type EvictTask struct {
  Task
  Budget   bool // run by the disk budget pass, which counts its own removals
  Released bool // queued by a release: the blob is gone from its space
}

type EvictResult struct {
  Freed int64 // bytes removed
}
```

Spawns: NONE

Retries: as `queue-shard.cache`. One that fails for longer, such as a disk that refuses the removal, is left a failed task to inspect and resubmit; until then the copy stays, unless the budget pass removes it or a read schedules a new eviction.

---

## Operations

### Sweepers

| Today | With the runner |
|---|---|
| Release sweeper | `queue-space-shard.release` with `NotBefore` |
| Stream sweeper (aborts a stream whose request died) | Not needed. A failed stream aborts its allocation inline; one abandoned by a crash is never concluded and the provider expires it within 24 hours. An unaccepted blob counts toward no space usage and is never served, so nothing needs to abort it sooner. References an abandoned request took are removed by `queue-space-shard.expire` |
| Multipart acceptor, and the accept sweeper that nudges it | `queue-multipart-upload-shard.accept`, which queues a `queue-space-shard.accept` per shard |
| Session expiry | `queue-multipart-upload.abort` with `NotBefore` |
| Reaping completed sessions | `queue-multipart-upload.reap` with `NotBefore` |
| Upload registration sweep | `queue-space-upload` |
| Cache residency and read retention windows | The TTL on each cached file: a pending `queue-shard.evict`, extended by `queue-shard.cache` |
| Disk budget eviction | Stays a sweeper: a TTL does not bound the disk, and a burst of writes can fill it before anything expires. Over budget, it runs a sync `queue-shard.evict` for each of the oldest cached files first, so each removal takes its turn with the digest's other tasks |
| Orphan file scan | Stays a sweeper. Unfinished `.tmp-*` files belong to no subject |

The runner itself polls for due tasks (`NotBefore` passed, subject free) and for tasks whose lease expired. That poller is generic and is the only one besides the two above.

### Monitoring

Pool and queue sizes are deployment settings, and the right values depend on the workload. The runner reports enough to tune them, through OpenTelemetry, labelled by queue and action, so the library stays agnostic to what the actions do:

* queue depth: tasks ready to run, tasks waiting on `NotBefore` or on a retry that holds their subject, and tasks failed for good
* age of the oldest ready task
* time a sync task or child waits for its subject's lease
* task duration and outcome (succeeded, failed, retried, expired), sync children included
* pool utilization: busy workers, and the most each pool runs
* lease takeovers, i.e. tasks taken over from a consumer that stopped renewing

The queue gauges are read from Postgres each time they are collected, and every registered action reports, at zero when it has no tasks.

---

## Trade-offs and future work

1. **Group commit (deferred).** Each object write holds `queue-bucket` for its own commit, so a bucket commits one write at a time, the same ceiling as today's per-bucket lock. Batching commits (one splice, one fsync, one root swap for many writes) is possible later by extending batching to top-level sync tasks, which `queue-bucket.commit` already is for every write except Complete's. A commit already carries a list of changes with a result for each, so a batch would apply every task's changes and hand each task its own results; its limit should count changes, not tasks. It adds real complexity to the runner, so revisit only if a bucket hits the ceiling.
2. **Multiple instances.** Leases make subject ownership work across processes, but two things stay on one instance's disk. The cache is the smaller one: another instance only misses it, and its tasks are placed on the instance holding the file. The bucket's catalog log is the blocker. Until a segment ships, the catalog blocks a bucket's root depends on exist only on the disk of the instance that committed them, so no other instance can commit to the bucket or read its newest objects, and losing that disk loses those blocks. Several instances need bucket ownership (a lease per bucket, with requests routed to its owner), catalog blocks made durable beyond one disk before a commit is acknowledged, and a way to wake a sync caller waiting on a task that runs in another process. Placement is in the runner already, so commits and ships can follow the bucket's owner.
3. **Write volume.** A `PutObject` of N shards writes about 4N + 2 task rows before it returns with the cache on (2N + 2 with it off): per shard a `queue-space-shard.add`, `queue-space-shard.expire`, `queue-shard.cache` and `queue-shard.evict`, plus a `queue-bucket.commit` and a `queue-space-upload.add`. An `UploadPart` writes about 3N + 2: per shard an add, an expire and a `queue-space-shard.accept`, plus the add-part and the upload's accept. Superseding a version or sealing a log segment adds a few more. Every task is durable on purpose (see the intro). Letting a request's inline sync tasks, other than `queue-space-shard.add`, skip their rows would roughly halve this, at the cost of a per-action durability decision that fails silently when wrong. Revisit only if row volume shows up in profiling.
4. **Batched concludes (deferred).** Each shard is concluded on its own, so a shard's accept is ordered with its releases by the shard's subject. The upload service accepts up to 1,000 shards in one `/ucan/conclude`, and today's acceptor sends them that way. Shards are accepted as they arrive, so a Complete waits only for the last few, and one round trip per shard has not shown up as a cost. Batching them again would need the runner to lease many subjects in one execution. Revisit if conclude round trips show up in profiling.
5. **Authority for delayed cleanup.** A cleanup task invokes the network with the authority of the request that queued it, captured into its arguments. Hilt delegates the cleanup commands (`/blob/remove`, `/blob/abort`, `/upload/remove`) from the tenant ([fil-forge/hilt#125](https://github.com/fil-forge/hilt/pull/125)), lasting until 8 days after the next UTC midnight, so they cover a `queue-space-shard.expire` 24 hours after an upload and the auto-abort a week after its upload was created, and they survive the access key's expiry, deletion or rotation. A store a waiting request lent for the space comes before the captured chains, and the space's latest captured authority stands in for a command the task holds no chain for. A task whose authority has lapsed fails permanently and is kept as a failed task, to be resubmitted with fresh authority, as a bucket delete does for its space's retractions.
6. **Filling the cache on reads (unbuilt).** The cache holds only what writes put there, so a blob read from the network is read from the network again next time. A read of a whole blob could fill it: write the blob to a temporary file as it streams, verify its digest, then run `queue-shard.cache` with the file, as a write does. Whether to is undecided, not ruled out. It would trade disk and write load on every miss against network reads of blobs read more than once, and the budget pass would have to keep up with reads as well as writes.

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
| In-process ownership of each upload's acceptor goroutine | `s3frontend/acceptor.go` | `queue-multipart-upload-shard.accept`, and `queue-space-shard.accept` per shard |
| In-process mutex per (space, digest), so two streams of the same bytes in one space never both record a location | `s3frontend` (`Backend.blobKeys`, taken in `concludeStreamed`) | `queue-space-shard` |
| In-memory spool holds per (space, digest), which stop a file a write is still uploading from being moved or removed | `blockstore/spool.go` (`Spool.holds`) | Nothing: an upload reads no local file, so there is no spool (see `queue-shard`) |
| Lease on `blob_streams` rows, renewed by the live request | `s3frontend/stream.go` | Nothing: an unconcluded allocation expires on the provider, and `queue-space-shard.add` is the upload's first durable step |

The migration lock in `migrations/migrations.go` stays: it serializes schema migrations at startup, not data.
