---
name: step-table
description: Render content with parallel structure as Ben's preferred box-drawing markdown table — his terminal renders a well-formed markdown table as a ┌─┬─┐ box. Use WHENEVER output has parallel structure: step/procedure & progress updates, results/summary reports, option comparisons, status/hypothesis matrices. Applies in every context and to every agent (subagents and other skills included), not just step procedures. The step-procedure column set is a preferred preset, not a fixed schema — drop columns that don't apply and add ones that do. Not for a single one-off item with no parallel structure.
---

# step-table

Ben thinks in tables, not prose. His terminal renders a well-formed markdown table as a ┌─┬─┐ box, so any time content has parallel structure, present it as ONE markdown table rather than prose or a stacked list. This holds in **every context** — main session, subagents, and other skills that produce output for Ben — not just step procedures.

This skill is the formatting authority for tables. It carries (1) **universal rules** that apply to every table, and (2) **column presets** for common content shapes. The presets are **preferences, not fixed schemas**: include a column only when it carries information for the content at hand, drop the ones that don't apply, and add ones the content needs.

## When a table applies

Mirror the tables-over-prose categories from `CLAUDE.md` — reach for a table whenever you'd otherwise repeat the same lead-in phrase ("Step 1:", "Option 1:", "H1:", "Field:"):

- **Step / procedure & progress** — ordered things to do, things being done, or a mix.
- **Results / summary reports** — path, line count, SHA, status fields, etc.
- **Option comparisons** — one row per option, columns for the dimensions traded off.
- **Status / hypothesis matrices** — one row per item being tested/verified, status per item.

Not for a single one-off item with no parallel structure.

## Universal rules (every table)

- **Never let the table collapse** into a stacked "Field: value" list. That happens only when a cell holds an *unbreakable* token wider than his terminal (~150 cols) — almost always a long command or URL. Keep those out of cells: write `cmd ↓` / `link ↓` in the cell and drop the full text in a fenced code block *beneath* the table. Prose cells may wrap across lines — multi-line rows are fine.
- **No HTML in cells — ever.** `<br>`, `<b>`, etc. render as literal text in his terminal, not as markup. A cell never holds more than ONE sentence or ONE short command; anything multi-part (numbered sub-steps, click-path + command, alternatives) goes *below* the table as a labeled code block or numbered list, referenced from the cell (`steps ↓`, `cmd ↓`).
- **Cell budget: ~12 words / ~80 chars.** A cell that needs more is content that belongs below the table with a `↓` reference. When several rows need long content, give each a bold label below (**Step 2 — …**) so the references stay unambiguous.
- **Choose columns to fit the content.** Carry only columns that inform *this* content. Drop preset columns that don't apply; add columns the content needs. Don't pad with empty cells, and don't drop a column merely to save width — shrink wording or move long content below instead.
- **One canonical table per thing.** When Ben asks a question or reports progress mid-table: answer/acknowledge first, **then re-print the ENTIRE table** with updated cells — never a partial slice, so he never has to scroll back.
- **Propagate to subagents explicitly.** Skill content does not reach subagents on its own: when spawning a subagent whose output will be shown to Ben, paste this "Universal rules" block (and the relevant preset) into its prompt. Declaring that the rules "apply to every agent" does nothing without this.

## Presets

### Step / procedure (preferred default for ordered work)

Columns, in order — keep the ones that apply:

`#` · `Step` · `Action` · `Why` · `Performer` · `Status` · `Expected`

- **`#`** — step number.
- **`Step`** — short title (≤ ~3 words).
- **`Action`** — the copy-pasteable command or exact click-path (long command/URL → `↓` + code block below).
- **`Why`** — one short phrase of rationale. Ben wants to understand *why*, not just *what*.
- **`Performer`** — emoji only: **🏃🏻‍➡️ = Ben (you, in his voice)** · **🤖 = me (Claude)**. Drop the column if every row has the same performer.
- **`Status`** — **✅ done** · **👉 now** · **⬜ later** (a short note after the emoji is fine, e.g. `👉 now (after step 5)`). Drop if it isn't a progress table.
- **`Expected`** — the observable result that confirms the step worked. Drop if trivial.

Put a one-line legend directly above the table, covering only the emoji columns you kept:

`Legend: ✅ done · 👉 now · ⬜ later · 🏃🏻‍➡️ you · 🤖 me`

### Results / summary

`Field | Value` — one row per reported field (path, line count, SHA, status, …).

### Option comparison

One row per option; columns are the dimensions being traded off (cost, complexity, risk, reversibility, …). Don't bury the tradeoffs in prose.

### Status / hypothesis matrix

One row per item under test/verification; a `Status` column (e.g. ✅ confirmed · ❌ falsified · 🔄 testing) plus whatever evidence columns apply.

## Example — step preset, full schema

Legend: ✅ done · 👉 now · ⬜ later · 🏃🏻‍➡️ you · 🤖 me

| # | Step | Action | Why | Performer | Status | Expected |
|---|---|---|---|---|---|---|
| 1 | Set staging vars | 5 vars on `coachME` | backend needs the Auth0 + gate config to mint & reset | 🤖 | ✅ done | vars set |
| 2 | Register iPhone | open reg link ↓ on the iPhone, install profile | Apple only installs on phones registered first | 🏃🏻‍➡️ | 👉 now | phone registered |
| 3 | Build iOS | cmd ↓ | the env var skips the Apple capability sync that errors | 🏃🏻‍➡️ | ⬜ later | install link (~15m) |

**Reg link** (open on the iPhone in Safari):
```
https://expo.dev/register-device/<id>
```
**Build command:**
```
EXPO_NO_CAPABILITY_SYNC=1 npx eas-cli build --profile e2e --platform ios
```

## Example — columns dropped that don't apply

Every step is Ben's and there's no progress tracking, so `Performer`, `Status`, and `Expected` are dropped:

| # | Step | Action | Why |
|---|---|---|---|
| 1 | Pull latest | `git pull` | start from trunk |
| 2 | Install deps | `npm ci` | lockfile-exact deps |
