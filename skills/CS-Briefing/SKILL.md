---
name: cs-exec-briefing
description: Generate and deliver weekly CS executive briefings from Staircase AI relationship signals, with week-over-week continuity tracking and Slack Canvas delivery. Use this skill whenever the user wants to run, generate, preview, or schedule a CS briefing, executive briefing, leadership digest, or weekly CS update built from customer health/sentiment signals — even if they don't say "briefing" explicitly. Also use it to set up or onboard a new briefing recipient (a CS VP, Director, or exec), and whenever a scheduled task prompt references the cs-exec-briefing skill.
---

# CS Executive Briefing

Generate a narrative weekly briefing for a CS leader: thematic patterns across their org, new risks with verbatim customer quotes, continuity on previously flagged accounts, gone-dark early warnings, expansion signals, and pre-written questions for their managers. Data comes from Staircase AI (via MCP); delivery is Slack Canvas + DM. A per-recipient history file provides week-over-week memory.

This skill is org-agnostic. Every org- and recipient-specific value (names, org tree, email formats, support address, delivery preferences) lives in config files in the working folder — never assume or hard-code them.

## Data files (all in the working folder)

| File | Purpose |
|---|---|
| `./recipients/_org.json` | Org-wide constants: company name, email format, support address, ticketing system |
| `./recipients/{first-last}.json` | One per recipient: org tree, ID map, delivery prefs, lookback window |
| `./history/{first-last}.json` | One per recipient: every past briefing (mutable state — never store config here) |

Schemas for all three are in `references/schemas.md`. The skill's own folder is read-only — data files always live in the working folder.

## Mode routing

Decide the mode first:

1. **Setup** — the user wants to onboard a new recipient, configure the briefing, or has no config file yet ("set up a briefing for Dana", "get this running for my VP"). Read `references/setup-mode.md` and follow it. Setup is a guided, step-at-a-time walkthrough — assume no prior Cowork expertise.
2. **Run (scheduled)** — the invoking prompt identifies itself as a scheduled run (e.g., "scheduled run" appears in the prompt, or it comes from a scheduled task). Execute the full pipeline below including delivery and history write.
3. **Run (preview)** — any other generation request ("run the briefing for Dana", "what would Monday's briefing look like?"). Execute the pipeline but **stop before delivery**: render the briefing in chat, write nothing to history, send nothing to Slack. Preview is the default because an ad-hoc mid-week run that writes history would double-count entries in continuity tracking, and delivering would ping the exec twice. If the user explicitly says to deliver ("...and send it"), treat it as scheduled.

If the recipient named in a run request has no config file, offer setup instead of guessing.

## Run pipeline

Execute in order. Don't skip steps.

1. **Load config.** Read `./recipients/_org.json` and `./recipients/{first-last}.json`. If either is missing or fails to parse, stop and follow Failure handling below.
2. **Read history.** Read `./history/{first-last}.json`. Load only the most recent `history_lookback_weeks` entries (from config; default 12) — older entries stay on disk but aren't needed for continuity language, and reading them all makes every run slower for no benefit. Extract: accounts flagged in the last briefing and their status, consecutive weeks each unresolved issue has been open, unactioned expansion signals, and gone-dark accounts that have or haven't re-engaged. If the file is missing or `briefings` is empty, this is **Week 1**: no continuity language anywhere, and Section 3 is omitted entirely.
3. **Query Staircase.** Pull current signals at the account level for every CSM in the recipient's org tree. For each at-risk account, fetch a verbatim customer quote via `staircase_fetch_evidence`. Then attribute owners and classify CSM presence — read `references/presence-detection.md` for the attribution rules (Owner field, aliases, numeric ID map, direct ICs) and the presence classifications. Never let a raw numeric ID reach the output.
4. **Draft the briefing.** Read `references/briefing-format.md` and follow it exactly — section order, status emoji criteria, brevity rules, quote selection, and the one-consolidated-email-per-manager rule. Write in an executive narrative voice: a well-informed colleague briefing the exec, not a system report.
5. **Deliver** (scheduled runs only). Per `delivery.method` in config:
   - `slack_canvas_dm` (default): create a Slack Canvas with the full briefing (formatting rules in `references/briefing-format.md`), then DM the recipient: "Your weekly CS briefing is ready — {canvas link}".
   - `slack_dm_text`: send the full briefing as DM text (no Canvas; mailto links won't be clickable — omit the Canvas instruction note).
6. **Write history** (scheduled runs only, and **only after delivery succeeds**). Append this week's entry to the history file per the schema, including the delivery timestamp and Canvas URL. History is written last on purpose: a failed run that already wrote history would make next week's briefing treat undelivered flags as "previously communicated," silently breaking continuity.

## Failure handling (scheduled runs)

The run is unattended, so degrade rather than stop, and never leave the recipient wondering:

- **Partial Staircase data** (some queries fail, some succeed): generate the briefing from what succeeded and state the gaps explicitly in the relevant sections ("Signals for {pod} were unavailable this run").
- **Canvas creation fails**: fall back to sending the full briefing as Slack DM text.
- **Total failure** (Staircase unreachable, config/history unreadable, nothing deliverable): do not write history. DM the recipient a short plain-language notice that this week's briefing failed to generate, **including the actual error** — e.g., "Your CS briefing couldn't be generated this week. Error: Staircase AI connector did not respond." Don't say it's "being looked into" — no one is automatically notified, so the honest error is what lets them act on it.
- In every failure case: no history write unless a real briefing was successfully delivered.

## References

| File | Read it when |
|---|---|
| `references/setup-mode.md` | Running setup / onboarding a recipient |
| `references/presence-detection.md` | Step 3 — owner attribution and presence classification |
| `references/briefing-format.md` | Step 4 — drafting; Step 5 — Canvas formatting |
| `references/schemas.md` | Creating or validating any config/history file |
| `references/validation-checklist.md` | Reviewing a dry run during setup |
| `references/troubleshooting.md` | Numeric/duplicate owner IDs, M365 questions, scheduling reliability |
