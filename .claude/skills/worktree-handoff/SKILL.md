---
name: worktree-handoff
description: Branch-scoped, local-ephemeral handoff for a single feature/worktree in flight. Use when the user says "wrap this worktree", "hand off this branch/feature", "park this worktree", "prep a fresh agent for this feature", or "context is rotting on this branch". NEVER commits or pushes — it writes local scratch under the git common dir so concurrent worktrees never contend. Companion to handoff-kickoff (the trunk/project handoff); /resume auto-loads whichever matches the current branch. If invoked on the default branch, defer to handoff-kickoff.
---

# Worktree Handoff

Branch-scoped sibling of `handoff-kickoff`. Where `handoff-kickoff` captures the **whole project** (committed + pushed to the trunk), this captures **one feature branch in flight** as local, ephemeral scratch so a fresh agent on the same branch resumes cleanly. It **never commits and never pushes** — that is the defining difference, and it's what lets many worktrees wrap concurrently without contending for a ref.

This skill specifies only its *deltas* from `handoff-kickoff`. For the shared mechanics — the continuous-learning sweep, verify-before-strip, the section schema, the length guard, the standing-instruction block — follow the like-named sections of `handoff-kickoff` so the two never drift.

**Before Step 0: Read `~/.claude/skills/handoff-kickoff/SKILL.md`.** Every "run X verbatim" / "same as Step N" reference below resolves against that file's actual text — never paraphrase the shared mechanics from memory of its description.

## When to invoke

- "wrap this worktree / branch / feature", "park this worktree", "hand this feature to a fresh agent", "context rotting on this branch".
- Mid-session if the lead agent's context degrades while on a feature branch.
- **If invoked on the default branch** (no feature branch checked out): stop and tell the user to use `handoff-kickoff` instead — trunk work belongs in the trunk handoff.

## Step 0 — Confirm branch context and resolve the local path

1. Current branch: `git branch --show-current`. Empty / detached HEAD → abort with a clear message (nothing to scope to); point at `handoff-kickoff`.
2. Default branch: `git symbolic-ref --short refs/remotes/origin/HEAD` (strip the `origin/` prefix); fall back to `main`. If current branch == default → stop and defer to `handoff-kickoff`.
3. Common dir: `git rev-parse --git-common-dir`, resolved to an absolute path. This is shared by every linked worktree of the repo.
4. Handoff file: `<common-dir>/claude/worktree-handoffs/<branch-slug>.md`, where `<branch-slug>` is the branch with `/` and any other path-unsafe characters replaced by `-`. Create the directory if missing.

> **Why under the common dir.** It's reachable from every worktree, survives `git worktree remove`, and is never tracked (it lives inside `.git/`, so no commit and no gitignore needed). `gc`/`prune`/`clean` don't touch a custom subdir there.

## Step 0.5 — Continuous-learning sweep

Run `handoff-kickoff`'s **Step 0.5 — Continuous-learning sweep** verbatim (corrections / operational procedures / preferences → durable mechanisms, one `AskUserQuestion` per item). It's global and orthogonal to which handoff you're writing — do it before the rest.

## Step 1 — Collect the branch handoff brief

Same as `handoff-kickoff` **Step 1**, but scoped strictly to this branch: in-flight decisions on this feature, verification debt from this branch's subagents, failed approaches on this branch, feature-specific open questions, and parked items (waiting on a human, the trunk, or another branch). Keep under ~30 lines. If the project uses epic ledgers, also run `handoff-kickoff` **Step 1.5**.

## Step 2 — Verify-before-strip against the prior branch handoff

If the resolved file exists, read it and apply `handoff-kickoff` **Step 2** verification + Applied/Abandoned/Waiting classification — but anchor all deltas to **this branch** (commits since the prior handoff's anchor; merge-base with the default branch). Carry unverifiable items forward tagged `[unverified]`.

## Step 3 — Assemble the handoff file

Use `handoff-kickoff` **Step 3**'s section schema, scoped to the branch, with these deltas:

- **Anchor** names the *branch*, its latest SHA, and its merge-base with the default branch.
- Add a **Relationship to trunk** line: what (if anything) must graft into the trunk handoff when this branch merges.
- Include the required **standing-instruction memory block** (identical to `handoff-kickoff`).

## Step 4 — Length guard

Same soft cap (~200 lines); if exceeded, split a digest into the same common-dir folder and link it.

## Step 5 — Write and report — NO commit, NO push

- Write the file to the resolved local path. **Never `git add`, never commit, never push.** This is the core distinction from `handoff-kickoff`.
- Report: (a) path written, (b) line count, (c) branch + anchor SHA, (d) anything `[unverified]`, (e) digest split y/n, (f) items flagged for trunk promotion, (g) an explicit note that it was **not** committed (so its absence from git isn't a surprise).

## Promotion to trunk

Learnings already reach global memory via the Step 0.5 sweep. Open work that outlives the branch is flagged in Step 5 as promotion candidates; the actual graft into the trunk handoff happens next time `handoff-kickoff` runs on the trunk. If the user says the branch is merging now, offer to delete this worktree handoff after promotion.

## Anti-patterns

- **Don't commit or push the worktree handoff** — local-ephemeral by design.
- **Don't write it inside the working tree** — use the git common dir so it's shared across worktrees and survives `git worktree remove`.
- **Don't run on the default branch** — that's `handoff-kickoff`'s job; defer.
- **Don't duplicate trunk-wide state** — scope strictly to this branch; project-wide items belong in the trunk handoff.
- **Don't skip the Step 0.5 sweep** — same rule as `handoff-kickoff`.
- **Don't re-derive shared mechanics** — reference `handoff-kickoff`'s named sections so the two skills stay in lockstep.
