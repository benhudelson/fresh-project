# Next-session kickoff — skills repo

## Anchor

- **Repo:** `/Users/benhudelson/.claude/skills` (GitHub: `benhudelson/skills`, private)
- **Branch:** `main` (default). **Anchor commit:** `b90e1a0` — "Initial commit: skills"
- **Date generated:** 2026-06-28. Treat the above commit + this file as ground truth.
- This is the **first** kickoff for this repo — no prior kickoff to reconcile against.

## Working directory + branch + date

- Working dir: `/Users/benhudelson/.claude/skills`
- Branch: `main`
- Today's date: (use the date the runtime injects)

## Hard operating rules

This repo has **no root `CLAUDE.md`**; the binding rules are the user's global `~/.claude/CLAUDE.md`. Key ones:

- **Ask, don't assume.** Clarify intent before writing code. Ben gates scope explicitly (he chose "just the analysis" twice this session — do not start refactors unprompted).
- **Simplest solution first**; don't add unrequested abstraction; don't touch unrelated code.
- **Flag uncertainty explicitly**; verify Claude/Anthropic + LLM facts against docs rather than asserting from memory.
- **Tables over prose** for parallel-structured output (see the `step-table` skill).
- **No `&&`/`;` chaining** of separately-allowed Bash commands — use parallel tool calls.
- Commit/push only when asked. (Invoking a handoff/skill that documents committing is itself the authorization.)

## What landed this session (verified)

| Item | State |
| --- | --- |
| `git init` + initial commit `b90e1a0`; `.gitignore` for `.DS_Store` | ✅ done |
| Created private GitHub repo `benhudelson/skills`, pushed `main` | ✅ done |
| Added collaborator **behudelson** with **admin** access — invite pending acceptance | ✅ done (acceptance not confirmed) |
| Full analysis of all 7 skills for project/platform/person coupling | ✅ delivered (analysis only; no skill files changed) |
| Verified Claude Code skill precedence via claude-code-guide agent | ✅ (agent-reported from official docs; see Verification debt) |

Repo contents: 7 skills — `handoff-kickoff`, `worktree-handoff`, `publish-spec`, `pursue`, `resume`, `roadmap-phase-kickoff`, `step-table` — plus `.claude/settings.local.json`, `.gitignore`.

## In-flight at handoff

- Topic when the session ended: **how to manage repo-specific skills** so project/platform context isn't baked into global skills.
- Last substantive exchange: Ben asked whether `roadmap-phase-kickoff` could live in *both* global and coachme with the **coachme copy winning inside that repo**. Answer established: **no** — Claude Code precedence is Enterprise → Personal (`~/.claude/skills/`) → Project (`./.claude/skills/`) → Bundled, so a global skill **shadows** a same-named project skill. (Ben declined to store this as memory.)
- **No decision made** to actually execute any relocation/refactor. Ben has kept this advisory ("just the analysis").

## Open work (carry-forward, classified)

1. **Two concrete defects in skill files** (found, not fixed):
   - `handoff-kickoff/SKILL.md:54` hardcodes memory dir `…/projects/-Users-benhudelson/memory/`; the correct runtime-resolved dir for a project is `~/.claude/projects/<project-hash>/memory/` (as `resume/SKILL.md:45` already does). The two skills disagree. → make handoff-kickoff resolve at runtime.
   - `roadmap-phase-kickoff/SKILL.md:31` hardcodes `/Users/benhudelson/projects/coachme`. → becomes "repo root" once relocated, or read from config.
   - *Next move:* cheap, isolated edits; do first if Ben greenlights any change.

2. **Relocate the 3 single-repo skills into their repos** (recommended; not authorized):
   - `pursue` → job-consultant repo `.claude/skills/`; `publish-spec` + `roadmap-phase-kickoff` → coachme repo `.claude/skills/`.
   - Drop the global copy (no same-named global twin — it would shadow the project one). Decide per repo whether to **commit** (ships to collaborators — good for the CoachME process skills; confirm job-consultant stays private for `pursue`).
   - *Next move:* needs Ben's go-ahead; then stage moves for review.

3. **Externalize the handoff family's per-repo specifics** (recommended; not authorized):
   - Keep `handoff-kickoff` / `worktree-handoff` **global** (precedence makes that correct — same procedure everywhere), push project specifics (epic-ledger rules, ADR dir, Linear team, memory-file names) into a per-repo profile doc the skill reads.

4. **`step-table` / `resume`** — leave global; only cleanup is swapping the CoachME example in `step-table` for a neutral one. Low priority.

## Closed / superseded

- **"Same-named skill in both global + project, project wins inside the repo"** — *not supported.* Global/personal shadows project. Don't reopen; the supported paths are (a) single home in the repo with no global twin, or (b) distinct names per repo.

## Out of scope for next session

- Any skill-file edits until Ben authorizes (he's kept this advisory).
- Building a tracker/ATS adapter abstraction (strategy E) — only if a second tracker/ATS ever appears.

## Context load order

1. This file.
2. The 7 `SKILL.md` files under `/Users/benhudelson/.claude/skills/*/`.
3. Global `~/.claude/CLAUDE.md` (the only binding rules — no repo-local CLAUDE.md).
4. Claude Code skills precedence docs: https://code.claude.com/docs/en/skills.md (if revisiting the relocation question).

## Verification debt

- The precedence claim ("personal overrides project") came from the `claude-code-guide` subagent citing the official docs URL above; **not independently re-fetched** by the lead agent. Treat as high-confidence but re-read the docs page before acting on a relocation if precision matters.
- Collaborator invite to **behudelson** is sent but **acceptance is unconfirmed**.

## Standing instructions for the next agent

> **Always update or remove memory the same turn an item changes state.** When any tracked item changes state (a commit lands, an epic transitions PM-Kanban stage, a decision is made, a blocker is resolved, an ADR is filed, an item is dropped or deferred), update the relevant memory file (most often `project_pending_decisions.md`) inline in the same turn. **When an item fully closes, *delete* the entry — do not leave "done" or "resolved" markers; closed entries accumulate and obscure what's actually open.** Cite the commit SHA / Linear ID / ADR slug for the state change in your conversation reply, not in the memory entry. If the change invalidates this kickoff prompt's "what landed" or "open work" sections, flag it to Ben rather than silently editing the kickoff — kickoffs are historical artifacts; memory is the live view.

## START HERE

The skills repo exists, is pushed, and a collaborator is invited. The substantive open thread is **skills management**: there's a delivered analysis, two known defects, and a recommended relocation/profile-doc plan — **all advisory; nothing is authorized to change yet.**

Ask Ben which he wants to move on, if any:
1. Fix the two defects only (cheap, isolated).
2. Relocate the 3 single-repo skills into their repos (+ decide commit-vs-local per repo).
3. Externalize the handoff family into per-repo profile docs.
4. Something else / leave as analysis.

Wait for Ben's answer. Do not start work until he picks a topic.
