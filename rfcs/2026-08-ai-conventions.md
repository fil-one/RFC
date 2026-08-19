# RFC: AI conventions

**Status:** Proposal
**Author:** Hannah Howard
**Audience:** Fil One engineers
**Date:** 2026-08-12

## TL;DR

We lean hard on AI tools now, and each of us wired them up alone. I propose we standardize three things:

1. **One writing standard.** Everyone adopts the FilOne Standard as the base of their writing style, however they want to do it. The recommended path is [workspace-setup](https://github.com/fil-one/workspace-setup). A required install waits for a monorepo.
2. **Readable prose is the author's job.** If a teammate ships you clearly unintelligible AI-generated text, you are fully empowered to hold them accountable.
3. **One home for agent guidance.** Every repo carries an `AGENTS.md`, and `CLAUDE.md` is the one-line import `@AGENTS.md`.

## Where we are

Most first drafts, code and prose, start machine-written now. What the tools know still depends on whose machine they run on. Ten of our fifty repos give agents guidance every tool can read, and two more are half-converted; six keep it in a `CLAUDE.md` only Claude sees; thirty-two offer nothing at all. The shared knowledge base and the writing rules reach a session only if that laptop happens to be wired for them. The same task run by two of us produces different code and different-sounding prose, and what any of us learns about making the tools good stays on one machine.

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

A machine-written first draft is still yours. Prose you expect other humans to read has to be readable by other humans, and shipping raw model output to a reviewer hands them your editing work. We already committed to an hour a day of review time in [dedicated review time and stacked PRs](2026-07-review-time-and-stacked-prs.md); prose a reviewer has to decode wastes it. If you read something clearly unintelligible and AI-generated, you are fully empowered to hold your teammate accountable for it. "This is unintelligible, revise so I can review" is a reasonable response to a PR.

## AGENTS.md in every repo

Write agent guidance once and every tool reads it. `AGENTS.md` is the cross-tool convention: Codex, Cursor, Copilot, and most other coding agents read it natively. Claude Code is the exception (it reads `CLAUDE.md`), and the one-line `@AGENTS.md` import is what closes that gap. The pair travels with the clone, so cloud sessions and CI get the same guidance as your terminal. The ask is that every repo carries the pair, starting with new ones. [Dedicated review time and stacked PRs](2026-07-review-time-and-stacked-prs.md) already routes its stack norm through each active repo's `AGENTS.md`, so that rollout depends on this one. The workspace carries the norm as a default too: setup.sh installs the `gh stack` tooling, and the generated workspace AGENTS.md states the stack norm for local sessions in every repo, including the thirty-two with no guidance of their own. The per-repo line still covers cloud sessions and CI.

## Rollout

1. Adopt the standard this week: run the script, or wire the style into your own setup. If the script fights your setup, tell me and I'll fix the script rather than your machine.
2. New repos start with `AGENTS.md` plus the one-line `CLAUDE.md`.
3. Backfill: six repos keep real guidance in a `CLAUDE.md` no other tool will find — `ingot`, `smelt`, `sprue`, `go-ipni-tools`, and two private repos. Convert them as we touch them; the sweep is an hour of agent work if anyone feels strongly.
