# RFC: Dedicated review time and stacked PRs

**Status:** Superseded by [AI conventions](2026-08-ai-conventions.md) — the norms live there; this document remains the evidence base and tooling evaluation
**Author:** Hannah Howard
**Audience:** Forge engineers
**Date:** 2026-07-31

## TL;DR

We are producing code much faster than we are reviewing it. I propose two commitments:

1. **Dedicated review time.** One hour a day, held on the calendar, and a first response to every review request within one business day.
2. **Stacked PRs for big work.** A PR should be reviewable in under an hour. Anything larger ships as a stack of small dependent PRs, and our Claude Code workflows get taught to produce stacks too. For tooling I recommend GitHub's native stacked pull requests (public preview as of July 30), with spr as the fallback where the preview hasn't reached us, and Graphite held in reserve.

## Where we are

We are currently moving faster than we ever have as a development team. Since June 1, we've merged 221 PRs. The median PR review process is healthy: 6 files, merged inside two days. It's the tail that's the problem: 34 merged PRs touched 30+ files, 12 touched 60+, and p90 time-to-merge is 11 days. Unfortunately, this tail covers our most important work: ucantone #30 (attested signatures, 114 files), ingot #35 and #36 (Hilt integration, 103 files each), piri #22 (Curio PDP pipeline, 86 files).

What's wrong? In short, building is more fun than reviewing, AI has made building a lot more fun and a lot more fast. So our bottleneck is often review and corresponding context management -- having to keep up with the rapid pace of new code committed and new ideas explored.

## What the evidence says

- **Review quality collapses with size.** The SmartBear/Cisco study (2,500 reviews, 3.2M lines of code) found defect detection several times better below 200 lines, with reviewers wearing out after about an hour, capping useful review at 300–400 lines. Google's guidance draws the same line: 100 lines is a reasonable change, and "a 200-line change in one file might be okay, but spread across 50 files it would usually be too large." Google's measured median change is 24 lines. Our 60-file PRs are far outside every one of these bounds.
- **One business day is the standard, and it costs about three hours a week.** Google's stated maximum is one business day to first response. Their measured median for an entire review is under 4 hours, at a measured cost of 3.2 hours per developer per week on average.
- **Stacking is how fast authors keep changes small.** Meta's workflow: send the first small step for review and build the next step on top while waiting; "commits become smaller, easier to reason about, and easier to review." Google prescribes the identical loop. Small changes are the goal; stacking makes them affordable when the author is fast. Nothing is faster than an agent.
- **AI shifts the work to verification.** DORA 2025 reports that "the time saved during initial code or content generation is often re-allocated to verification overhead," and that AI adoption correlates with higher delivery throughput and higher instability at the same time. DX's 400-company sample measured 51.9% of code AI-authored in Q2 2026, with median PR size nearly doubled in a year.

## Proposal 1: dedicated review time

Each of us holds one hour a day on the calendar for review, checks the review queue before starting new build work, and gives a first response to any review request within one business day. Google's measured cost is about 3 hours a week, so an hour a day is deliberately above steady state: we have a review backlog to drain and agents multiplying our output. Revisit the number in a month.

## Proposal 2: stacked PRs for anything big

A PR should be reviewable in under an hour, which the evidence puts at roughly 400 changed lines, though the content is just as important -- critical auth logic takes longer than code-gen'd serialization for example. Therefore, bigger work must ship as a stack of dependent PRs, each one a reviewable step.

The norm binds our agents too. Each active repo's AGENTS.md gets an instruction that work expected to exceed the guide ships as a stack, and Claude Code gets the stack tooling below.

## Tooling

I looked at GitHub's new native support, Graphite, spr (which we have already used), and the wider field. An agent has to be able to drive the whole loop — create, update, restack, submit — non-interactively from a terminal.

### Recommended: GitHub native stacked pull requests

GitHub shipped stacked PRs to public preview on July 30, rolling out to all repositories "over the coming days." A stack is a linear chain of PRs in one repo. Reviewers see each layer's diff on its own plus a stack map; branch protections and required checks apply to every PR in the stack; when a lower PR merges, the ones above rebase and retarget automatically. Squash merge is supported.

The agent path is first-party. The `gh stack` CLI extension (gh >= 2.90) covers the whole loop non-interactively — init, add, push, `submit --auto`, sync, rebase, `view --json`, `merge --yes --squash`, with defined exit codes — and `gh skill install github/gh-stack` installs a skill that teaches the workflow to coding agents; GitHub's rollout docs name Claude Code specifically. There is also a REST stacks API. No new vendor and no extra GitHub App permissions (GitHub has not published plan gating for the preview), and reviewers need zero setup because review happens in the PR UI we already use.

