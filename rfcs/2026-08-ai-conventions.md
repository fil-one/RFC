# RFC: AI conventions

**Status:** Proposal
**Author:** Hannah Howard
**Audience:** Fil One engineers
**Date:** 2026-08-12

## TL;DR

We lean hard on AI now, and each of us wired it up alone. This RFC sets the conventions for how we work with AI: the writing standard, the guidance agents get, and how AI-built code ships and gets reviewed. It supersedes [dedicated review time and stacked PRs](2026-07-review-time-and-stacked-prs.md); the norms live here, and that document keeps the evidence and the tooling evaluation.

1. **One writing standard.** Everyone adopts the FilOne Standard as the base of their writing style, however they want to do it. The recommended path is [workspace-setup](https://github.com/fil-one/workspace-setup). A required install waits for a monorepo.
2. **Readable prose is the author's job.** If a teammate ships you clearly unintelligible AI-generated text, you are fully empowered to hold them accountable.
3. **One home for agent guidance.** Every repo carries an `AGENTS.md`, and `CLAUDE.md` is the one-line import `@AGENTS.md`.
4. **Stacked PRs are the default for big work.** Anything bigger than about an hour of review ships as a stack of small dependent PRs. Tooling adoption works like the writing standard: however you like, workspace-setup recommended.
5. **The review hour stays required.** One hour a day, held on the calendar.
6. **An accelerated workflow for AI-built work.** Spec first with human review, then review and merge your own stack if you are confident. Human review on request, always. Adversarial AI review before submitting is recommended.

## Where we are

Most first drafts, code and prose, start machine-written now. What the tools know still depends on whose machine they run on. Ten of our fifty repos give agents guidance every tool can read, and two more are half-converted; six keep it in a `CLAUDE.md` only Claude sees; thirty-two offer nothing at all. The shared knowledge base and the writing rules reach a session only if that laptop happens to be wired for them. The same task run by two of us produces different code and different-sounding prose, and what any of us learns about making the tools good stays on one machine.

We also merge code faster than we review it, and the biggest PRs carry the most important work. The measurements and the external evidence are in [dedicated review time and stacked PRs](2026-07-review-time-and-stacked-prs.md).

## Generated writing sounds like us

[knowledge-base/writing-style.md](https://github.com/fil-one/knowledge-base/blob/main/writing-style.md) is the FilOne Standard: the writing rules for anything reader-facing, because reading overly-AI-flavored prose is getting to be significant cognitive overhead at work, and costs us credibility for anything external. Adopt it as the base of your own setup, and personalize on top as long as the spirit holds.

The recommended path is workspace-setup:

```bash
git clone git@github.com:fil-one/workspace-setup.git
./workspace-setup/setup.sh ~/projects/filecoin
```

You get a workspace that symlinks your fil-forge and fil-one clone trees, clones the shared knowledge base, and installs the FilOne Standard as Claude Code's default output style, so commit messages, PR descriptions, docs, and plain terminal replies follow it. It will not disturb the checkouts you already have. Start sessions from the workspace directory and every session gets the knowledge base, routing rules, and writing standard. If you'd rather wire the style yourself (a copy in `~/.claude/output-styles`, an import in your own agent config, instructions to whatever tool you use), that's fine too, as long as generated prose starts from the standard.

The required, synced install waits for a monorepo, if we move to one. Claude Code can pin a shared plugin from a repo's checked-in settings, but across fifty repos that means seeding fifty pointer files; in a monorepo it is one file, and the standard ships with the repo.

## Readable prose is the author's job

A machine-written first draft is still yours. Prose you expect other humans to read has to be readable by other humans, and shipping raw model output to a reviewer hands them your editing work. Prose a reviewer has to decode burns the review hour. If you read something clearly unintelligible and AI-generated, you are fully empowered to hold your teammate accountable for it. "This is unintelligible, revise so I can review" is a reasonable response to a PR.

## AGENTS.md in every repo

Write agent guidance once and every tool reads it. `AGENTS.md` is the cross-tool convention: Codex, Cursor, Copilot, and most other coding agents read it natively. Claude Code is the exception (it reads `CLAUDE.md`), and the one-line `@AGENTS.md` import is what closes that gap. The pair travels with the clone, so cloud sessions and CI get the same guidance as your terminal. The ask is that every repo carries the pair, starting with new ones.

## Stacked PRs are the default for big work

A PR should be reviewable in under an hour, which the evidence puts at roughly 400 changed lines, though the content matters as much as the count. Bigger work ships as a stack of small dependent PRs, each one a reviewable step. This is the default and expected.

The recommended tooling is GitHub's native stacked pull requests, driven by the `gh stack` CLI extension and its agent skill, with spr as the fallback where the preview hasn't reached a repo; the full evaluation (Graphite, av, the rest of the field) is in [dedicated review time and stacked PRs](2026-07-review-time-and-stacked-prs.md). Install it however you like: workspace-setup installs the extension and the agent skill for you.

The norm binds our agents too. Each active repo's `AGENTS.md` carries the stack instruction, and the generated workspace AGENTS.md states it as a default for local sessions in every repo, including those with no guidance of their own.

## The review hour stays required

Each of us holds one hour a day on the calendar for review, checks the review queue before starting new build work, and gives a first response to any review request within one business day. The accelerated workflow shifts the queue toward specs and away from routine PRs.

## Accelerated AI development workflow

When taking on a sizable ticket (one that won't fit a single reviewable PR), you may use the following process to accelerate delivery:

1. Submit an ADR, RFC, or architectural prose spec of what you are going to implement, and get human review from a colleague on that.
2. Submit the rest of the PR stack, and review and merge it yourself if you are confident.

For smaller tasks that fit in a single reasonably sized PR, you may also simply review and merge your own code if you are confident. You can always request human review on any code if you feel it's necessary.

Adversarial AI review on your PRs before submitting is recommended but not required. Everyone should have received an invite to our ChatGPT Business workspace (the plan formerly called Team), which comes with Codex; tell me if you haven't. I recommend turning on automatic Codex code reviews for the repos you work in.

### Turning on automatic Codex reviews

Codex reviews flag only P0 and P1 issues, so the signal is terse. Setup, per [OpenAI's GitHub integration docs](https://learn.chatgpt.com/docs/third-party/github):

1. Sign in to ChatGPT with your work account and set up Codex cloud, connecting GitHub and choosing our repos when prompted.
2. Open [chatgpt.com/codex/settings/code-review](https://chatgpt.com/codex/settings/code-review) and select the repo. Changing these settings takes GitHub push or admin permission on it.
3. Turn on **Code review**, then **Automatic reviews**.
4. Under **Repository preferences**, pick which PRs get reviewed. If the options match security review's (**Review all PRs**, **Review team PRs**, **Follow personal**), choose **Follow personal**, so reviews follow each contributor's own opt-in instead of firing on every PR in the repo. The docs only spell these choices out for security review.
5. Check it works on any open PR with a comment: `@codex review`. A `## Code Review Rules` section in a repo's `AGENTS.md` tunes what it flags.

## Rollout

1. Adopt the writing standard this week: run the script, or wire the style into your own setup. If the script fights your setup, tell me and I'll fix the script rather than your machine. Re-running the script also installs the stack tooling.
2. Turn on automatic Codex reviews (steps above).
3. New repos start with `AGENTS.md` plus the one-line `CLAUDE.md`. Backfill: six repos keep real guidance in a `CLAUDE.md` no other tool will find — `ingot`, `smelt`, `sprue`, `go-ipni-tools`, and two private repos. Convert them as we touch them; the sweep is an hour of agent work if anyone feels strongly.
4. The review hour continues, and the accelerated workflow is available starting now.
