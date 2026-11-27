# RFC: Hilt's tenant changes on the task runner

Status: Experimental

## Authors

- [Alan Shaw](https://github.com/alanshaw)

## Motivation

A tenant that is disabled or write-locked in Hilt can keep using S3 for up to a day. Hilt's status change flips a column and revokes nothing, so a key that Ingot has cached keeps working until its cached authority expires at the next UTC midnight. A write-locked tenant can also still upload, because nothing narrows what its keys may do. The trial egress limit and the trial storage limit both depend on these two statuses.

[hilt#137](https://github.com/fil-forge/hilt/pull/137) fixes this by revoking and reissuing every grant a tenant's keys hold when its status changes. To do that safely it adds a tenant-wide lock that key creation, policy writes and authorization all have to take. Its author marked it as blocked: the whole change runs as one all-or-nothing step under an 8-second deadline, and a tenant with enough keys can exceed it and keep its old status.

That deadline is not particular to #137. Every operation in Hilt that changes who may do what holds row locks while it calls the vault and the revocation service, and each of them runs under the same 8 seconds. They lock each other's rows in a fixed order to avoid deadlock, and report a concurrent change to the caller when a wait runs out. Each new feature in this area adds another lock to that order.

Ingot had the same problem and solved it with a task runner ([RFC](./2026-10-ingot-task-runner.md)): one task at a time per subject, no transaction open across a network call, and handlers that are retried until they succeed. This RFC proposes that Hilt runs its authority-changing operations the same way, serialized on the tenant.

## Goals

- A tenant's status change takes effect at Ingot within seconds while the revocation service is up, for a tenant with any number of keys.
- Write-locking stops uploads and bucket creation and leaves reads, listings and deletes working.
- Two changes to one tenant's authority never run at the same time, without either taking a lock the other must know about.
- A change that fails part way resumes and finishes without the caller retrying.
- No request is authorized from a state a change has already replaced, or while that change is waiting or running.
- A revocation service outage blocks no tenant. During one, a key Ingot has already cached keeps its old access until the revocation is published or the cached copy expires at the next UTC midnight, which is the delay every status change has today.

## Design

### Which operations become tasks

The operations that change what a tenant's keys may do run as tasks on one queue, with the tenant's DID as the subject:

| Operation | Today |
|---|---|
| Bucket policy write and removal | Locks the policy row, then the affected principals' rows, then each affected key's grants, and publishes revocations inside those locks. A wildcard statement reaches every key the tenant has. |
| Principal removal | Three steps, ordered to avoid a deadlock with a policy write, with a re-check for a policy that names the principal again. |
| Access key creation and deletion | Share locks on the principal and its policies, and a rollback run from eight failure points. |
| Tenant status change | Flips a column. |
| Bucket deletion | Holds the bucket's policy lock while it revokes the bucket's grants. |
| Tenant deletion | A cascade through the DID directory, the revocation service, the vault and six stores, which the caller repeats after a failure. |

The REST and RPC handlers keep their request and response shapes. Each submits its task and waits for the result, so a caller sees the same success or failure it does today. A lock timeout no longer reaches the caller as a concurrent change: a second change to the same tenant waits its turn.

Tenant deletion runs to completion while the caller waits, as it does today, and answers `204` once the tenant is gone. If the caller gives up first, the task carries on and finishes. A repeated request then finds nothing left to delete and answers `204`, so fil-one's retry of a failed deletion keeps working and is needed less.

Everything else stays as it is: authorization, bucket listing and info, metrics, the usage report, bucket creation, tenant provisioning and the provider admin commands. They are reads, single idempotent writes, or flows whose signing key exists only in memory and cannot be resumed.

### A change, then its revocations

A change to a tenant's authority is two kinds of task.

**The change** is short and touches only Hilt's database and the vault. In one commit it:

1. writes the new state: the status, the policy document, the removed key;
2. stores the grants the new state adds, issued from the keys' permissions and the policies;
3. takes the grants the new state removes out of the delegation store and records them as revocations owed.

From that commit Hilt answers every authorization from the new state and holds no grant that is due to be revoked. The caller waits for this task and gets its result. If it fails, nothing was committed and the caller gets the error.

**The revocations** follow as tasks the change spawns, on a second queue, each publishing one batch of the revocations owed and deleting what it published. They retry with a capped backoff for as long as it takes. A revocation carries no nonce, so publishing one twice is recorded once. The batches are what remove the 8-second limit: a tenant with many keys takes longer and still finishes.

Until a key's revocations are published, a copy of its old authority that Ingot has cached keeps working, until that copy expires at the next UTC midnight. That is how every status change behaves today. With the revocation service up, it lasts seconds.

Tenant deletion is the exception to the split. The revocation service verifies a revocation against the tenant's DID, so deletion publishes its revocations inside its own task, before it deactivates the DID, as it does now.

### Authorization waits for a pending change

Authorization is not a task. It answers most S3 requests that reach Hilt, and running it on the tenant's subject would put a database write on every one and make a tenant's requests run one at a time.

It checks instead whether the tenant has a change waiting or running:

1. One indexed read asks for the tenant's unfinished tasks on the change queue.
2. With none, authorization answers at once. This is the usual case.
3. With one, it waits for that task to finish, up to 10 seconds, and then answers. Past that it fails the request as temporarily unavailable.

The deadline and the failure are the ones authorization has today, when it waits 10 seconds on a row lock behind a policy write. Ingot already turns that failure into an S3 `503 ServiceUnavailable`, which the AWS SDKs retry with backoff, so neither Ingot nor S3 clients see anything new.

A task's row is committed when it is submitted. So from the moment Hilt accepts a status change, new authorizations for that tenant wait for it and are answered from the state it leaves. This replaces the four share locks authorization takes today, and the tenant-wide lock #137 adds.

Authorization does not wait for revocations. Waiting would add nothing: Hilt already answers from the new state, and a key Ingot has cached does not ask Hilt. A revocation service outage therefore slows revocations down and fails no request.

Authorization reads the key, the principal, the policy and the grants in separate queries. It reads them in one database snapshot so that a change committing between two of them cannot produce an answer that mixes old and new.

A reply already on its way to Ingot when a change commits can outlive the change. [Replies that outlive a change](#replies-that-outlive-a-change) describes when that matters and the follow-up that closes it.

### Disabling and write-locking

**Disabled.** The change sets the status and records every grant the tenant's keys hold as a revocation owed. Nothing is issued. Returning to active sets the status and issues fresh grants for every key from its stored permissions and policies.

**Write-locked.** fil-one's [Service Orchestrator integration guide](https://github.com/fil-one/fil-one/blob/main/docs/service-orchestrator-integration/README.md#tenant-state-enforcement) defines the state: uploads and bucket creation are blocked, and reads, listings and deletes succeed. The change sets the status, records every grant as a revocation owed, and issues narrower ones that carry reads, lists and deletes. Authorization also has to refuse writes itself for requests that reach it, which needs each S3 operation classed as a read, a cleanup or a write. hilt#137 has that classification and [ingot#411](https://github.com/fil-forge/ingot/pull/411) has the matching change for Ingot, which must enforce the reduced set of S3 actions because deletes and uploads share some of the same Forge commands. Both carry over.

Key creation and policy writes read the tenant's status when they issue grants, and they run on the same subject as the status change. A key created for a write-locked tenant therefore gets the narrow grants without a lock to arrange it.

### Replies that outlive a change

An authorization reply built just before a change commits can reach Ingot after it, still saying what the key could do before. Ingot remembers every revoked delegation and does not cache a reply that names one, but a reply names only the key's own short-lived delegations, which Hilt creates for each reply and which are never revoked. The grants behind them reach Ingot separately, from `/s3/bucket/info`, and by then Hilt returns the new ones. So nothing in the reply or its chain is revoked, and Ingot can keep the out-of-date permission list until its cache expires at the next UTC midnight.

For a disabled tenant this does no harm: its keys hold no grants, so Ingot has no chain to serve a request from. For a write-locked tenant it does. A key that may delete holds every Forge command an upload needs, so Ingot decides from the cached permission list, and an out-of-date one lets uploads continue. It needs a reply built within moments of the write-lock, for a key that may delete.

This is a known gap in the first implementation. The follow-up that closes it:

- **Authorization replies carry the grants they rest on,** the bucket's grant to the tenant and the tenant's grants to the key, as delegation blocks.
- **Hilt reads those grants before the tenant's status and the key's permissions.** Every change to what a key may do on a bucket replaces that key's grants over the bucket in the same commit, so a reply that was out of date when it left Hilt carries grants that are being revoked. Ingot's existing check then refuses to cache it, or drops it when the revocation arrives. No snapshot read is needed.
- **`/s3/bucket/info` is removed.** Completing a chain is its only use, and the reply then carries the whole chain. Authorization is already Ingot's slow path, taken only when its cache cannot answer, so the extra blocks are sent only then, and a key's first request for a bucket needs one round trip where it needs two today.

### The runner

The task runner moves out of Ingot's repository into a module both services import. Hilt adds two queues and their tables in a migration, one for changes and one for revocations, and starts workers with the service.

The runner gains one call. It can list the unfinished tasks of one action on a subject, and authorization needs every unfinished task on a subject of the change queue, whatever its action. The new call returns those, running or waiting, oldest first.

A task's arguments and result are stored in Postgres. Access key creation returns a secret once, and the secret stays out of both: the handler generates the key and writes it to the vault before it submits the task, and passes only the key's DID.

The runner defers a handler's writes to the task's commit, so a handler cannot read a row it has just written. The flows that write a row and read it back are restructured to carry the value forward.

### Hilt's in-memory backend is removed

Every store in Hilt has an in-memory twin, kept in step with the Postgres one by a shared test suite. The runner needs Postgres. The in-memory stores and the storage type that selects them are removed, and the service tests run against Postgres in a container, as the store tests already do.

## Operations

Revocations that are not getting published leave cached access in place longer, and nothing else shows it. Hilt exports the age of the oldest revocation owed, with an alert when it passes a few minutes.

A revocation that can never be signed, because the tenant's key is gone from the vault, is not retried. Its task fails for good and is listed with the queue's failed tasks, and the cached access it would have removed ends at the next UTC midnight.

## Alternatives considered

### The tenant fence of hilt#137

Keep the locks and add one more: a tenant row lock that status changes take exclusively and everything else shares. It is implemented and tested. It keeps the 8-second all-or-nothing step, which its author judges unsafe for large tenants, and it adds to the lock order every later change has to follow.

### A single link per tenant in every chain

Put one delegation between the tenant and each key's grants, so that a status change revokes or replaces that one link whatever the number of keys. Key grants are issued directly by the tenant today, so this adds a hop to every chain and changes what Ingot validates. It would shrink the work of a status change and does nothing for policy writes or principal removal. It could follow this RFC.

### Disable only, with a retry loop

Flip the status, call the revocation routine tenant deletion already has, and retry from a periodic pass. It is the smallest change that makes the egress limit effective, and it needs no coordination between writers, because nothing issued for a disabled tenant can be used. It leaves write-locking unfixed and the existing locks in place.

## Evaluation criteria

- A trial tenant disabled by fil-one's usage job is refused a GetObject on a key Ingot had cached, within a minute.
- A write-locked tenant is refused a PutObject and allowed a GetObject and a DeleteObject, on a cached key and on a new one.
- A status change for a tenant with 2,000 keys completes.
- Two policy writes and a key creation issued at once for one tenant all succeed, with none reporting a concurrent change.
- Hilt killed part way through publishing a status change's revocations finishes them after restart, with no call repeated by fil-one.
- With the revocation service down, a status change succeeds, requests that reach Hilt are answered from the new status, no tenant sees a `503`, and the revocations are published once the service returns. A key Ingot had cached keeps working until then.

## Order of work

1. Extract the task runner into a shared module, with Ingot importing it, and add the calls that list a subject's unfinished tasks and wait for several tasks.
2. Remove Hilt's in-memory stores and move the service tests to Postgres.
3. Add the two queues and the table of revocations owed to Hilt, and the pending check to authorization.
4. Move the tenant status change onto it, with disabling and reactivation. Write-locking still only sets the status at this point.
5. Move access key creation and deletion.
6. Move policy writes.
7. Write-locking: narrowed grants and the refusal of writes in authorization. It comes after steps 5 and 6 because a write-locked tenant stays live, so every operation that issues grants has to be on the tenant's queue before narrowing them is safe. Disabling has no such need, since nothing issued for a disabled tenant can be used.
8. Move principal removal, bucket deletion and tenant deletion.

9. Follow-up, after the steps above have merged: authorization replies carry the grants they rest on, read before the status, and `/s3/bucket/info` is removed. See [Replies that outlive a change](#replies-that-outlive-a-change).

Each move removes that operation's locks. The four share locks in authorization go with the last writer that needs them.

## References

- [hilt#137: Reconcile tenant status with fenced stored authority](https://github.com/fil-forge/hilt/pull/137)
- [ingot#411: Map tenant write-lock denials and verify cached action boundaries](https://github.com/fil-forge/ingot/pull/411)
- [RFC: Ingot task runner](./2026-10-ingot-task-runner.md)
- [fil-one: Service Orchestrator integration guide, tenant state enforcement](https://github.com/fil-one/fil-one/blob/main/docs/service-orchestrator-integration/README.md#tenant-state-enforcement)
