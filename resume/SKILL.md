---
name: resume
description: Load the handoff that matches the current branch — the trunk kickoff (handoff-kickoff) on the default branch, or the branch's local worktree handoff (worktree-handoff) on a feature branch — plus its referenced memory files, summarize current state, and wait for direction. Use ONLY when the user explicitly types `/resume` — do NOT fire on ambient phrases like "let's continue", "resume work", or "pick up where we left off". This is opt-in by design; ambient invocation defeats the user's choice to be in the directory without engaging project state.
---

# Resume

Restores session context by reading the most recent kickoff written by `handoff-kickoff`, plus the memory files it points at. Does not start work — orients the agent and waits for direction.

## When to invoke

Only on explicit user invocation (`/resume`). Treat any other phrasing — "let's continue," "resume work," "pick up where we left off," "what's the state," etc. — as ambient conversation, not as a trigger for this skill. The user has chosen the slash command as the deliberate signal that they want full project context loaded.

## Step 0 — Resolve which handoff applies (branch-aware)

0. **If not in a git repo** (git commands error): skip branch detection entirely and read the trunk fallback path (`docs/kickoff-prompts/next-session-kickoff.md`, or the `CLAUDE.md`-directed path). There are no worktree handoffs without git.
1. **Current branch** — `git branch --show-current`. **Default branch** — `git symbolic-ref --short refs/remotes/origin/HEAD` (strip `origin/`), fallback `main`.
2. **Trunk kickoff path** — as before: grep root `CLAUDE.md` for `handoff-path:` / `handoff-file:`; compose `<handoff-path>/<handoff-file>`; fallback `docs/kickoff-prompts/next-session-kickoff.md`.
3. **Worktree handoff path** — `<git-common-dir>/claude/worktree-handoffs/<branch-slug>.md` (`git rev-parse --git-common-dir`; branch with `/` and unsafe chars → `-`).
4. **Pick the primary:**
   - On the **default branch** → the trunk kickoff is primary; there is no worktree handoff.
   - On a **feature branch** → the worktree handoff is primary **if it exists**; otherwise fall back to the trunk kickoff and say so explicitly ("no worktree handoff for `<branch>` — loaded trunk").

## Step 1 — Read the primary handoff

- **Trunk kickoff** (single canonical file, overwritten each regen — prior snapshots live in git history): on the default branch read from the working tree; from any other branch read via `git show <default-branch>:<handoff-path>/<handoff-file>` — never check out the default branch, never copy the file into the current branch's tree (per root `CLAUDE.md § Handoff path`).
- **Worktree handoff**: read directly from the resolved local common-dir path. It is never tracked in git, so there is no `git show` form — just read the file.
- Read the resolved file **end-to-end**. The load-bearing sections are: Anchor (commits, branch), In-flight, Out of scope, and START HERE.

## Step 1.5 — Index other handoffs and flag stale ones (read-only)

- **Index** — list the other `*.md` files in the worktree-handoffs dir plus the trunk kickoff, and print a one-line awareness index (e.g. `Other handoffs: trunk @ <date>; worktree: feat-auth, feat-billing`). This is what makes many concurrent worktrees livable — you see sibling sessions without loading them.
- **Flag stale** — for each worktree handoff file, derive its branch from the filename and check `git rev-parse --verify <branch>`. If the branch no longer exists (merged/deleted), mark it `[stale]` in the index so the user knows to clean it up. **Report only — this skill never deletes.**

## Step 2 — Follow pointers into memory files

The kickoff emits pointers (per root `CLAUDE.md § Handoff narrative routing`) to memory files holding verification debt and parked items. Read each pointer's target. The standard pair is:

- `project_verification_debt.md`
- `project_pending_decisions.md`

If the kickoff names additional memory files, read those too. Memory files live in the project's auto-resolved memory directory (`~/.claude/projects/<project-hash>/memory/`).

## Step 3 — Read other pointers only as needed

The kickoff may point at ADRs, the polish backlog, the provisioning ledger, specs, or NFR docs. Do **not** pre-fetch all of them — that defeats the context-efficiency rationale for this skill. Read selectively here only if a pointer is essential to summarize current state in Step 4. Deeper reads happen after the user picks a direction in Step 5.

## Step 4 — Summarize for the user

Report in a tight block (table preferred when fields are parallel):

- **Anchor** — latest commit SHA on the relevant branch + branch state (ahead/behind origin if non-trivial)
- **In-flight** — one line from the kickoff's In-flight section (or "Nothing" if empty)
- **Next move** — one line distilled from START HERE
- **Open carry-forward** — count of items in verification debt + pending decisions (counts, not contents)

Keep the summary under ~15 lines. The user reads the kickoff themselves when they need the full picture.

## Step 5 — Wait for direction

End with a one-line ready signal (e.g., "Ready when you are."). Do **not** start work until the user picks a direction. This is non-negotiable — `/resume` orients, it does not act.

## Anti-patterns

- **Do not pre-fetch every pointer** in the kickoff. Read selectively.
- **Do not start work** after Step 4. The user picks direction in Step 5.
- **Do not modify** the kickoff or memory files. This skill is read-only.
- **Do not fire on ambient keywords.** Only on explicit `/resume` invocation.
- **Do not check out the default branch** to read the trunk kickoff from a feature branch — use `git show <default>:<path>`. Worktree handoffs are read straight from disk (never in git).
- **Do not delete anything.** Stale worktree handoffs are flagged in Step 1.5, never removed — this skill is fully read-only.
- **Do not summarize the contents** of verification debt or pending decisions in Step 4 — just count them. The user reads the files themselves if needed.
