---
name: pursue
description: When Ben decides to move forward on (pursue / start outreach on) a specific company or job from the search, scaffold and populate a pursuit directory under pursuits/active/outreach/<company>--<role>/ in the job-consultant repo with research.md, outreach.md, and interview-questions.md, following Ben's established conventions. Also handles moving a pursuit between lifecycle stages (outreach -> application -> interviews) and to/from inactive. Use when Ben says "pursue X", "start outreach on X", "let's move forward on X", "set up a pursuit for X", "X looks great, let's go", "move X to application/interviews", "X went cold / passed / rejected", or otherwise signals he wants to chase or re-stage a specific company/role. Do NOT fire on ambient mentions of a company in a digest, or while merely sourcing/ranking. This is the application phase (beyond sourcing-only) and only fires on explicit intent to pursue or re-stage.
---

# pursue

Stand up and track an active job pursuit: research the company, find the right contact, draft outreach in Ben's voice, prep interview questions, and move the pursuit through its lifecycle as it progresses. This flow was done by hand for iFIT, Notable, and Virta; this skill makes it repeatable.

## Directory layout & lifecycle (source of truth = location)

A pursuit's **stage is encoded by where its directory lives** — there is no separate status table to keep in sync. The unit that moves is a single (company, job) pursuit directory.

```
pursuits/
  active/
    outreach/      <- contacting people; pre-application
    application/    <- application submitted, awaiting response
    interviews/     <- in the interview loop
  inactive/         <- not currently moving (parked / passed / rejected / withdrawn / closed)
```

- **Naming (flat `company--role`):** every pursuit dir is `<company-slug>--<role-slug>` (double-dash separator), e.g. `virta--em-product-platform`, `notable--ai-platform-architect`. The role slug is ALWAYS included so multiple jobs at the same company stay distinct (e.g. `acme--staff-eng` and `acme--eng-manager` can sit in different stages at once). When a pursuit is company-level networking with no specific open role, use the target lane as the role slug (e.g. `ifit--ai-eng-leadership`).
- **New pursuits start in `active/outreach/`.**
- **Active stages are exactly three:** `outreach` -> `application` -> `interviews`. There is no separate "offer" stage; an accepted/declined offer moves the pursuit to `inactive/`.
- **Inactive is flat** (no reason sub-folders). The reason (parked / passed / rejected / withdrawn / accepted) lives in the status line at the top of the pursuit's primary file (see below), not in the path.
- **Multiple jobs per company are always separate directories** and may live in different stages or split across active/inactive simultaneously. Never merge them.

## Moving a pursuit between stages

Status changes = a `git mv` of the whole pursuit directory, plus updating the status line in its files.

1. `git mv pursuits/active/<from-stage>/<slug> pursuits/active/<to-stage>/<slug>` (or to/from `pursuits/inactive/<slug>`).
2. Update the **status line** at the top of the pursuit's primary file (a `> STATUS ...` blockquote, e.g. `> STATUS 2026-06-24: moved to application` or `> STATUS 2026-06-24: PASSED — 40% travel`). This is the only place nuance (reason, dates, next step) is recorded.
3. Commit locally with a message naming the transition (e.g. `Virta: move outreach -> application`). Do NOT push.

Do not invent new stage folders. The four locations above (3 active + inactive) are the whole lifecycle.

## When to invoke
- Explicit intent to **start** a pursuit ("pursue X", "X looks great, let's go") -> scaffold under `active/outreach/`.
- Explicit intent to **re-stage** a pursuit ("move X to interviews", "X passed / went cold") -> `git mv` + status-line update.
- NOT on ambient digest mentions, NOT during plain sourcing/ranking.

