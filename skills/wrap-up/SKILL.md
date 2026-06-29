---
name: weekly-wrap-up
description: Produce Rhiannon's personal weekly status across all systems. Pull from Google Calendar, Slack, meeting transcripts (both processed recaps and raw transcript bodies), and the Jira GAIN board. Output three sections in order — completed this week (top), in-progress (middle), brief planned next week (bottom). Bulleted high-level summaries, not raw activity dumps. Trigger ONLY on "weekly wrap up", "weekly wrapup", "wrap up the week", "weekly wrap-up". Do NOT confuse with the separate `weekly-recap` skill (which produces a shareable HTML for stakeholders) — this one is Rhiannon's private status view.
---

# Weekly Wrap-Up — Personal Cross-System Status

When the user says "weekly wrap up", produce a personal status summary in three blocks. This is different from `weekly-recap` (which builds a shareable HTML for the team). This is Rhiannon's own view of her week.

## Step 1: Define the week

Default: Monday of this week through Friday of this week. If today is mid-week, the "this week" frame still covers Mon-Fri of the current week; in-progress items reflect current state.

Compute and state: `Week of YYYY-MM-DD to YYYY-MM-DD`.

## Step 2: Pull from all four sources in parallel

Same four sources as the `todos` skill. Single message, parallel calls:

1. **Calendar** — `list_events` from Monday of this week through next Friday (+5 days past today). Capture both meetings attended and focus blocks used.

2. **Slack** — `slack_search_public_and_private` with these queries:
   - `from:me after:<Monday>` — what Rhiannon shipped, posted, or replied to this week
   - `to:me after:<Monday>` — what was sent to her (for in-progress context)
   - Priority channels: `#proj-csops`, `#gainsight-gurus`, `#cs-lab-digital-scale`, `#rev-ops-csm`, `#prog-enterprise-apps-rollout`, `#customer-success-all`
   - Look for shipped deliverables: shared docs, posted recordings, dashboard links, confirmed decisions

3. **Meeting transcripts** — two layers, same as the `todos` skill:
   - **Recaps (primary):** Grep `projects/transcripts/processed/**/*.recap.md` for recaps dated within the week. Recaps drive most of the wrap-up content because they already contain decisions and action items.
   - **Raw transcripts (secondary):** Grep `projects/transcripts/processed/**/*.md` (excluding `*.recap.md`) and `projects/transcripts/inbox/**/*.md` when:
     - A recap is sparse and the underlying discussion shows real progress that belongs in "Completed" or "In progress"
     - You need to verify a status claim (e.g. someone said something shipped — confirm in the transcript)
     - A new transcript landed in the inbox this week and hasn't been recapped yet (treat its body as evidence; flag that it needs `process-transcripts` at the end)
   - Use the most recent `projects/todos/YYYY-MM-DD.md` for the open-items baseline.

4. **Jira GAIN board** — `searchJiraIssuesUsingJql`:
   - Completed: `project = GAIN AND assignee = currentUser() AND status changed to Done after "<Monday>"`
   - In progress: `project = GAIN AND assignee = currentUser() AND statusCategory = "In Progress"`
   - Created next week's planned: `project = GAIN AND assignee = currentUser() AND statusCategory = "To Do"`

## Step 3: Classify each item

For every action item, decision, or commitment surfaced, classify into one of:

- **Completed** — evidence shows the deliverable shipped, the question got answered, the meeting happened with outcome, or the ticket moved to Done
- **In progress** — work has begun (focus block used, Slack reply started, partial deliverable visible, ticket in progress) but not finished
- **Planned next week** — calendar event or Jira ticket dated next Mon-Fri, or a Slack commitment with a date in next week

If something doesn't fit any bucket (vague, no evidence either way), don't include it.

## Step 4: Format the output

**Audience:** this output goes to Rhiannon's boss (the CCO). It is an executive status update, not a calendar dump or activity log. Tone should match the CCO-voice memory: professional, high-level, bullets only, no day-by-day minutiae.

**Default output format: standalone HTML report** built via the `html-report` skill conventions. Markdown is the fallback only when the user explicitly asks for "just text" or "markdown only". The HTML structure is locked in based on the 2026-05-15 reference build at `projects/weekly-wrap-ups/2026-05-15-weekly-wrap-up.html` — match that layout:

1. **Header block** — eyebrow ("CS Ops · Weekly Status"), h1 ("Week of <Mon date> – <Fri date>, YYYY"), subtitle ("Rhiannon Wigglesworth · Reporting to CCO"), issued-date line
2. **Q2 (or current quarter) Scorecard** — card with all active major workstreams (from `project_active_workstreams.md`), each row showing: project name, status pill (On Track / At Risk / Data Blocked / In Design / Done), progress bar, % complete. Right-aligned quarter-end date as anchor. Ask the user for % complete before the build — never guess.
3. **This week — what shipped** — 2x2 grid of win cards (4 cards is the target; 3 or 5 acceptable), each with bold title + one-sentence summary. One-line ad-hoc summary below the grid covering questions answered / fields edited / data validated / admin resolved.
4. **Forward look — next 2 weeks by project** — one section per active workstream from memory, outcome-framed bullets. Omit a project bucket entirely if nothing is planned (don't pad).
5. **Blockers & dependencies** — boxed signal row with cross-functional dependencies, data issues, and team-wait items.
6. **Footer** — provenance line citing data sources (Calendar, Slack channels, transcripts, Jira) and the week dates.

Design follows the `html-report` skill warm-neutrals palette (`--bg #fafaf7`, `--accent #5b3a29`, etc), 760px max-width, system font stack, no emoji, no AI watermarks. Save to `projects/weekly-wrap-ups/YYYY-MM-DD-weekly-wrap-up.html`.

**Email variant:** when the user asks for an email-formatted version (or to draft the wrap-up in Gmail), produce a second file at `projects/weekly-wrap-ups/YYYY-MM-DD-weekly-wrap-up-email.html` using email-safe HTML (table-based layout, all inline styles, no `<style>` block, web-safe fonts, 600px width). Visual design matches the standalone report. When asked to draft in Gmail, use the Gmail `create_draft` tool with `htmlBody` set to the email-formatted content and `to` set to Tyler McNally (`tyler.mcnally@tempo.io`, Rhiannon's CCO) unless another recipient is specified.

**Markdown fallback (rare):** only used if the user explicitly asks for "just text" or "markdown only". Same three logical groupings as the HTML — completed work (with mandatory Ad-hoc bundle), in-progress workstreams, forward look by project — but rendered as plain markdown sections.

### Rules for the Completed / "What Shipped" section

- **Only include builds, key developments, and high-impact meetings.** Skip routine 1:1s, recurring syncs without decisions, and small admin tasks.
- **Always include an "Ad-hoc work" item** as the last entry — it is non-optional. Bundle every small completion (single-question Slack replies, field tweaks, validation pulls, access requests, single-line confirmations) into this entry.
- The Ad-hoc entry uses a structured format with these bullet types (omit any with no content for the week):
  - `Answered <X> questions across <channels>: <topics>`
  - `Edited <field/object/dropdown> to include <change>`
  - `Validated data for: <comma-separated list>`
  - `Resolved <admin/access item>`
- Main entries should have 1-2 sub-bullets max. If a single completed item has more than 2 bullets, it probably belongs in In Progress.
- Imperative titles. No padding.

### Rules for the Q2 Scorecard

- **One row per active major workstream** from `project_active_workstreams.md`. Consolidate related sub-deliverables under a single project row (e.g. renewal participant list + outreach analytics dashboard both feed Digital Renewal Journeys — one row).
- **Status pills** are constrained to: `On Track`, `At Risk`, `Data Blocked`, `In Design`, `Done`. Don't invent new pill labels.
- **Always ask the user for % complete** before the build. Never guess for an exec-facing doc.
- **Cross-check related Jira tickets.** When a completed item this week resolves part or all of an open Jira ticket on the GAIN board, do NOT silently update the ticket. Instead, add an "### Ask before sending" section after the report: name the ticket, summarize what landed, and propose either (a) a comment + transition to Done, or (b) a scope-down to what's left. Wait for explicit user confirmation before making Jira changes. Honor the standing rule in `feedback_jira_ticket_updates.md`.
- **Respect the GAIN board freeze.** Per `project_gain_board_bulk_update.md`, ticket statuses cannot currently be transitioned until Tyler Angyal completes the strategic board configuration. Note this as a blocker in the Blockers card rather than proposing ticket edits that can't yet be executed.

### Rules for the Planned Next Week section

- **Bucket by project, not by day.** The user explicitly does not want a day-by-day calendar view here.
- Active projects to bucket under (this list evolves — verify against current memory and Slack signals; ask user if uncertain):
  - Digital Renewal Journeys
  - Onboarding Team in Gainsight (Success Plans)
  - NPS Survey end-to-end workflow
  - Verified Outcomes / Value Outcomes mapping
- Bullets are outcome-framed ("Build participant list for Time Sheets + Structure annual licenses"), not calendar-framed ("Monday 9:15 focus block").
- Source the bullets from: Jira ticket due dates on GAIN, blocked focus-time on the calendar, action items from this week's transcripts and Slack DMs.
- If a workstream has nothing planned next week, omit its bucket entirely — don't pad.
- Keep this section actionable and brief; 2-4 bullets per project is the target.

## Step 5: After building

After writing the file(s), output the relative path to the user. If an email variant was requested, output both paths. Do not narrate the HTML content — the user views the rendered file directly.

## Rules

- **HTML is the default.** Markdown is only used if the user explicitly asks for "just text".
- **Past tense for shipped, present tense for in-progress, future tense for planned.**
- **Voice rules from anti-slop apply.** No em-dashes in prose, no hedge words, no padded sentences.
- **Pull live data every time.** Never recycle last week's output as a template. Calendar shifts, Slack flows, Jira moves.
- **If a project bucket would be empty, omit it.** Don't pad with "Nothing planned this week".
- **Cross-check completed items against `todos` skill output.** If something shows here as shipped and also shows on the to-do list, the to-do list is wrong — flag it at the end so memory stays consistent.
