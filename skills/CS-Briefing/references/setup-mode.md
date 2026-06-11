# Setup Mode — Configuring the User's Briefing

Setup configures the briefing **for the current user**. Never ask who the briefing is for — the recipient is the person running setup. The person may have never configured a Cowork skill before, so use plain language throughout (no JSON talk unless they ask to see the files) and confirm before writing files.

Setup is **discovery-first**: pull everything the connected MCPs can tell you *before* asking anything, then present what was found in grouped confirmation passes. The user's job is to confirm and correct, not to dictate from memory. Only ask open questions for things the MCPs genuinely can't know.

## Step 0 — Intro and prerequisites

**Open with a one-or-two-sentence orientation** before doing anything else, e.g.:

> I'll set up your weekly CS briefing. I'll pull most of your information automatically from Staircase and Slack, then ask you a few questions to confirm details and fill in what I can't detect.

Then verify:

- **Staircase AI MCP connector** is connected (try a lightweight query). If not, tell the user how to connect it in Cowork settings and pause until it works.
- **Slack MCP connector** is connected.
- A **working folder** is selected, so `./recipients/` and `./history/` can persist between sessions. If only the temporary scratchpad is available, warn that config and history will not survive the session and help them select a folder first.

## Step 1 — Discovery pass (run before asking anything)

Gather silently; tell the user only that you're "pulling your info from Slack and Staircase — one moment." Collect:

1. **User profile (Slack)**: look up the current user's own Slack profile (their email is available from the session context). Capture: full name, title, timezone, Slack email.
2. **Org constants**:
   - **Company name** — from the Slack workspace name; fall back to the email domain.
   - **Email format** — infer the pattern from observed internal addresses (the user's own, plus Slack profiles / Staircase thread participants on the same domain). Note any addresses that don't fit the pattern as candidate exceptions.
   - **Support address** — scan Staircase thread participants for support@-style addresses on the org's domain.
3. **Org structure (Staircase)**: query the distinct account **Owners** across the org. If Staircase exposes user/team endpoints, pull those too. While collecting, automatically flag:
   - **Numeric IDs** appearing as owners (resolve with the user in Step 3).
   - **Alias candidates** — the same email appearing under two different names.
   - **Candidate groupings** — any team/pod structure visible in the data (shared segments, team fields), to propose rather than build by interview.

Skip Step 2's org-constants confirmation entirely if `./recipients/_org.json` already exists.

## Step 2 — Confirm discovered basics (one grouped pass)

Present **one summary** of what was discovered — recipient basics (name, title, timezone, Slack email) and, on first-ever setup, org constants (company name, email format and exceptions, support address) — and ask the user to confirm or correct. Don't re-ask anything that was discovered; only ask directly for fields discovery couldn't fill. The **ticketing system** (e.g., Salesforce, Zendesk) usually can't be detected — ask for it here.

Write `./recipients/_org.json` per `schemas.md` (first-ever setup only) and confirm.

## Step 3 — Org structure (Staircase-proposed, user-corrected)

Don't make the user dictate their org tree from memory. Using the Step 1 discovery:

1. Present the discovered Owner list, **already grouped into proposed pods** wherever the data supports it. Ask the user to correct the grouping, name each pod's manager, and confirm each pod's type (`csm`, `renewals`, `sales`, `support`, `partner_success`).
2. Ask about teams that don't own accounts (Partner Success/Training, Support) — their presence on threads still gets classified, so they matter even if absent from the Owner list.
3. Ask about **Individual Contributors** who report directly to the user with no pod manager.
4. Ask about **exclusions** — internal/ops roles with no customer-facing accounts that should never be queried or reported on.
5. Resolve any **numeric IDs** flagged during discovery and record them in the ID map. Catching these at setup beats discovering them in a live Monday briefing.
6. Present any **alias candidates** flagged during discovery (same email, two names) and ask about other recent name changes (marriage, etc.) — record aliases so attribution doesn't split one person into two.

If the Staircase queries failed, fall back to a pure interview: have the user list pods, managers, and members themselves, and warn that name spellings must match Staircase owner names exactly.

## Step 4 — Preferences

One grouped pass, with sensible defaults pre-filled:

1. **Delivery method**: Slack Canvas + DM (recommended — clickable mailto links work in Canvas) or plain DM text.
2. **Cadence**: which day, what time, in the timezone discovered from their Slack profile. Suggest the morning before their weekly manager sync — the briefing is built to feed that meeting. Default suggestion: Monday 8:00 AM local time.
3. **History lookback**: explain that the briefing remembers past weeks to say things like "flagged three weeks running," and that by default it looks back **12 weeks**. Ask if they'd like a different window, then record it as `history_lookback_weeks`.

## Step 5 — Write files

Create `./recipients/{first-last}.json` per `schemas.md`, and initialize `./history/{first-last}.json` with the recipient block and an empty `briefings` array. Show the user a short plain-language summary of what was configured (not raw JSON) and confirm.

## Step 6 — Mandatory dry run

Run the full pipeline in **preview