## Before you start
- Confirm the company AND the specific role (role drives the slug). Company slug = lowercase kebab ("Virta Health" -> `virta`); role slug = lowercase kebab of the title or target lane.
- If a matching pursuit dir already exists (any stage), UPDATE it (don't clobber); add missing files / refresh.
- Read the live constraints first so nothing contradicts them: memory `user-ben-mission-values`, `user-ben-writing-voice`, `project-pursuit-workflow`, and `profile/values.md`.

## Step 1 — Research → `<slug>/research.md`
1. **Pull the role** from the ATS (full description, not a snippet): Ashby `https://api.ashbyhq.com/posting-api/job-board/<token>?includeCompensation=true`, Greenhouse `boards-api.greenhouse.io/v1/boards/<token>/jobs`, Lever `api.lever.co/v0/postings/<token>?mode=json`. Capture comp, and scan the FULL text for travel % and PTO model. (For company-level networking with no open role, write a company brief and note the role situation.)
2. **Financing + trajectory** (web): total raised, last round, valuation, investors, and recent revenue/growth or profitability. The operating story often matters more than the last round.
3. **Employee / engineering feedback** (web): Glassdoor/Blind overall rating and specific engineering pros/cons (culture, infra/tech-debt, management turnover, resourcing, PTO reality). Be honest; surface flags.
4. Write `research.md`: mission, financing, recent performance, leadership, the role + comp, and the eng-feedback caution. Flags included, never buried.

## Step 2 — Find the contact → feeds `outreach.md`
- Target an **engineering-space leader** (Director/Head of Engineering, the likely hiring manager or one-up), NOT a high-profile exec/CEO (less responsive). Correct for stale leadership (departed CTOs etc.) using current sources.
- Prefer a **warm intro / employee referral**. Note LinkedIn connection degree if known (3rd-degree can't be DM'd -> connection-request-with-note, or find a 2nd-degree mutual).
- Find a **genuine hook**: something the contact actually said/wrote (podcast, blog, talk, patent) or a real product/announcement. Tell Ben to verify/consume it before sending.
- **Always link the source URL** for every hook, quote, or claim surfaced about a contact or company (blog post, podcast episode, LinkedIn, funding article). No unsourced content — Ben needs to verify it and may reference it. (Ben correction, 2026-06-24.)

## Step 3 — Draft outreach → `<slug>/outreach.md`
- **Ben's voice** (memory `user-ben-writing-voice`): no em dashes, plain short sentences, understated/honest, low-pressure, sign off "Cheers, Ben". Avoid AI tells ("I'll keep this short", tricolons, hype).
- Produce: a **LinkedIn connection request** (keep under ~200 chars) and a **follow-up** message.
- **Angle = most authentic alignment**: runner/athlete hook for fitness cos; healthcare-AI insider for health cos; shared builder/IP; etc. Lead with what's real, not a template.
- Ben's personal AI training tool: an asset at non-competitors (frame as a personal, non-commercial tool), but keep it OUT at direct competitors (e.g. a fitness-app company) — competitor/flight-risk signal.
- File holds: a `> STATUS ...` line at the top, contacts table, hook(s) + sources, the two drafts, and routing notes.

## Step 4 — Interview questions → `<slug>/interview-questions.md`
- Draft questions that (a) test the specific risks surfaced in research (culture/resourcing/turnover flags), and (b) confirm fit (is it really his AI lane? PTO reality? mandate clarity?).
- **Every question states its rationale.**
- **Every question grounded in research content (a review quote, a funding fact, a JD line) links its direct source URL** (Ben correction, 2026-06-24) — same rule as hooks. No unsourced quotes.

## Step 5 — Surface honest flags to Ben
In your reply, call out dealbreakers/cautions plainly: travel over ~15%, comp below target, culture red flags, fit mismatches (ML/data-eng/deep-infra heaviness). Never oversell; Ben values the straight read.

## Step 6 — Record + commit
- If warranted, log a thumbs-up for the company: `python -m jobsrc.feedback up "<Company>" --title "..." --note "..."` (don't penalize a good company for a bad-fit role).
- Commit the pursuit directory locally (do NOT push; repo holds personal data and is private by choice).

## Anti-patterns
- Do **not** invent new stage folders or a separate status table — the four locations (outreach/application/interviews/inactive) ARE the status.
- Do **not** merge two roles at the same company into one dir — keep each (company, job) distinct.
- Do **not** move a pursuit by copying — use `git mv` so history follows, and update the status line.
- Do **not** target recruiters or high-profile execs/CEOs.
- Do **not** fabricate a hook. If you can't verify it, say so and use a real one.
- Do **not** use em dashes or marketing cadence — match Ben's voice exactly.
- Do **not** send anything. Produce drafts Ben controls.
- Do **not** oversell. Lead with the honest picture, flags first.
- Do **not** surface any hook/quote/claim about a candidate without a **linked source URL**.
- Do **not** fire while merely sourcing/ranking, or on an ambient company mention.
- Do **not** chain independent shell commands with `&&`/`;` — global `CLAUDE.md` rule (parallel Bash blocks; no `cd <path> && ...`). Restate it in the prompt of any subagent this skill spawns.
