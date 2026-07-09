---
name: publish-spec
description: Publish a finalized feature spec to Linear — reference the phase Initiative (manual prereq), create the feature Project, and an ENG-N Issue per `plan.md` task group, then write the assigned IDs back into `plan.md`. Use when the user says "publish the spec", "push <feature> to Linear", "create ENG issues for <spec>", "run /publish-spec", or any equivalent. Additive-only: never modifies or deletes existing ENG-N annotations.
---

# Publish spec

Reads a `specs/YYYY-MM-DD-<slug>/` directory, creates the corresponding Linear hierarchy via the Linear MCP, and writes the assigned ENG-N IDs back into `plan.md`. The spec directory is the source of truth for *what*; Linear is the source of truth for *status*. Canonical workflow: `docs/decisions/2026-05-11-roadmap-workflow.md` as amended by `docs/decisions/2026-05-15-phase-initiative-mapping.md` (Phase → Initiative, manual prereq). Spec directory shape: `specs/CLAUDE.md`.

## When to invoke

- The user says "publish the spec", "push <feature> to Linear", "create ENG issues for <spec>", "run /publish-spec", or equivalent.
- A finalized `requirements.md`, `plan.md`, `validation.md`, and `evals/` all exist in the spec directory. The NFR gate has been passed (every applicable NFR row in `validation.md` § NFR enumeration has a verification path). Specs that aren't review-clean should not be published — once an Issue lands in Linear, the published task group becomes immutable per the additive-only rule.

## Idempotency contract

- Re-running on a spec where some groups already have ENG-N annotations: those groups are skipped (no Linear writes, no plan.md edits). Only groups with the `**Linear:** *(pending publish)*` placeholder are processed.
- Re-running on a spec where the Linear Project does not yet exist: create it.
- Re-running on a spec where the Linear Project exists: reuse it; never create a duplicate.
- The phase Initiative is a manual prereq — created in the Linear UI before first invocation in a phase. Skill aborts cleanly with a clear message if the Initiative is missing.
- Never modifies or deletes an existing ENG-N annotation or its corresponding Linear Issue. If a task group needs to be removed or reshaped after publish, do it via a new spec — never by editing an already-published group.

## Step 0 — Locate the spec directory

If the user named a path: use it. Otherwise ask. Spec path must be `specs/YYYY-MM-DD-<slug>/` with the four required artifacts (`requirements.md`, `plan.md`, `validation.md`, `evals/`). Abort with a clear message if any artifact is missing — publish does not run on an incomplete spec.

## Step 1 — Identify the phase

Read `requirements.md`. Look for an explicit phase reference (e.g., "Phase 2", "closes every Phase 2 exit criterion in `docs/roadmap.md`"). Propose that phase to the user and confirm before continuing. If no unambiguous phase reference exists, ask the user which phase.

Read `docs/roadmap.md` to recover the phase's full name (e.g., "Phase 2 — Backend infrastructure"). That string is the Initiative name.

## Step 2 — Linear preflight (read-only)

Use the Linear MCP to inspect existing state. No writes in this step. Tool prefix is `mcp__claude_ai_Linear__` in the current environment; adjust if the MCP server is registered under a different name.

1. **Team.** `list_teams`. If exactly one team has key `ENG` (or "Engineering"), use it. Otherwise ask the user. Capture the team ID.
2. **Initiative.** `list_initiatives` and check for an Initiative named `<phase-name>`. If present, note it as confirmed. If missing, surface in Step 3 that the Initiative is a manual prereq (per `docs/decisions/2026-05-15-phase-initiative-mapping.md`) and must be created in the Linear UI before Step 4 can reference it.
3. **Project.** `list_projects` filtered by team. Match by Project name (see Step 3 for the naming rule). If found → "reuse" with ID; else → "create."
4. **Existing issues under the Project.** If the Project already exists, `list_issues` filtered by that Project. Cross-reference each Issue's title against `plan.md` task group headings (heading minus `## N. `). For every title match, mark the corresponding group as "already published" — leave alone in Steps 5 and 6.

## Step 3 — Propose the publish plan

Present to the user, **before any Linear writes**:

