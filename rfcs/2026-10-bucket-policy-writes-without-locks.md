# RFC: Bucket policy writes without locks or preconditions

**Status:** Proposal

## Authors

- [Srdjan S](https://github.com/srdjan)

## Introduction

Bucket policy writes become unconditional and the last write wins, as on AWS.
This RFC amends [Forge S3 tenant IAM](./2026-09-forge-s3-tenant-iam.md) and the
[bucket policy ADR](https://github.com/fil-one/fil-one/blob/main/docs/architectural-decisions/2026-08-bucket-policies-m2.md):

- Bucket policies lose their `ETag` at Hilt, Ingot and the console.
- `PutBucketPolicy` and `DeleteBucketPolicy` ignore `If-Match` and
  `If-None-Match`, as AWS does.
- Hilt drops its database locks around policy writes and revocations, and the
  lock-wait failures with them.
- A write publishes its revocations inside its transaction, before it writes
  any row.

## Design

### Consistency rule

The last committed write wins, and no write waits for another. Each write is
one transaction: read the stored document and the affected keys' delegations,
publish their revocations, write the new delegations and the document, commit.
A failure at any step, a failed publish included, rolls back and leaves the old
state.

Rotation reads and replaces a key's delegations over the written bucket only
(`ReplaceBySubject`) and leaves its delegations over other buckets alone, so
writes to different buckets never interact.

Two concurrent writes to one bucket can leave a key holding both writes'
delegation sets, since neither delete sees the other's uncommitted insert.
Hilt's authorize still decides each request from the stored document, so the
extra set grants nothing there. A warm Ingot cache filled between the two
commits can keep the other write's access until the next UTC midnight, because
one of the sets is never revoked.

### Revocations before the first write

The publish runs after the transaction's reads and before its first row write.
Postgres holds a row a transaction has written until it commits, so with no row
written yet, no other write waits on Swarf. The cost is one open, idle
connection per write for a Swarf round trip. A 204 still means Swarf accepted
the revocations. A commit that fails after the publish leaves revoked
delegations stored, which Ingot's revoked-CID cache already covers: it serves
such a key from Hilt uncached until a later write commits.

### Other revoking writes

Access key deletion, principal removal, bucket deletion and tenant removal take
the same order without locks: read, publish, delete, commit. A failed publish
leaves everything in place and the call retryable.

- An access key created for a principal during its removal survives it. Hilt
  refuses the key while the principal is removed, and a later `PUT` that
  revives the principal's id brings the key back.
- Tenant removal deactivates the tenant's `did:plc` only after its publish
  succeeds, since Swarf verifies revocations against that DID.

Authorize and `/s3/bucket/info` read without share locks, and bucket info
drops its re-read check.

### Policy operations and failures

`PutBucketPolicy` replaces or creates the policy and answers 204.
`DeleteBucketPolicy` deletes it, and a bucket with no policy still answers
`NoSuchBucketPolicy`. Neither these nor `GetBucketPolicy` return an `ETag`.
Hilt drops the `etag` column from `bucket_policy` and keeps the canonical form.

These failures are removed:

- `PreconditionFailed` (412) and `InvalidPrecondition` (400) on the policy
  command.
- `ConcurrentChange` (409 `OperationAborted`) on policy writes, principal
  creation and removal, and access key creation and deletion.
- `TemporarilyUnavailable` (503) on authorize and bucket info.
- `store.ErrLockTimeout`, `store.LockTimeout` and the `lock_timeout` setting.

Object conditional writes are unchanged.

### Ingot

Ingot stops mapping the removed failures and stops setting `ETag` on the policy
operations. Its revoked-CID cache is unchanged.

### Fil One console

- The policy API takes no `If-Match` or `If-None-Match` and returns no `ETag`.
  `POLICY_CONFLICT` and the retry of a 409 are removed.
- Bucket creation writes the first policy without `If-None-Match: *`.
- The policy tab loses its stale-draft state and conflict alert. Two admins
  saving at once both succeed and the later save is stored.
- The retention check compares a non-privileged caller's document with the
  policy it read. An Owner's change that lands between that read and the write,
  including the removal of a retention grant, can be overwritten. We accept
  this race.

## Compatibility and migration

Deploy Hilt, then Ingot, then the console. A client sending a precondition gets
an unconditional write instead of a 412; the console is the only such client.
An old console reading from a new Ingot gets no `ETag` and its editor refuses
to save, so the console ships in the same release. Hilt's migration drops the
`etag` column.

## Superseded text

In [Forge S3 tenant IAM](./2026-09-forge-s3-tenant-iam.md):

- [Bucket policies](./2026-09-forge-s3-tenant-iam.md#bucket-policies): the
  `ETag`, the preconditions, the ETag half of the canonical-form paragraph, and
  the compare-and-set paragraph.
- [Bucket creation](./2026-09-forge-s3-tenant-iam.md#bucket-creation):
  `If-None-Match: *`.
- [Policy changes and delegation rotation](./2026-09-forge-s3-tenant-iam.md#policy-changes-and-delegation-rotation):
  the concurrency paragraph, its four guarantees, and the timeouts.
- [Principal removal](./2026-09-forge-s3-tenant-iam.md#principal-removal): the
  lock in step 3 and the ETag in step 2.
- [Failures](./2026-09-forge-s3-tenant-iam.md#failures): the four removed
  failures.
- [Ingot - S3 API](./2026-09-forge-s3-tenant-iam.md#ingot---s3-api): the policy
  `ETag`.
- [Example flows](./2026-09-forge-s3-tenant-iam.md#example-flows): the
  precondition steps.

In the [bucket policy ADR](https://github.com/fil-one/fil-one/blob/main/docs/architectural-decisions/2026-08-bucket-policies-m2.md):

- "Concurrent edits take a compare-and-set" under the decision.
- Requirement 3 on the storage system, the ETag for conditional edits.

## Evaluation criteria

- Concurrent writes to different buckets naming the same principal all
  succeed, and each key holds exactly the delegations the stored documents
  grant.

## References

- [Forge S3 tenant IAM](./2026-09-forge-s3-tenant-iam.md), the RFC this one
  amends.
- [Bucket access by region](https://github.com/fil-one/fil-one/blob/main/docs/architectural-decisions/2026-08-bucket-policies-m2.md),
  the ADR this one amends.
- [AWS PutBucketPolicy](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutBucketPolicy.html),
  which defines no precondition headers.
- [AWS conditional writes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html),
  which cover objects only.
- FIL-1646.
