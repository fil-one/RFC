# `fil-forge` CI: standardization proposal

**Status:** Proposal  
**Author:** Petra Jaros  
**Audience:** Fil One engineers  
**Date:** 2026-06-26

## Where we are

CI across the `fil-forge` repos has grown without a deliberate standard. Two shared systems exist, both only partly adopted:

- **ipdxco Unified CI** — the test/lint wrappers and the release-tagging flow. Good coverage (race, lint, coverage, codegen check), but only `indexing-service` is wired into the sync; everyone else carries a drifting hand-copy, and three repos (`guppy`, `go-ucanto`, `ucantone`) skip it entirely — `ucantone` has no lint job at all. The libraries don't use the release-tagging flow — they're currently consumed at commit pseudo-versions, not tagged releases.
- **libforge reusable workflows** — the cross-repo workspace test. Works well; the model for anything else we share.

A few going-concern repos (`forgectl`, `ucantool`) have no test/check CI at all. Org-wide, there is no dependency automation, no vulnerability scanning, and nothing pinned to a commit SHA.

## Not in scope

- **The `fil-one` repos.** The `fil-forge` repos have much more in common, and much of this wouldn't apply to the `fil-one` repos, so this is just about `fil-forge`.
- **Deployment and service image/binary publishing.** We're only shipping library code while Forge moves to Fil One, so this is on hold and left alone for now (the current deploy tooling will be reworked when we get there).
- `storetheindex` (upstream fork), `filecoin-services` (Solidity), and the one-off/docs repos.
- `go-ucanto` (deprecated for `ucantone` and no longer active).

## Proposal

1. **Pin Actions to SHAs and add Dependabot** to manage the bumps (it also covers go modules). One rule, no floating tags.
2. **Standardize on Unified CI** for test/lint: normalize the wrapper files, bring `guppy`/`ucantone` onto it, and add it to the going-concern repos that have no CI today (`forgectl`, `ucantool`).

## Sequence

- **First:** Dependabot org-wide; pin + normalize the wrappers on repos already using them; add CI to `forgectl` & `ucantool`. Should be completely straightforward.
- **Next:** Migrate `guppy` and `ucantone` onto Unified CI. Fairly straightforward, but might run into failures (including maybe real bugs that weren't caught before) that need fixing.

## Decisions needed

- **Pin vs. bot.** Pinning to SHAs means _not_ running the Unified CI distribution bot (it rewrites files back to a floating tag). Recommended: pin.
- **Dependabot vs. Renovate.** Recommended: Dependabot (simpler, per-repo). (I'm actually a fan of Renovate, but I think it's a little more effort to set up, and not worth it right now. Maybe later.)
- Migrating the bespoke repos changes their branch-protection check names; update those in the same step.

## Open question for the future (not blocking)

- **Library versioning.** Libraries are consumed at commit pseudo-versions today, not tagged releases. Whether to use the Unified CI release-tagging flow for our eventual tagged releases is a separate question we don't need to settle here.