- **Spec:** `<path>`.
- **Team:** `<team-key>`.
- **Initiative:** `<phase-name>` — must exist in Linear UI (manual prereq per `docs/decisions/2026-05-15-phase-initiative-mapping.md`); Step 2 `list_initiatives` result noted (confirmed | missing).
- **Project name:** the descriptive form of the spec slug, title-cased (e.g., `2026-05-14-backend-provisioning` → "Backend provisioning"). Allow the user to override.
- **Project description:** one-line pointer (`Spec: specs/<spec-dir>/`). No requirements restatement.
- **Issues to create:** one row per task group with a `*(pending publish)*` placeholder, showing the proposed Issue title and the group's `**Done when:**` line.
- **Issues to skip:** any task groups already annotated with an ENG-N or matched against an existing Linear Issue in the Project.

Wait for explicit user confirmation, including confirmation that the Initiative exists in the Linear UI. Do not proceed on tepid signals.

## Step 4 — Create the Project (if needed)

`save_project` with:
- Name as confirmed in Step 3.
- Description: `Spec: specs/<spec-dir>/`.
- `addInitiatives: ["<phase-name>"]`. If the Initiative does not exist, this call fails — abort the skill with a clear message instructing the user to create the Initiative in Linear UI and re-invoke.
- Team ID from Step 2.

Capture the project ID.

## Step 5 — Create the Issues

For each task group with `**Linear:** *(pending publish)*` and not already matched in Step 2.4:

`save_issue` with:
- **Title:** the group heading minus `## N. ` (e.g., "Initialize `apps/backend/` project").
- **Description** (markdown, pointer-style — no requirements restatement):
  ```
  Spec: specs/<spec-dir>/
  Plan: specs/<spec-dir>/plan.md § N. <Heading>

  <one-line group intro paragraph copied verbatim from plan.md>

  ## Checklist
  - [ ] <bullet 1>
  - [ ] <bullet 2>
  ...

  ## Done when
  <plan.md "Done when" line>

  ## Depends on
  <plan.md "Depends on" line>
  ```
- **Project ID** from Step 4.
- **Team ID** from Step 2.

Capture each returned `(ENG-N, Title)` pair for Step 6.

## Step 6 — Write ENG-N annotations back to `plan.md`

For each newly-created Issue, replace that task group's `**Linear:** *(pending publish)*` line with `**Linear:** ENG-N (Title)` — paired form per root `CLAUDE.md` § Linear references (first-mention pairing).

**Additive-only guard.** Only the literal `**Linear:** *(pending publish)*` placeholder is replaced. If the line under a heading already contains an ENG-N annotation, do not touch it — that group was published in a prior run and is immutable.

Do not commit. The user owns the commit on the `plan.md` change.

## Step 7 — Report

Output to the user:
- Spec path published.
- Initiative: name + (referenced | missing-aborted).
- Project: ID + name + (created | reused).
- Issues created: count + bulleted list of `ENG-N (Title)` pairs.
- Issues skipped: count + reason (already annotated | matched existing Linear Issue).
- `plan.md` lines modified: count.
- Any failures (Linear MCP errors, missing artifacts, write-back conflicts) with their location.

## Anti-patterns

- **Don't write to Linear before user confirmation in Step 3.** Linear writes are not local and not free to undo.
- **Don't overwrite an existing ENG-N annotation in `plan.md`.** Additive-only is non-negotiable — it's how the workflow ADR keeps the repo and Linear in sync without two-way sync.
- **Don't restate `requirements.md` content in any Linear surface.** Project and Issue descriptions are pointers.
- **Don't create a second Project with the same name** under the same team. Always reuse if Step 2 finds one.
- **Don't bundle the `plan.md` write-back with other edits.** The skill writes the file and stops; the user owns the commit.
- **Don't publish a spec that's missing `validation.md` or `evals/`.** The NFR gate and evals pipeline assume publish only happens on review-clean specs.
- **Don't infer the phase silently.** Even when `requirements.md` names the phase clearly, propose-and-confirm before Step 4 (the first Linear write — Create the Project). Silent inference here is the exact failure mode the workflow ADR's "explicit, additive publish" rule exists to prevent.
- **Don't create the Initiative via MCP.** A `save_initiative` tool exists in the current Linear MCP, but per `docs/decisions/2026-05-15-phase-initiative-mapping.md` the Initiative is a deliberate manual prereq — creating it from this skill would bypass that decision. (Revisiting the ADR now that the tool exists is Ben's call, outside this skill.) If Step 4's `save_project` fails on Initiative-not-found, surface that error and stop.
