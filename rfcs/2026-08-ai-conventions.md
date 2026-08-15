# RFC: AI conventions

**Status:** Proposal
**Author:** Hannah Howard
**Audience:** Fil One engineers
**Date:** 2026-08-12

## TL;DR

We lean hard on AI tools now, and each of us wired them up alone. I propose we standardize three things:

1. **One bootstrap.** Everyone sets up their machine with the [workspace-setup](https://github.com/fil-one/workspace-setup) script.
2. **One writing standard.** Generated prose follows the FilOne Standard, which the workspace makes Claude Code's default output style.
3. **One home for agent guidance.** Every repo carries an `AGENTS.md`, and `CLAUDE.md` is the one-line import `@AGENTS.md`.

## Where we are

Most first drafts, code and prose, start machine-written now. What the tools know still depends on whose machine they run on. Ten of our fifty repos give agents guidance every tool can read, and two more are half-converted; six keep it in a `CLAUDE.md` only Claude sees; thirty-two offer nothing at all. The shared knowledge base and the writing rules reach a session only if that laptop happens to be wired for them. The same task run by two of us produces different code and different-sounding prose, and what any of us learns about making the tools good stays on one machine.

## One bootstrap script

```bash
git clone git@github.com:fil-one/workspace-setup.git
./workspace-setup/setup.sh ~/projects/filecoin
```

You get a workspace directory that symlinks to your fil-forge and fil-one clone trees, clones the shared knowledge base, and writes the AGENTS.md, CLAUDE.md, and Claude Code settings that tie it together. It asks before installing the optional ai-dev process module; declining is fine. It will not disturb the checkouts you already have: clones stay wherever you keep them, and the workspace reaches them through symlinks. Mine still points at `~/projects/go/src/github.com` — nothing moved, and I work from a much simpler path than the full Go tree.

Start sessions from the workspace directory. That gives every machine the same knowledge base, routing rules, and writing standard.

## Generated writing sounds like us

[knowledge-base/writing-style.md](https://github.com/fil-one/knowledge-base/blob/main/writing-style.md) is the FilOne Standard: the writing rules for anything reader-facing, because reading overly-AI-flavored prose is getting to be significant cognitive overhead to work, and costs us credibility for anything external. AGENTS.md imports the rules into every session started from the workspace, and setup.sh also installs them as the FilOne Standard output style, so commit messages, PR descriptions, docs, and plain terminal replies follow them too. On a new workspace the style is the default; if you already have a `.claude/settings.local.json`, the script leaves it alone and prints the line to add. Rule changes go to knowledge-base by PR; pull your clone and re-run setup.sh to pick them up.

## AGENTS.md in every repo

Write agent guidance once and every tool reads it. `AGENTS.md` is the cross-tool convention: Codex, Cursor, Copilot, and most other coding agents read it natively. Claude Code is the exception (it reads `CLAUDE.md`), and the one-line `@AGENTS.md` import is what closes that gap. The ask is that every repo carries the pair, starting with new ones.

## Rollout

1. Run the script this week. It takes a few minutes; if it fights your setup, tell me and I'll fix the script rather than your machine.
2. Existing workspace: add the `outputStyle` line yourself — the script never edits a settings file you already have.
3. New repos start with `AGENTS.md` plus the one-line `CLAUDE.md`.
4. Reassess in a month: if the output style feels heavy in day-to-day coding, we flip it to opt-in.

## Decisions needed

- **Default-on output style.** The style shapes how Claude talks in every session, including casual ones. I propose default-on with personal opt-out (delete the `outputStyle` line).
- **Backfill.** Six repos keep real guidance in a `CLAUDE.md` no other tool will find. Convert them in one sweep now, or as we touch them? I lean as-touched — the sweep is an hour of agent work if anyone feels strongly.
