# RFC: Try mounting a Fil One bucket as a folder

**Status:** Proposal
**Author:** [James Kurz](https://github.com/jameskurz-filecoin)
**Audience:** Fil One / Forge engineers
**Date:** 2026-08-19

## TL;DR

Some backup and data tools want a directory path, not the S3 API. AWS already
ships [Mountpoint](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mountpoint.html)
for that. I want to point it at a real Fil One bucket in Forge staging and see
what happens.

This RFC is **not** asking engineering to adopt or maintain a 100k-line Rust
fork. I already have one (`posixmount` / `filfs`). Presenting that as the
product was the wrong ask. The first experiment is upstream Mountpoint, maybe
rclone mount as a comparison. If that is enough, we write a docs page and
stop. If it fails in a way docs cannot fix, I will come back with a short list
of actual patches.

What I need now: a staging bucket and access key, and a Linux box that can
reach the regional S3 endpoint. Nothing gets installed on Forge servers.

## What this is

[Mountpoint for Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mountpoint.html)
is a FUSE client. You run it on a Linux machine. It talks S3. The bucket shows
up as a folder. It is not a POSIX filesystem: no atomic rename, no in-place
overwrite, no file locks. The files are still S3 objects. That is the whole
product.

`filfs` is my fork of Mountpoint v1.23.0. `filfs-csi` is my fork of the
Mountpoint CSI driver v2.7.0. Both live in
[fil-one/posixmount](https://github.com/fil-one/posixmount). They are clients.
They do not run on storage servers. There is no new service to deploy into
`eu-central-3`.

Credentials are a normal AWS key file, the same kind the AWS CLI uses. Nothing
is compiled into the binary. I had added an optional "region profile" (endpoint
URL, allowed operations, size limits) that can be baked into a build. That is
config, not secrets, and it is not required for the first test.

Kubernetes CSI would run a node plugin on cluster nodes. That is a real
cluster component, and I am **not** asking for it here.

## Why bother

FIL-281, in one sentence: a long Proxmox-style backup over a mount died in the
middle of a multipart upload, and we could not finish a restore. Native S3 to
the same bucket worked. I want to reproduce that on staging, then restore and
checksum.

The other real cases are tools that only know how to write to a path. CSI
volumes for pods are a later question.

## The fork, honestly

Hannah and Alan are right: taking on Mountpoint is taking on ~100k lines of
Rust that nobody on this team owns. That is a significant risk. I should have
said so on page one.

Why I forked it at all:

- Pin the Fil One endpoint so a binary cannot be pointed at random S3.
- Return a clean "not supported" for rename / append / random write, instead of
  looking like a POSIX disk and then corrupting a backup.
- Tighter credential-file permissions.
- A CSI driver later.

None of that is a reason to skip the obvious test: run **upstream Mountpoint**
against Fil One S3. If rclone mount or Mountpoint works, the product is a
documentation page, which is the amount of code Alan asked for.

I still think we may want a thin wrapper later. I do not think we should
decide that before seeing vanilla Mountpoint fail at something we care about.

FIL-917, in one sentence: we cannot yet mount only a prefix / folder inside a
bucket. First test is the whole bucket.

Clockwork does not belong in this RFC. Ignore any earlier mention of it.

## What I am asking for

1. A Forge staging bucket and a bucket-scoped key. `eu-central-3` is fine if
   that is the staging region we already use; the region does not matter to the
   client.
2. Network access from a Linux amd64 or arm64 machine to that S3 endpoint.
3. Agreement that the first experiment is upstream Mountpoint (and maybe
   rclone mount), not `filfs`.
4. If that works for backup/restore, we write a docs page and stop. If it
   fails, I open a small follow-up with the actual gap.

## What I am not asking for

- Shipping `filfs` or the CSI driver as a Fil One product.
- Engineering owning the Rust fork.
- Deploying anything onto Forge servers.
- Dynamic provisioning, macOS/Windows clients, or a claim that this is POSIX.
- Anything involving Clockwork.

The four open posixmount PRs
([#18](https://github.com/fil-one/posixmount/pull/18)–[#21](https://github.com/fil-one/posixmount/pull/21))
should wait. They were written as if we had already decided to ship the fork.

## How the staging test would run

On a Linux **client**, not on a Forge node:

1. Create a bucket and key through the existing Fil One console or API.
2. Confirm AWS CLI or boto3 can PUT/GET against it.
3. Mount with upstream Mountpoint using that same key.
4. Copy bytes both ways and checksum.
5. Reproduce the FIL-281 backup: long multipart write, interrupt it, restore,
   checksum.
6. Unmount. Revoke the key if needed. The bucket is still just a bucket.

## Decisions needed

1. Can I get a staging bucket and key for this client-side test?
2. Do we agree the first test is upstream Mountpoint, not adopting the fork?
3. If Mountpoint is good enough, is a docs page the product we want to ship?

## References

- [AWS Mountpoint](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mountpoint.html)
- [posixmount repo](https://github.com/fil-one/posixmount) (the fork; not the
  proposal)
- [UPSTREAM.md](https://github.com/fil-one/posixmount/blob/main/UPSTREAM.md)
  (Mountpoint v1.23.0 and CSI driver v2.7.0 pins)
