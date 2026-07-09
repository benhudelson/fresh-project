---
name: roadmap-phase-kickoff
description: Kicks off a CoachME roadmap phase by resolving the referenced heading in `docs/roadmap.md`, creating the project-convention branch `phase-<n>-<kebab-slug>`, eliciting the feature spec via the AskUserQuestion tool (waiting for the user's answers before any file is written), and scaffolding `specs/YYYY-MM-DD-<feature-slug>/` with `requirements.md`, `plan.md`, `validation.md`, and `evals/` per `specs/CLAUDE.md`. Use when the user (a) references a roadmap phase heading via `@sym:` (e.g., `@sym:## Phase 3 — Mobile app scaffold`) paired with intent to begin work, or (b) says any of: "start the next phase", "kick off phase X", "kick off the next phase", "begin Phase N", "open the next phase branch", "scaffold the spec for Phase X", or an equivalent phrasing. Do NOT trigger on ambient mentions of phase numbers without explicit intent to begin. Does not auto-commit or push. Aborts loudly if the phase reference cannot be resolved, the branch already exists, or the target spec directory already exists.
---

# Roadmap phase kickoff

Scaffolds a feature spec for a CoachME roadmap phase: reads the phase from `docs/roadmap.md`, opens the phase branch per the project convention, elicits the spec content from Ben via AskUserQuestion (no inference), and writes the four-artifact spec directory required by `specs/CLAUDE.md`. The skill is the bridge from a roadmap line to a spec ready for the spec-driven pipeline (`/publish-spec` downstream).

Canonical references:
- Roadmap workflow: `docs/decisions/2026-05-11-roadmap-workflow.md`.
- Phase-branch workflow: `docs/decisions/2026-05-16-phase-branch-workflow.md`.
- Spec directory shape (hard): `specs/CLAUDE.md`.
- NFR gate (hard): root `CLAUDE.md` § Context discipline → § Spec-driven pipeline; canonical NFR doc `docs/architecture/nfrs.md`.

## One-time migration prompt (self-removing — delete this section once resolved)

This skill is CoachME-specific and is slated to move into the repo. Before Step 0, check whether this file still lives at the personal/global location: `test -e ~/.claude/skills/roadmap-phase-kickoff/SKILL.md`. If it does NOT exist there, this section is stale — ignore it. If it does, surface a one-time migration to Ben via AskUserQuestion **before** starting the kickoff:

- **Proposal:** move this skill — and offer `publish-spec` in the same pass (also CoachME-only, at `~/.claude/skills/publish-spec/`) — into `<repo-root>/.claude/skills/<name>/SKILL.md`.
- **Critical:** the personal copies must be **deleted in the same move** — personal skills shadow same-named project skills, so a leftover global copy makes the relocated one unreachable.
- **Options:** migrate both (recommended) · migrate this skill only · skip this session (ask again next time) · never (delete this section from the global copy and proceed).
- **If Ben approves:** `mkdir -p .claude/skills/<name>` in the repo, copy each SKILL.md in **with this section removed**, delete the corresponding `~/.claude/skills/<name>/` directory, and note that both the skills repo (`~/.claude/skills` is a git repo — deletion needs a commit) and the coachme working tree are now dirty. Ben owns both commits (in coachme, decide then whether the skill ships to collaborators). Then continue the phase kickoff normally.
- This migration is the sole exception to the "no file writes before Step 4" invariant below.

## When to invoke

- The user references a roadmap heading via `@sym:## Phase N — <Title>` (any heading level) with explicit intent to begin work on it.
- The user says "start the next phase", "kick off Phase N", "begin Phase N", "open the next phase branch", "scaffold the spec for Phase N", or equivalent.
- Do **not** invoke on ambient mentions of phase numbers ("Phase 3 will need…", "we talked about Phase 2 earlier") without an action verb signalling intent to begin.

## Invariants

- **No file writes before Step 4 completes.** Branch creation in Step 3 is the only mutation allowed before the user's answers are in hand.
- **No auto-commit. No auto-push.** Ben owns the first commit on the new branch.
- **No inference of spec content.** Every section of every artifact is sourced from the user's answers in Step 4. If a section has no answer, write the section heading with the placeholder `*(awaiting user)*` and surface it in the Step 6 report — do not invent prose.
- **Fail loudly.** Phase reference unresolvable, branch already exists, or target spec dir already exists → abort with a clear error before any further work. Do not auto-pick a different name.

## Step 0 — Preflight

1. Confirm cwd is the CoachME repo root — the repo containing both `docs/roadmap.md` and `specs/CLAUDE.md`. Bail if not (do not guess a path; ask Ben to run the skill from the repo).
2. Confirm the working tree is clean OR that pending changes are unrelated and safe to carry onto a new branch. If a phase branch checkout would clobber uncommitted edits, abort with a clear message.
3. Resolve **today's date** via `date +%Y-%m-%d`. Use it verbatim for the spec dir name — do not derive from system context strings.

## Step 1 — Resolve the phase reference

The user's reference is one of:
- A symbol citation like `@sym:## Phase 3 — Mobile app scaffold` (any heading level).
- A bare phrase like "Phase 3" or "the next phase" tied to intent.

Resolve it against `docs/roadmap.md`:

1. Grep `docs/roadmap.md` for `^#+ Phase ` headings.
2. Match the user's reference by phase number when given (e.g., "Phase 3"), or by title fragment if number isn't given.
3. "The next phase" = the lowest-numbered phase whose exit criteria are not yet delivered. If that's ambiguous, ask AskUserQuestion to confirm before continuing.

**Fail loudly** if no heading matches the reference, or if multiple headings match and the user hasn't disambiguated. Output the candidate matches (or "no matches") and stop. Do not guess.

Capture the resolved heading exactly as it appears in the file (e.g., `Phase 3 — Mobile app scaffold`).

## Step 2 — Derive names and pre-check existence

From the resolved heading `Phase <N> — <Title>`:

- **`phase_number`** = `<N>` (the integer after "Phase ").
- **`title_slug`** = `<Title>` kebab-cased: lowercase, ASCII, replace non-alphanumeric runs with `-`, strip leading/trailing `-`. Em-dashes, en-dashes, slashes, ampersands → `-`.
- **`branch_name`** = `phase-<N>-<title_slug>` (matches root `CLAUDE.md` § Branch and PR workflow).
- **`feature_slug`** = `<title_slug>` (no `phase-N-` prefix — `specs/CLAUDE.md` example: `2026-05-14-backend-provisioning`, not `2026-05-14-phase-2-...`).
- **`spec_dir`** = `specs/<YYYY-MM-DD>-<feature_slug>/` using today's date from Step 0.

Pre-check, **before** creating anything:

1. Branch must not exist locally: `git rev-parse --verify --quiet refs/heads/<branch_name>` returns non-zero.
2. Branch must not exist on origin: `git rev-parse --verify --quiet refs/remotes/origin/<branch_name>` returns non-zero.
3. Spec dir must not exist: `test ! -e <spec_dir>`.

If any check fails, **abort** with a clear message naming the conflicting artifact. Do not auto-suffix (`-v2`, `-redo`, etc.) — surface to the user and stop.

## Step 3 — Create the branch

If Step 2 passes:

```
git checkout -b <branch_name>
```

No push, no upstream tracking. The branch is local until Ben commits and pushes himself.

If the checkout fails (e.g., the working tree has incompatible local edits surfaced in Step 0), abort and surface the error verbatim. Do not force.

## Step 4 — Elicit the spec via AskUserQuestion

Use the AskUserQuestion tool to collect the answers needed to populate the four artifacts. **No file writes happen in this step.** Wait for the user to answer every question before proceeding.

AskUserQuestion supports 1–4 questions per call with 2–4 structured options each plus an automatic "Other" free-form path. For each free-form area below, either offer reasonable starting options + "Other (specify)" or surface the area as a chat prompt requesting prose from the user. The skill is satisfied either way; the load-bearing rule is *every answer must come from the user, not inferred*.

Required coverage — every artifact section below must be answered before Step 5 runs. Group questions sensibly to minimize round-trips; aim for ≤4 AskUserQuestion calls plus at most one free-form chat prompt for prose dumps.

**For `requirements.md`:**
- **Identity** — what is this feature; which phase it closes; whether it bundles multiple ADRs/decisions.
- **Decision context** — which ADRs and canonical docs this spec *operationalizes* (pointer list, not restatement).
- **In scope** — numbered bullet list of behaviors/components shipped by this feature.
- **Out of scope** — bullet list of related concerns explicitly deferred and where they go.
- **User-observable outcome / "done" definition** — numbered list of observable behaviors that hold when the spec is done.
- **Shape choices recorded in this spec** (optional) — local design choices that aren't new ADRs.

**For `plan.md`:**
- **Task groups** — for each: heading title, one-paragraph intro, checklist of sub-tasks, "Done when" line, "Depends on" line. Each group will ship as one Linear ENG-N Issue.
- **Group count** — ask explicitly; do not infer from scope size.

**For `validation.md`:**
- **Behavioral acceptance criteria** — one row per `done` outcome from `requirements.md`, mapping to its programmatic eval(s) and any manual operator step.
- **Applicable NFR rows** — for each, the row pointer into `docs/architecture/nfrs.md`, how it is verified for this spec, and the artifact path that captures the verification. Ask Ben to enumerate; do not infer the NFR list.
- **Skipped NFR rows + reasoning** — explicit list, with one-sentence reasoning per row.
- **Operational artifacts** — list of `docs/operations/<file>.md` artifacts that must exist at sign-off.
- **Sign-off conditions** — numbered list.

**For `evals/`:**
- **Eval ID prefix** — short stable token unique to this spec (e.g., `PROV`, `ONBD`). Ask Ben; do not invent.
- **Eval inventory** — for each eval: short title, what it specifies (one sentence), preconditions, inputs, expected outputs, notes. One eval per programmatic acceptance criterion from `validation.md` is the default; allow Ben to merge/split.

If Ben declines to populate a section ("leave it blank for now", "I'll fill it in later"), write the heading + `*(awaiting user)*` placeholder in Step 5 and call it out in the Step 6 report. Do **not** invent prose to fill the gap.

## Step 5 — Write the spec directory

Only after Step 4 has captured every answer (or explicit `*(awaiting user)*` placeholders):

1. `mkdir -p <spec_dir>/evals`.
2. Write `<spec_dir>/requirements.md`. Section shape mirrors `specs/2026-05-14-backend-provisioning/requirements.md`:
   - `# <Feature title> — requirements`
   - `## Identity`
   - `## Decision context`
   - `## In scope`
   - `## Out of scope`
   - `## User-observable outcome ("done" definition)`
   - `## Shape choices recorded in this spec` (only if Ben provided any; omit otherwise).
3. Write `<spec_dir>/plan.md`. Section shape mirrors `specs/2026-05-14-backend-provisioning/plan.md`:
   - `# <Feature title> — plan`
   - One paragraph intro pointing at `requirements.md` and noting that ENG-N annotations are written by `/publish-spec` (additive-only).
   - For each task group: `## <N>. <Title>`, then a blank line, then `**Linear:** *(pending publish)*`, then the intro paragraph, then a checklist (`- [ ]`), then `**Done when:** …`, then `**Depends on:** …`. Separator `---` between groups.
4. Write `<spec_dir>/validation.md`. Section shape mirrors `specs/2026-05-14-backend-provisioning/validation.md`:
   - `# <Feature title> — validation`
   - `## Behavioral acceptance criteria` — table with columns `# | Acceptance criterion | Programmatic coverage | Manual / operator verification`.
   - `## NFR enumeration` with `### Applicable NFR rows` (table: `NFR pointer | How verified for this spec | Artifact`) and `### Skipped NFR rows (with reasoning)` (table: `NFR section / row | Why skipped here`). Per root `CLAUDE.md`, the NFR section is mandatory and pointer-only (no restatement of `docs/architecture/nfrs.md`).
   - `## Operational artifacts` — bulleted list of `docs/operations/<file>.md` paths.
   - `## Sign-off conditions` — numbered list.
5. Write `<spec_dir>/evals/README.md`. Section shape mirrors `specs/2026-05-14-backend-provisioning/evals/README.md`:
   - Heading + one-paragraph intro.
   - `## Eval file shape` — the standard section list (Heading / What this specifies / Preconditions / Inputs / Expected outputs / Notes).
   - `## Runner` — note that the runner is operational tooling, deferred until first eval-author session.
   - `## Manual-vs-automated boundary` — note which evals require manual verification.
   - `## Index` — one bullet per eval file (`EVAL-<PREFIX>-<seq>-<slug>.md` — short title).
6. For each eval Ben enumerated in Step 4, write `<spec_dir>/evals/EVAL-<PREFIX>-<seq>-<short-slug>.md` with the six standard sections (`# EVAL-<PREFIX>-<seq>: <title>` / `## What this specifies` / `## Preconditions` / `## Inputs` / `## Expected outputs` / `## Notes`).

**Do not commit.** Leave the working tree dirty with the new files staged or unstaged per default `git checkout -b` behavior. Ben owns the first commit.

## Step 6 — Report

Output a tight table (per root `CLAUDE.md`):

| Field | Value |
| --- | --- |
| Phase resolved | `Phase <N> — <Title>` |
| Branch created | `<branch_name>` (local only; not pushed) |
| Spec directory | `<spec_dir>` |
| Files written | count + bulleted list |
| Sections left as `*(awaiting user)*` | count + bulleted list of `path § section` |
| Next move | `git add <spec_dir> && git commit` (Ben), then `/publish-spec` once the spec is review-clean |

If any section is `*(awaiting user)*`, flag it as a blocker before `/publish-spec` can run — the publish skill aborts on incomplete specs.

End with a one-line confirmation that nothing was committed or pushed.

## Failure modes (all abort loudly, before any further work)

| Trigger | Detail |
| --- | --- |
| Phase reference unresolvable in `docs/roadmap.md` | Output the candidate matches (or "no matches"); stop. No branch created, no files written. |
| Phase number/title disambiguation ambiguous | Output candidates; ask via AskUserQuestion; do not guess. |
| Branch already exists (local or `origin/`) | Output the existing ref; stop. Do not auto-suffix. |
| Spec directory already exists | Output the existing path; stop. Do not auto-suffix or merge into it. |
| Working tree has uncommitted edits incompatible with `git checkout -b` | Output `git status`; ask Ben before proceeding. |
| User declines to answer required Step 4 questions | Write `*(awaiting user)*` placeholders **only if** Ben explicitly chooses that path; if Ben simply does not answer, **wait** — do not write inferred prose. |

## Anti-patterns

- **Do not invent spec content.** Every section is sourced from Ben's answers. Inferred prose is the failure mode this skill exists to prevent.
- **Do not auto-commit or push.** Ben owns the first commit and the first push of the phase branch.
- **Do not auto-suffix names** on collision. The user's phrase resolves to one branch and one spec directory; if either exists, that is a signal to stop and clarify.
- **Do not pre-populate NFR rows** by reading `docs/architecture/nfrs.md` and choosing rows yourself. The NFR row list is Ben's call — surface the row pool in the question if helpful, but the *choice* is his.
- **Do not skip the AskUserQuestion step** even when Ben has hinted at content in prior conversation. The skill's contract is "elicit via the question system" — honor that.
- **Do not write a single mega-file or condensed three-file variant.** The four-artifact structure (`requirements.md` + `plan.md` + `validation.md` + `evals/`) is a hard invariant of `specs/CLAUDE.md` and the downstream `/publish-spec` skill depends on it.
- **Do not commit the kickoff to a phase branch.** Phase-independent housekeeping (including this skill's own future edits) lands on `main` per `docs/decisions/2026-05-16-phase-branch-workflow.md`.
