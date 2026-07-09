---
name: handoff-kickoff
description: Generate a next-session kickoff prompt that hands off in-flight work to a fresh agent without context rot. Use when the user says things like "wrap this session", "write a kickoff prompt", "hand this off to the next agent", "context is getting full / rotting", "let's prep for a fresh agent", or "update the kickoff". Verifies completion before stripping items, carries forward parked + verification-debt items, and preserves project invariants. On launch it first runs a continuous-learning sweep — surfacing corrections, operational procedures, and preferences from the session and durably storing the ones Ben approves — before the handoff routine.
---

# Handoff Kickoff

Produces a dated kickoff prompt at the project's configured handoff path so the next agent can pick up cleanly. The point is preventing context rot, so robustness beats speed — verify claims before treating them as done.

## When to invoke

- User signals: "wrap this session", "write/update the kickoff prompt", "hand off to the next agent", "context is rotting", "prep a fresh agent".
- Mid-session if the lead agent notices its own context is degrading and a handoff is the right next move.
- If invoked while on a non-default branch, note that a **worktree handoff** (`worktree-handoff`) may be the better fit for branch-scoped work, and confirm with the user before generating a trunk kickoff.

## Step 0 — Resolve the output path and filename (project-local, configurable)

The skill is global, but kickoff files belong with the repo (they reference commit SHAs and local paths).

**Path** — resolve in this order; stop at the first match:

1. **Explicit override in `.claude/handoff-config`** (single line: `path: <relative-path>`).
2. **`CLAUDE.md` directive** — grep root `CLAUDE.md` for `handoff-path:` and use its value.
3. **Existing convention** — if any of these dirs already contain `*kickoff*.md` files, reuse: `docs/kickoff-prompts/`, `docs/handoffs/`, `.claude/handoffs/`, `docs/transcript-reports/` (legacy — projects predating a dedicated kickoff dir).
4. **Default** — `docs/kickoff-prompts/`. Create the directory if missing.

**Filename** — single canonical file, overwritten each session-boundary regen. No dated filenames; no `-vN` versioning. Git history is the audit trail for prior snapshots. Resolve in this order:

1. **`CLAUDE.md` directive** — grep root `CLAUDE.md` for `handoff-file:` and use its value.
2. **Default** — `next-session-kickoff.md`.

If the resolved file already exists, overwrite it — the prior content is preserved in git history (the new commit's parent SHA is the prior kickoff). Never `--amend` an existing kickoff commit.

## Step 0.5 — Continuous-learning sweep (run before the rest of the routine)

This is the first interactive step every time the skill launches. Its purpose is the continuous-learning loop: surface the lessons of this session and make them *durable* before they evaporate at the handoff boundary. Do this **before** Step 1 and everything after it.

### What to review

Review the transcript **since the last handoff**. The lower bound is the prior kickoff's `Anchor` (date / commit SHA from Step 0's resolved file); the practical scope is the current session's conversation in context. If no prior kickoff exists, review the whole session.

### What to extract — three categories

Scan for moments where Ben shaped how you work, and classify each into exactly one:

| Category | What it looks like | Durable home (default mechanism) |
| --- | --- | --- |
| **Correction** | Ben fixed a wrong output, assumption, or behavior ("no, not like that", "you got X wrong", a redo) | A `feedback` memory capturing the rule + **why** + **how to apply**, so it isn't repeated. If the fix is an *automated* behavior ("whenever X, do Y"), a hook via the `update-config` skill. |
| **Operational procedure** | A process instruction meant to hold going forward ("from now on…", "always do it this way", "the procedure is…") | If it's an automated/"every time X" behavior → a hook in `settings.json` (via `update-config` — the harness, not Claude, must enforce these). If it's a judgment rule → a `CLAUDE.md` directive or `feedback` memory. |
| **Preference** | A stated like/dislike about format, tone, tooling, or defaults | A `user` or `feedback` memory + `MEMORY.md` pointer. Promote to `CLAUDE.md` only if it should bind every project unconditionally. |

### De-duplicate first

Before presenting anything, check whether each candidate is **already captured** — scan existing memory files in the session's auto-resolved memory directory (`~/.claude/projects/<project-hash>/memory/` — use the exact path the runtime injects in the system prompt's Memory section, same as `resume` does) plus its `MEMORY.md`, root `CLAUDE.md`, and `settings.json` hooks. Drop items already durably stored. The point is to ask only about genuinely new learnings, not to re-litigate settled ones.

### Present each item one-by-one for decision

For **each** surviving item, make a separate `AskUserQuestion` call (one item per question — do not batch, even when the answer is effectively yes/no). The question must:

