# RFC: Shelve the Mountpoint fork; document off-the-shelf if we need a folder later

**Status:** Proposal
**Author:** [James Kurz](https://github.com/jameskurz-filecoin)
**Audience:** Fil One engineers
**Date:** 2026-08-19

## TL;DR

Some workloads cannot use S3 directly; they want a folder. I built a prototype
on AWS [Mountpoint](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mountpoint.html).
Adopting it would make this team own ~100k lines of Rust. That is not a current
priority. Off-the-shelf is a good outcome.

**Decision:** do not adopt the fork. Extract anything reusable from
[posixmount](https://github.com/fil-one/posixmount) (path-style notes, tests,
credential-file rules), mark the repo a spike, and stop the review. If a real
customer later needs a mount, try stock Mountpoint or rclone from a laptop
first. Only then consider a tiny patch or a maintained third-party client.

## What I looked at

Mountpoint is a FUSE client you run on Linux. It talks S3 with a normal access
key. The bucket shows up as a folder. It is not POSIX: no rename, no append, no
file locking, no in-place edit.

`filfs` in posixmount is my fork of Mountpoint v1.23.0 (and its CSI driver).
It is also a client. Nothing from it runs on Forge. Credentials are a key
file, not baked into the binary.

I forked too early. The “deeper POSIX” write-back in that repo is not even
turned on. `filfs` on main also wants a control-plane URL before it will
mount. Vanilla Mountpoint does not. So the fork is the wrong tool even for a
first test.

The open posixmount PRs should close or stay parked. No CSI.

## Decision requested

1. **Do not adopt the fork.** Engineering is not taking on that Rust tree.
2. **Treat posixmount as a spike.** Harvest anything useful (path-style on
   `ingot.staging.fil.one`, 0600 credential files, “this is still S3”
   semantics). Then shelve it.
3. **If a real customer later needs a mount,** try this first — from a laptop,
   not on Forge:

   ```bash
   mount-s3 "$BUCKET" /mnt/filone \
     --endpoint-url https://ingot.staging.fil.one \
     --region eu-central-3 \
     --force-path-style
   ```

   If that works, the product is a docs page. If it fails, we talk about a
   tiny patch or a maintained third-party client — not reviving the fork by
   default.

## Alternatives

- **Do nothing now.** My recommendation. No customer is blocked on this.
- **Write the Mountpoint docs page without more code.** Fine if someone hits
  the need next quarter.
- **Buy/use rclone mount or another maintained client.** Also fine; compare
  only when we have a workload.
- **Keep building filfs.** No. The maintenance cost is the whole point of
  this RFC.

## What I am not asking for

- A staging deploy onto Forge.
- Engineering ownership of posixmount.
- Clockwork, CSI, desktop clients, or POSIX claims.

## References

- [AWS Mountpoint](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mountpoint.html)
- [posixmount](https://github.com/fil-one/posixmount) (spike to harvest, then shelve)
