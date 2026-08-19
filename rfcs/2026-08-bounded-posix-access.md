# RFC: Try mounting a Fil One bucket as a folder

**Status:** Proposal
**Author:** [James Kurz](https://github.com/jameskurz-filecoin)
**Audience:** Fil One engineers
**Date:** 2026-08-19

## TL;DR

I want to mount a Fil One bucket as a directory using **upstream** AWS
[Mountpoint](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mountpoint.html),
from a laptop, against a staging bucket.

This RFC does **not** ask anyone to adopt, review, or maintain
`fil-one/posixmount`. That repo is a Mountpoint fork I started too early. The
deep-POSIX work in it is not even turned on. If stock Mountpoint works, the
deliverable is a docs page (`--endpoint-url`, `--force-path-style`, `--region`).
If a real backup dies, I come back with the actual failure — not a 100k-line
Rust tree.

What I need: a staging **bucket and key**. I create them in the existing
console if I can; if I cannot, I need someone to mint one. Nothing gets
installed on Forge.

## What this is

Mountpoint is a FUSE client. You run it on Linux. It talks S3 with an access
key. The bucket shows up as a folder. It is not POSIX: no atomic rename, no
in-place overwrite, no file locks. The files are still S3 objects.

`filfs` is my fork of Mountpoint v1.23.0 (plus the CSI driver v2.7.0) in
[fil-one/posixmount](https://github.com/fil-one/posixmount). It is also a
client. It does not run on storage servers. There is no new service.

Credentials are a normal AWS key file, the same as the CLI. They are not in
the binary. (An unmerged branch compiles in a JSON **config** blob — endpoint,
allowed operations, size limits — for `eu-central-3`. That is not a secret, and
it is not part of this test.)

## Why bother

Some backup and data tools want a path, not the S3 API. We have not actually
tried Mountpoint (or rclone mount) against Fil One S3.

The motivating failure, in one sentence: a long Proxmox-style backup over a
mount died in the middle of a multipart upload, and we could not finish a
restore. Native S3 to the same bucket worked.

## The fork, honestly

I forked too early. This RFC does not need that fork.

I looked at the code before writing this. On `posixmount` main:

- The client still talks ordinary SigV4 S3.
- Staged write-back (the thing that would be “deeper POSIX”) defaults to off
  and then refuses anyway, because the CRT client reports multipart recovery as
  unavailable.
- `filfs` currently wants a control-plane URL before it will mount. Vanilla
  Mountpoint does not. That extra requirement is a reason **not** to use the
  fork for the first test.
- Prefix-only mounts already exist in upstream Mountpoint (`--prefix`). We do
  not need a fork for “folder inside a bucket.”

Kubernetes CSI would be a DaemonSet on **whoever’s cluster is mounting**, not
on Forge. It is out of this RFC.

The four open posixmount PRs
([#18](https://github.com/fil-one/posixmount/pull/18)–[#21](https://github.com/fil-one/posixmount/pull/21))
should stay parked.

## What I am asking for

1. A staging bucket and a bucket-scoped key. I will use the existing console if
   I have access. `eu-central-3` is only the bucket’s region, not a place we
   ship code.
2. I run Mountpoint on a client that can already reach Fil One S3 (laptop or
   any Linux box). Nobody deploys onto Forge nodes.

First command, after AWS CLI PUT/GET works with path-style addressing:

```bash
mount-s3 "$BUCKET" /mnt/filone \
  --endpoint-url https://ingot.staging.fil.one \
  --region eu-central-3 \
  --force-path-style
```

`--force-path-style` is the Fil One-specific bit. Virtual-host style does not
work on this gateway.

Then: copy bytes both ways and checksum. Then try the long backup, interrupt
it, restore, checksum. Unmount. Revoke the key if needed. The bucket is still
just a bucket.

If that works, we write a docs page and stop. If it fails, I open a small
follow-up with the actual gap.

## What I am not asking for

- Shipping `filfs` or the CSI driver as a Fil One product.
- Engineering owning the Rust fork.
- Installing anything on Forge.
- Dynamic provisioning, macOS/Windows, or a claim that this is POSIX.

## Decisions needed

1. Can I use (or be given) a staging bucket and key for this client-side test?
2. Do we agree the first test is upstream Mountpoint, not the fork?
3. If Mountpoint is good enough, is a docs page the product we want to ship?

## References

- [AWS Mountpoint](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mountpoint.html)
- [posixmount](https://github.com/fil-one/posixmount) (the parked fork)