It is a day-old preview and with that comes caveats. Enablement is per-repo, but it has already reached us for the most part: the stacks API responds on piri, ingot, sprue, and fil-one, and piri has a live five-PR stack (#48–#53, opened July 31). Other features like merge-queue are incomplete, and numerous bugs and inconsistencies can be found on community discussions and the Hacker News post of the announcement, but presumably GitHub will get a lot of pushback on that and improve the product quickly.

### Graphite

Graphite has the most complete workflow of the three and the only vendor-built MCP server for splitting large diffs into stacks (beta, bundled in the CLI); the gt CLI is fully non-interactive. The CLI and MCP work on the free tier, but the free tier covers personal repos only, so org use starts at Starter, $20 per user per month billed annually ($1,440/year for six seats); the merge queue and unlimited AI review are on Team at $40 ($2,880/year). Seats are counted automatically and include anyone whose PR the Graphite agent reviews. Automatic restack after a merge fires only when the merge goes through Graphite rather than the GitHub UI, and its GitHub App wants read-write on actions, checks, contents, pull requests, and workflows. Lock-in is modest; branches and PRs stay ordinary GitHub objects. Verdict: hold in reserve. If the native preview is still rough in a month, the 30-day trial (no card required) is the next move.

### spr (ejoffe/spr)

Free, and proven here: the RAG work went out as a seven-PR series in fil-one (#449–455) using spr, and the 207 review comments on those PRs survived its force-push model without a single thread going outdated. One commit = one PR, amend and `git spr update`; agents drive it trivially. The costs: it is a one-maintainer project (last release April 2026, maintainer still responsive in July); every stack update re-pushes the layers below and re-triggers their CI; two people editing one stack can clobber each other; and failures print Go panics instead of error messages. Verdict: the known-working fallback for repos the preview hasn't reached. Nothing new to learn.

### The rest of the field

av (aviator-co/av) is the best of the remainder: free, MIT, actively maintained, fully non-interactive against the GitHub API. If the native preview disappoints and we want an actively maintained tool rather than spr, av is the fallback. ghstack, git-machete, git-town, git-branchless, jj, Sapling, and the charcoal Graphite fork don't beat these three for a small team on standard GitHub squash merges.

## Rollout

1. Next week: everyone installs gh >= 2.90 and `gh extension install github/gh-stack`. The preview has already reached piri, ingot, sprue, and fil-one; check any other repo with `gh api repos/<org>/<repo>/stacks`.
2. Claude Code users run `gh skill install github/gh-stack`; each active repo's AGENTS.md gets the stack norm.
3. Pilot: piri's open stack is the first data point; the next substantial feature in piri or ingot ships as a native stack too. Where the preview hasn't landed and the work can't wait, spr.
4. The review hour needs no tooling and starts now.
5. Reassess in four weeks. Preview stable: adopt as standard. Still rough: Graphite trial or av.

## Decisions needed

- **The hour.** I propose one hour a day. Google's measured average is about three hours a week, so this is deliberately generous while the backlog drains.
- **Pilot now vs. wait.** The GitHub preview is days old and will have sharp edges; spr covers the gaps. I propose pilot now.
- **Size guardrails.** Social norm only, or a soft CI warning above ~400 changed lines? I lean social to start.

## References

- SmartBear/Cisco code review study: https://static0.smartbear.co/support/media/resources/cc/book/code-review-cisco-case-study.pdf
- Google eng-practices — small CLs and review speed: https://google.github.io/eng-practices/review/developer/small-cls.html and https://google.github.io/eng-practices/review/reviewer/speed.html
- Sadowski et al., "Modern Code Review: A Case Study at Google" (ICSE-SEIP 2018): https://sback.it/publications/icse2018seip.pdf
- Meta on stacked commits: https://engineering.fb.com/2022/11/15/open-source/sapling-source-control-scalable/
- METR randomized trial (2025) and follow-up (2026): https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/ and https://metr.org/blog/2026-02-24-uplift-update/
- DORA 2025 on AI tensions: https://dora.dev/insights/balancing-ai-tensions/
- DX on AI-authored code share and PR size: https://getdx.com/blog/ai-authored-code-has-nearly-doubled/
- GitHub stacked PRs — changelog and docs: https://github.blog/changelog/2026-07-30-stacked-pull-requests-are-now-in-public-preview/ and https://docs.github.com/en/pull-requests/how-tos/stacked-pull-requests
- gh-stack CLI and agent skill: https://github.com/github/gh-stack
- Graphite pricing: https://graphite.com/pricing
- ejoffe/spr: https://github.com/ejoffe/spr
- av: https://github.com/aviator-co/av