- **Restate the item** in one line so Ben recognizes it (quote his words where it helps).
- Offer **concrete durable mechanisms** as the options, most-recommended first with `(Recommended)` in the label. Tailor options to the category using the table above — typically some subset of: write/update a memory file, add a `CLAUDE.md` directive, configure a hook via `update-config`, edit a relevant skill. Each option's `description` should state exactly what will be written/changed and where.
- The list always implicitly includes "Other" (free text) and the user may decline — if none fit, treat the answer as skip.

Keep going until every surviving item has been presented and answered.

### Implement, then proceed

Apply each chosen mechanism **immediately**, in this same step:

- **Memory** — write the file with correct frontmatter (`feedback`/`user`/`project`/`reference`), and for `feedback`/`project` include the `**Why:**` / `**How to apply:**` lines. Link related memories with `[[name]]`. Update an existing file instead of duplicating when one already covers the topic. Then regenerate `MEMORY.md` per *Concurrency-safe writes* below — do **not** hand-append the pointer line.
- **Hook / setting** — invoke the `update-config` skill to configure `settings.json` (automated "whenever X" behaviors can only be enforced by a hook, not by memory). Writes to `settings.json` must be atomic (see below).
- **CLAUDE.md directive** — add a tight, unambiguous line under the right section, written atomically (see below).
- **Skill edit** — modify the relevant skill file when the lesson is about how a skill itself should behave.

### Concurrency-safe writes (sessions in other worktrees share global memory)

Other sessions may run this sweep at the same time against the same global files. Two guards keep that safe:

1. **Atomic writes for `settings.json` and root `CLAUDE.md`.** Never edit them in place from the sweep. Compose the full new content, write it to a temp file in the same directory, then `mv` it over the target (atomic rename) so a concurrent reader/writer can never observe a half-written file — this prevents the one catastrophic outcome (malformed JSON breaking the harness). Re-read the file immediately before composing to shrink the lost-update window; prefer routing `settings.json` through `update-config`. Atomic rename prevents *corruption*, not lost *updates* — the residual lost-update risk is accepted given its rarity.
2. **Regenerate `MEMORY.md`; never hand-append.** After writing/updating memory files, rebuild `MEMORY.md` by scanning the frontmatter (`name` + `description`) of every `*.md` in the memory dir, preserving any manual header, and write it via the same temp+rename. Because the index is *derived from the files on disk*, a lost concurrent write self-heals: the orphaned memory file is still present and the next regen re-includes it. This makes the common race (a dropped index line) non-destructive.

Report a one-line confirmation per item (mechanism + path/target). Then continue to Step 1. If the sweep surfaces nothing new, say so in one line and proceed.

## Step 1 — Collect the current agent's handoff brief

Before spawning the synthesis subagent, the lead agent (you, in the current session) writes a structured brief from working memory. The fresh agent cannot reconstruct this from git or Linear. Cover:

- **In-flight decisions** — what was Ben mid-discussion on when the session ended? Quote the last unresolved exchange if useful.
- **Verification debt** — claims a subagent reported as done that were *not* independently checked (file writes, Linear updates, eval edits). The next agent must verify before relying on them.
- **Failed approaches** — paths attempted that didn't work, so the next agent doesn't redo them.
- **Open questions raised but not filed** — items that came up mid-session but aren't yet ADRs / Linear issues / spec edits.
- **Parked items waiting on a human** — anything blocked on Matt, Ben, or external input that needs to keep surfacing.
- **Uncommitted / untracked working-tree state** — run `git status --porcelain` and list the modified/untracked paths the anchor SHA does *not* capture, so the next agent knows the working tree carries unsnapshotted work. (Resume is in the same worktree, so the changes are physically present — the brief just has to point at them.)

Keep the brief tight (under ~30 lines). Pass it to Step 2 as input.

## Step 1.5 — Sweep epic ledgers (final γ-rule safety net)

Projects that use a per-epic ledger system (look for `docs/epic-ledgers/` with a `README.md` defining the format) have a γ rule: tagging `IN` per CLAUDE.md § Scope discipline commits to a same-turn ledger write. The α PreCompact hook is the in-session safety net. **The handoff-kickoff sweep is the final safety net.**

Before spawning the synthesis subagent:

1. Identify ledgers touched in the current session by what the conversation actually discussed (Linear epic IDs surfaced, ADRs cited, transcripts dispositioned). Do not rely on the file mtimes — IN material that was *not* written yet has no mtime trace.
2. For each affected epic, read the existing ledger (or recognize it doesn't exist yet — lazy creation is the steady state).
3. For every IN-tagged recommendation from the session that has not already been written, write it now: Zone 2 History entry + Zone 1 update if the IN material changes the present-state narrative + bump `last_updated` in frontmatter. Use supersession language ("supersedes 2026-05-NN paragraph 2") when refining earlier entries.
4. If a ledger does not exist for an epic with new IN material, create it from the format in `docs/epic-ledgers/README.md`.
5. Significant unflushed IN material at this step is a γ-rule violation worth surfacing in the kickoff's "Failed approaches" or "In-flight" sections — the next agent needs to know enforcement was loose so they tighten it.

Skip this step entirely if `docs/epic-ledgers/` does not exist in the project — not every project uses this system.

## Step 2 — Spawn synthesis subagent (agentic, not mechanical)

Delegate the actual kickoff write to a `general-purpose` Agent, passing `model: opus` on the Agent call — verify-before-strip and the Applied/Abandoned/Waiting classification are judgment-heavy, and handoffs often close out sessions running a smaller coding model. The subagent's job:

**Read:**
- The most recent prior kickoff file in the resolved output dir (anchor commit, prior open work).
- Root `CLAUDE.md` (invariants — operating rules, attribution ban, ADR cadence, spec-driven gate).
- The handoff brief from Step 1.

**Verify before stripping** — for each item in the prior kickoff's "open work" / "what landed" sections, check:
- *Code/docs claims* — `git log <prior-anchor>..HEAD -- <path>` confirms the change exists. File existence via `Read` for new ADRs or specs.
- *Linear claims* — `mcp__linear__get_issue` confirms status / comment posted. Carry forward anything unverifiable with a `[unverified]` tag.
- *Eval claims* — open the eval file and confirm the new IDs / counts match.

**Classify each prior-kickoff item into one of three states:**
- *Applied* — verified completed. Strip from "open work" but capture in the new "what landed" section.
- *Abandoned or changed* — decision was rolled back or reframed. Keep a one-line note in a `## Closed / superseded` section so the next agent doesn't reopen it.
- *Waiting on human* — still blocked. Carry forward verbatim with the blocker named.

**Pull session deltas** since the prior kickoff's commit anchor:
- New ADRs in `docs/decisions/` (filename + one-line gist).
- New/modified specs and evals.
- Linear status changes on referenced issues (PRO-/ENG-).
- New commits with subject lines.

## Step 3 — Assemble the kickoff file

Sections in this order:

1. **Anchor** — prior session date(s) + commit SHA(s) the new agent should treat as ground truth.
2. **Working directory + branch + today's date** placeholder (use `(use the date the runtime injects)` for date if uncertain).
3. **Hard operating rules** — pulled from root `CLAUDE.md`, not copy-forwarded from the prior kickoff (avoids stale-rule drift). Include: spec-driven gate, attribution ban, ADR cadence, drift-check pre-commit, delegation pattern if present.
4. **What landed this session** — verified deltas only. Group by ADR / spec / eval / Linear / commit.
5. **In-flight at handoff** — direct from the Step 1 brief. This is the part the next agent uses to resume mid-decision.
6. **Open work (carry-forward, classified)** — buckets of remaining work. Each bucket: title, one-line description, recommended next move, dependency notes. Mark `[unverified]` items distinctly.
7. **Closed / superseded** — one-liners on abandoned or reframed decisions so they aren't reopened.
8. **Out of scope for next session** — explicit not-yet items.
9. **Context load order** — refresh to point at the *newest* canonical files, not stale paths from the prior kickoff.
10. **START HERE** — regenerate the opener fresh; it must reflect the *current* board state, not last session's framing. End with: "Wait for Ben's answer. Do not start work until he picks a topic." (or project-equivalent direction if CLAUDE.md specifies).

### Required standing instruction (every kickoff)

Every generated kickoff MUST include — verbatim, in a top-level section labelled `## Standing instructions for the next agent` placed immediately before `## START HERE` — the following block:

> **Always update or remove memory the same turn an item changes state.** When any tracked item changes state (a commit lands, an epic transitions PM-Kanban stage, a decision is made, a blocker is resolved, an ADR is filed, an item is dropped or deferred), update the relevant memory file (most often `project_pending_decisions.md`) inline in the same turn. **When an item fully closes, *delete* the entry — do not leave "done" or "resolved" markers; closed entries accumulate and obscure what's actually open.** Cite the commit SHA / Linear ID / ADR slug for the state change in your conversation reply, not in the memory entry. If the change invalidates this kickoff prompt's "what landed" or "open work" sections, flag it to Ben rather than silently editing the kickoff — kickoffs are historical artifacts; memory is the live view.

This is non-negotiable. The reason is `feedback_memory_state_updates.md`: stale memory leaks into next-session kickoffs and produces wrong premises (surfaced 2026-05-06 after a v2 kickoff described already-committed items as uncommitted).

## Step 4 — Length / rot guard

Soft cap ~200 lines. If the assembled file exceeds it, the synthesis subagent should split: keep the kickoff lean and move detailed deltas to a separate `YYYY-MM-DD-handoff-digest.md` in the same dir, then link from the kickoff. Warn the user if the cap is hit and the digest split happened.

## Step 4.5 — Prune stale worktree handoffs (default branch only)

Only when this kickoff is generated on the default branch (the trunk authority). Housekeeping for branch-scoped scratch left by `worktree-handoff`.

1. List `<git-common-dir>/claude/worktree-handoffs/*.md` (`git rev-parse --git-common-dir`). Skip the step if the dir is absent.
2. For each file, derive its branch from the filename and run `git rev-parse --verify <branch>`.
3. **Delete** the file if its branch no longer exists — merged-then-deleted or deleted-without-merge both land here; the scratch is dead.
4. **Report, don't delete**, any file whose branch still exists but whose mtime is old (>30 days): it may be an abandoned branch the user would still want to resume, so flag it and let them decide.
5. Report the prune result: deleted count + filenames, and any old-but-live handoffs flagged.

## Step 5 — Write, commit, push, report

- Write the kickoff file.
- **Commit when applicable.** From the output dir, check `git rev-parse --is-inside-work-tree`. If inside a git working tree and the kickoff path is not gitignored:
  - Stage the new kickoff **plus any other untracked `*kickoff*.md` files in the same dir** — kickoffs belong with the audit trail, and leaving prior ones untracked recreates the pain this step exists to solve.
  - Do **not** stage anything else. The kickoff commit is its own clean atom; never bundle in-flight work, ADR edits, or other session changes — those get their own commits owned by the user.
  - Commit with a subject matching the project's commit convention (infer from `git log -10 --oneline`; reasonable default `kickoff: handoff <YYYY-MM-DD>`). Body optional; a one-liner is fine because the kickoff content speaks for itself.
  - **Always push the kickoff commit to `main`.** Run `git push origin HEAD:main` so the commit lands on remote `main` regardless of the branch currently checked out (no branch switch, no stash). This is a normal fast-forward push — **never** force (`--force`/`--force-with-lease`). If the push is rejected as non-fast-forward (main diverged), do **not** force past it: report `pushed: failed (main diverged, not forced)` and leave the local commit in place for the user to reconcile.
  - Skip the commit with a recorded reason if any precondition fails (not a git repo, path gitignored, pre-commit hook fails, staging produced an empty diff). Never use `--no-verify` to force past a failing hook. If the commit is skipped, there is nothing to push.
- Report back to the user with: (a) path written, (b) line count, (c) commit SHA or `not committed: <reason>`, (d) push result (`pushed to main @ <sha>` or `pushed: failed (<reason>)`), (e) any prior untracked kickoffs swept into the same commit (count + filenames), (f) anything flagged `[unverified]`, (g) whether a digest split happened, (h) any prior-kickoff items the synthesis subagent could not classify confidently.

## Anti-patterns

- **Don't trust the TodoList** — completion claims must be verified against repo / Linear state.
- **Don't copy hard operating rules from the prior kickoff** — pull from CLAUDE.md so rule changes propagate.
- **Don't preserve historical kickoffs in-tree** — the single canonical file is overwritten each regen; git history is the audit trail. Never reintroduce dated or `-vN` filenames.
- **Don't drop parked items** because they were also parked last session — that's exactly the kind of thing context rot loses. Carry-forward indefinitely until the blocker is named as resolved.
- **Don't drop verification debt** — if Step 1's brief flags something unverified and Step 2's subagent can't verify it either, carry it forward tagged.
- **Don't write a verbose narrative.** The next agent needs a checklist + pointers, not a story. Aim for dense, scannable, with file paths and IDs.
- **Don't skip the Step 0.5 learning sweep** — it runs every launch, before the handoff brief. Skipping it lets session lessons evaporate, which is the whole problem it exists to prevent.
- **Don't batch the learning-sweep items** — one `AskUserQuestion` per item, even for yes/no decisions. Ben decides each mechanism explicitly.
- **Don't re-ask already-stored learnings** — de-dupe against existing memory, `CLAUDE.md`, and hooks first, so the sweep only surfaces genuinely new items.
- **Don't claim a correction is "stored" without a durable mechanism** — a memory, hook, directive, or skill edit must actually be written. An automated "whenever X" behavior requires a hook (via `update-config`); memory alone can't enforce it.
