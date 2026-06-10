---
name: cs-briefing
description: Generates and delivers a weekly CS Executive Briefing using Staircase AI signals and Slack. Say "configure briefing" or "set up briefing for [Name]" to run the interactive setup wizard for a new recipient — it walks through org structure, scheduling, and initializes the history file. Once configured, run on a schedule (weekly, Monday 8AM) to auto-generate and deliver a narrative briefing to a CS leader. Requires Staircase AI MCP and Slack MCP connectors.
---

# CS Executive Briefing

Generates a weekly narrative briefing for a CS leader — not an account dump, but a thematic summary of risks, momentum, expansion signals, and grounded questions to raise with their team. Delivers to Slack Canvas + DM every Monday morning using Staircase AI (signals) + Claude (synthesis).

---

## Mode Detection

Determine which mode to run based on the invocation:

- **Setup mode**: User says "configure briefing", "set up briefing for [Name]", "add briefing recipient", "new cs briefing", or similar → run the Setup Wizard below.
- **Execution mode**: Task fires on schedule with no user message, OR user says "run briefing for [Name]" or "generate briefing for [Name]" → run the Briefing Execution workflow below.
- **Status check**: User says "briefing status", "list briefings", or "briefing recipients" → list all configured recipients and their last delivery date.

---

## SETUP WIZARD

Run this interactively when the user wants to configure the briefing for a new recipient. Walk through each step in order. Checkpoint after every answer by updating the working draft config — so nothing is lost if context fills up.

Do not batch multiple questions together. Ask one question at a time.

### Setup Step 1 — Create working draft

Before asking anything, create a working draft at `./config/briefing-setup-draft.json`:

```json
{ "status": "in_progress", "step": 1 }
```

Tell the user: "I'll walk you through configuring the CS Briefing for a new recipient. I'll save progress as we go — this takes about 5 minutes. Let's start."

---

### Setup Step 2 — Recipient basics

Ask each of the following, one at a time. After each answer, write the captured value to `./config/briefing-setup-draft.json`.

1. **Full name** of the CS leader who will receive the briefing.
2. **Title** (e.g., "VP of Customer Success", "Director of Customer Success").
3. **Slack email** — the email they use to log into Slack. This is used for DM delivery.
4. **Delivery time** — what time on Monday should the briefing arrive? (Default: 8:00 AM. Accept any time in HH:MM format.)
5. **Timezone** — the recipient's local timezone (e.g., `US/Eastern`, `US/Pacific`, `US/Central`, `Europe/London`, `Australia/Sydney`).

---

### Setup Step 3 — Company email format

Ask: "What email format does your company use for internal addresses? For example: `firstname.lastname@company.com`, `firstlast@company.com`, or `f.lastname@company.com`?"

Capture the pattern and domain (e.g., `firstname.lastname@acme.com`). Based on the recipient's name and the pattern, generate their email address and show it to the user for confirmation.

If the auto-generated email is wrong, note the correct address as an exception in the config.

Also ask: "What is your support team's email address? (e.g., `support@yourcompany.com`) I'll use this to identify support-routed threads in Staircase."

Update the draft config.

---

### Setup Step 4 — Org structure

Explain: "Now I'll map the org tree. Tell me about each manager who reports to [Recipient First Name], and who's on their team."

Repeat the following for each manager-level report. After each complete pod is captured, update the draft config.

For each manager, ask:
1. Manager's full name.
2. Team type — which of these best describes this pod?
   - **CSM** — customer-facing, owns accounts in Staircase
   - **Renewals** — manages commercial renewal ownership
   - **Support** — handles support cases (their names won't appear as Staircase owners)
   - **Partner Success / Training** — enablement team, no account ownership in Staircase
   - **Operations** — internal role, excluded from all Staircase queries
3. Team members — name and title for each person on this pod. The user can paste a list or enter one at a time.

After collecting each pod, ask: "Any more manager-level reports? Or are we done with the org structure?"

Then ask: "Does [Recipient First Name] have any Individual Contributors who report directly to them with no manager in between — people who own accounts but sit outside any of the pods above?"

If yes, collect each IC's name and title.

---

### Setup Step 5 — Staircase ID map and name aliases

Ask: "Did your org run a Staircase POC before going live? POC migrations sometimes create duplicate numeric user IDs that appear as account owners instead of names."

If yes: "Do you know any numeric IDs and who they resolve to? Tell me each one as: [number] → [CSM Name] ([Manager]). You can always add more later."

Collect all known mappings.

Then ask: "Are there any CSMs who've recently changed their name — for example due to marriage — who might appear under different names in different systems?"

If yes, collect: former name, current name, and their email address.

Update the draft config.

---

### Setup Step 6 — Review and confirm

Present a full summary of everything captured:

```
Here's the configuration I've built for [Full Name]:

Recipient: [Full Name], [Title]
Slack email: [email]
Delivery: Every Monday at [time] [timezone]

Org structure:
  [Pod Name] — Manager: [Manager Name] (type: [type], [N] members)
  [Pod Name] — Manager: [Manager Name] (type: [type], [N] members)
  ...
  Direct ICs: [list or "none"]

Email format: [pattern]
Support email: [address]
ID map: [N entries / "none"]
Name aliases: [N entries / "none"]

Does this look right? (Say "yes" to finalize, or tell me what to correct.)
```

Make any corrections the user asks for, then confirm again.

---

### Setup Step 7 — Create files

Once confirmed:

**1. Write the final config to `./config/[first-name]-[last-name].json`:**

```json
{
  "recipient": {
    "full_name": "[Full Name]",
    "first_name": "[First Name]",
    "last_name": "[Last Name]",
    "title": "[Title]",
    "slack_email": "[email]",
    "delivery": {
      "day": "Monday",
      "time": "[HH:MM]",
      "timezone": "[timezone]"
    }
  },
  "email_format": {
    "pattern": "[pattern]",
    "domain": "[domain]",
    "exceptions": []
  },
  "support_email": "[support@company.com]",
  "org": {
    "pods": [
      {
        "name": "[Pod Name]",
        "type": "csm | renewals | support | partner_success | operations",
        "manager": {
          "name": "[Manager Name]",
          "email": "[auto-generated from email format]"
        },
        "members": [
          { "name": "[Name]", "title": "[Title]" }
        ]
      }
    ],
    "direct_ics": [
      { "name": "[Name]", "title": "[Title]" }
    ]
  },
  "staircase": {
    "id_map": [
      { "id": "[numeric ID]", "resolves_to": "[CSM Name]", "manager": "[Manager Name]" }
    ]
  },
  "name_aliases": [
    { "former_name": "[Former Name]", "current_name": "[Current Name]", "email": "[email]", "manager": "[Manager Name]" }
  ]
}
```

**2. Create an empty history file at `./history/[first-name]-[last-name].json`:**

```json
{
  "recipient": {
    "name": "[Full Name]",
    "slack_email": "[email]",
    "title": "[Title]",
    "managers": ["[Manager Name]"],
    "direct_ics": []
  },
  "briefings": []
}
```

**3. Delete the working draft:** `./config/briefing-setup-draft.json`

**4. Output the scheduling instructions:**

```
✅ Configuration complete for [Full Name].

Files created:
  ./config/[first-name]-[last-name].json  — configuration
  ./history/[first-name]-[last-name].json — history (empty, Week 1 ready)

To schedule in Cowork:
  1. Open Cowork and create a new Scheduled Task
  2. Frequency: Weekly, every Monday at [time] [timezone]
  3. Paste this as the task prompt:
     
       run cs briefing for [Full Name]
     
  4. Enable "Keep awake" for reliable unattended execution
  5. Pre-approve tool permissions when prompted on first run:
     - Staircase AI MCP — read access
     - Filesystem — read/write to ./history/ and ./config/
     - Slack MCP — Canvas create, DM send

To test before scheduling: say "run briefing for [Full Name]" right now.
```

---

## BRIEFING EXECUTION

Run this on the scheduled task, or when the user says "run briefing for [Name]".

### Identify recipient

Extract the recipient name from the invocation. Match against config files in `./config/` (excluding `briefing-setup-draft.json`). Load the matching `[first-name]-[last-name].json` config file.

If no config file matches, output: "No briefing configured for [Name]. Say 'configure briefing for [Name]' to set one up." and stop.

---

### Step 1 — Read configuration and history

Read `./config/[first-name]-[last-name].json`. Load all fields:
- Recipient name, title, Slack email, delivery time, timezone
- Full org structure (all pods, their type, manager, and team members)
- Email format pattern and domain
- Support email address
- Staircase ID map entries
- Name aliases

Read `./history/[first-name]-[last-name].json`. From the most recent entry in the `briefings` array, extract:
- All accounts flagged and their current status (`resolved`, `at_risk`, `churning`, `gone_dark`, etc.)
- How many consecutive weeks each unresolved issue has been flagged
- Expansion signals previously noted that are still unactioned (`resolved: false`)
- Accounts previously gone dark that have or haven't re-engaged

If the `briefings` array is empty → **Week 1**. No continuity language. Omit the "Still Watching" section entirely in Step 3.

Store all of this in working memory for use across all subsequent steps.

---

### Step 2 — Query Staircase AI

Using the Staircase AI MCP connector, pull current signals across the recipient's full org. Query at the account level for all members of pods typed `csm` and `renewals` in the config.

**Exclude** from queries: pods typed `support`, `partner_success`, and `operations` — these teams do not own accounts in Staircase.

For each account surfaced with a risk signal, attempt to retrieve a verbatim customer quote using `staircase_fetch_evidence`.

#### CSM Attribution

For every account, the CSM is the **Owner field** returned directly by Staircase. Always read and use this field — do not treat attribution as unknown if an Owner name is present.

Cross-reference the owner name against the org structure in the config to determine their manager. If the name matches someone in the config, attribute them to the correct manager. If the name does not match anyone, note them by name and flag that they are not in the known org tree.

**Name aliases**: Apply any alias mappings from the config. If a CSM appears under a former name, attribute them by their current name and correct manager.

**Numeric ID fallback**: If a numeric owner ID appears, look it up in `staircase.id_map` from the config. If found, substitute the resolved name and manager. If not found, output: "Owner ID [number] — not in known ID map, manual lookup required." Never pass a raw numeric ID into the briefing as a CSM name.

**Direct IC attribution**: If Staircase returns any of the `direct_ics` from the config as account owner, attribute them as: "Direct IC — [Recipient First Name]".

#### CSM Presence Detection

For every at-risk account, inspect the thread participants in Staircase. Classify each account using this logic:

- **CSM present** — Account owner appears in recent thread activity. No flag needed.
- **CSM absent — Renewals present** — Account owner not on thread; one or more Renewals team members are. Note names and titles.
- **CSM absent — Sales present** — Account owner not on thread; a Sales Rep appears under the "Sales Rep Name" field in Staircase. Note the rep.
- **CSM absent — Renewals and Sales present** — Both are on the thread, CSM is not.
- **CSM absent — Support present** — A Support team member appears on the thread, or the support email (from config) is a participant. Flag as support-routed.
- **CSM absent — Partner Success / Training present** — A member of a pod typed `partner_success` in the config is on the thread.
- **CSM absent — no internal contact detected** — No internal participants visible. Flag as unowned communication.

#### Email construction

Build all mailto links using the email format pattern from the config. Apply the pattern to construct manager email addresses from their names. Honor any exceptions stored in the config.

---

### Step 3 — Draft the briefing

Write in an executive narrative voice — direct, specific, actionable. Each section reads like a well-informed colleague briefing the exec, not a system report. Apply continuity language from the history throughout.

Continuity language examples:
- *"Flagged for the third consecutive week — no re-engagement plan confirmed."*
- *"Sentiment recovered from 10 → 34 since last week's flag — recovery call appears to be working."*
- *"Previously flagged — marking resolved."*

**If Week 1**: use no continuity language anywhere.

**Brevity rules — apply to every section:**
- Themes (Section 1): Max 3 sentences per theme. Lead with the pattern, state ARR exposure, give one-line implication.
- At-risk accounts (Section 2): One verbatim quote per account. Select the quote expressing emotional state or stated intent — prefer language with a direct consequence, a decision, or explicit sentiment. Skip procedural or logistical quotes.
- Gone dark entries (Section 4): Max 2 lines per entry.
- Expansion signals (Section 5): One-line signal summary per account.
- Manager questions (Section 6): Max 2 sentences per question — the setup and the ask.

---

#### Section 1 — 📊 What's Happening Across the Segment

Aggregate themes across the full book. Do not list accounts one by one. Answer: what patterns are showing up in customer conversations this week? What product areas, topics, or sentiment trends appear across multiple CSMs or pods? What is the overall tone of the business right now?

Max 3 sentences per theme.

---

#### Section 2 — 🔴 New Risks This Week

Only accounts where something **materially changed this week**. Previously flagged accounts belong in Section 3, not here.

For each account include:
- Account name + Staircase link, with status emoji directly after the name (see table below)
- CSM name (account owner) and manager name
- ARR, health score, and sentiment score
- One-sentence summary of what changed
- CSM presence status (from the Step 2 classification)
- One verbatim customer quote (most impactful only)

**Status emoji criteria:**

| Emoji | Status | When to use |
|---|---|---|
| 🔴 | Confirmed Churn / Churning | Formal non-renewal submitted, deactivation confirmed, or customer has explicitly stated intent to leave |
| 🟠 | At Risk | Active escalation, explicit evaluation of alternatives, sentiment below 20, health below 20, or renewal within 90 days with unresolved blockers |
| 🟢 | Recovering | Renewal recently signed AND no material blockers remaining |
| 🟡 | Watching | Previously flagged, no new deterioration, static signal |

*A renewal signing alone does not qualify an account as Recovering if material blockers remain. Use 🟠 until all post-close items are resolved. Use 🔴 any time a customer has stated explicit intent to leave, regardless of scores.*

---

#### Section 3 — ⚠️ Still Watching

Accounts flagged in prior briefings with no resolution. For each:
- State how many consecutive weeks the issue has been open
- Note whether the signal is improving, worsening, or static
- Include updated health/sentiment scores if they changed
- Include a new verbatim quote if available (most impactful only)

**If Week 1: omit this section entirely — no header, no placeholder, no mention of continuity tracking.**

---

#### Section 4 — 🌑 Accounts Gone Dark

Accounts with no customer-initiated engagement for 2+ weeks. Surface as early churn warnings before they become active churn cases.

For each (max 2 lines):
- Account name, Staircase link, ARR, CSM name, manager, last known engagement date
- Note if previously flagged and for how many consecutive weeks

---

#### Section 5 — 🌱 Expansion Signals

Positive signals: new locations, upsell interest, NPS promoters, positive sentiment spikes.

For each (one line per account):
- Account name, CSM name, manager, one-line signal summary
- Flag if the signal was noted in a prior briefing and is still unactioned

---

#### Section 6 — ✅ Questions to Ask This Week

Generate 2–3 specific, grounded questions for the exec to raise with each relevant manager. Each question must be:
- Tied to a specific account or signal from this briefing
- Max 2 sentences — the setup and the ask
- Friendly and curious in tone — checking in, not escalating

**Email consolidation**: Generate ONE mailto link per manager. All questions for a given manager are combined into a single pre-written email body — one click, one email.

For each manager email:
- **Subject line**: Most critical account as the subject, followed by "— and a couple of other things" if there are multiple questions
- **Body**: Friendly one-liner opening, each question as a short paragraph, close with "Thanks, [Recipient First Name]." Max 6–8 sentences total.
- **Tone**: Friendly and curious — checking in, not escalating.

At the top of Section 6, include this line in italics:
*💡 Click a manager's name to open a pre-written email ready to send.*

Format each manager link as:
> **[Manager Name](mailto:manager.email@company.com?subject=Account%20Name%20%E2%80%94%20and%20a%20couple%20of%20other%20things&body=...)** — Brief one-line summary of what the email covers.

---

#### Briefing Footer

At the very end of the briefing, include exactly this line:

```
Generated by Claude + Staircase AI · Next scheduled run: Monday, [next Monday's date] at [delivery time from config] [timezone from config]
```

If any Staircase admin notes apply (e.g., unresolved numeric IDs not in the map), add them on a separate line below the footer.

---

### Step 4 — Update history file

After drafting the briefing, append a new entry to `./history/[first-name]-[last-name].json`. If the file does not exist, create it using the schema below. Set `delivery_timestamp` to null now — it will be updated in Step 5.

Append to the `briefings` array:

```json
{
  "date": "YYYY-MM-DD",
  "week_range": "YYYY-MM-DD to YYYY-MM-DD",
  "delivery_timestamp": null,
  "canvas_url": null,
  "themes": [
    { "title": "...", "summary": "...", "trending": "up|down|stable" }
  ],
  "accounts_flagged": [
    {
      "name": "...",
      "staircase_url": "...",
      "arr": null,
      "csm": "...",
      "manager": "...",
      "health_score": 0,
      "sentiment_score": 0,
      "status": "at_risk | gone_dark | churning | confirmed_churn | recovering | resolved",
      "signal_type": "...",
      "summary": "...",
      "first_flagged": "YYYY-MM-DD",
      "resolved": false
    }
  ],
  "accounts_gone_dark": [
    { "name": "...", "staircase_url": "...", "first_flagged": "YYYY-MM-DD" }
  ],
  "expansion_signals": [
    { "account": "...", "signal": "...", "first_flagged": "YYYY-MM-DD", "resolved": false }
  ],
  "relationship_changes": [
    { "account": "...", "type": "csm_transition | champion_change", "detail": "..." }
  ]
}
```

---

### Step 5 — Deliver via Slack

#### Step 5a — Create the Slack Canvas

Using the Slack MCP connector, create a Slack Canvas:
- **Title:** `CS Executive Briefing — Week of [Monday's date]`
- **Content:** Full briefing in this exact order:

**1. Status Legend** (immediately after the title):
```
---
**Status Legend**
🔴 Confirmed Churn / Churning · 🟠 At Risk · 🟡 Watching · 🟢 Recovering · 🌑 Gone Dark · 🌱 Expansion Signal
---
```

**2. TL;DR block** (immediately after the legend):
```
## ⚡ 3 Things You Need to Know This Week

🔴 **Top Risk:** [One sentence — most critical risk, named account, ARR, why it matters now]

🌱 **Top Opportunity:** [One sentence — most actionable expansion signal, named account or CSM, what's ready to close]

✅ **Critical Question This Week:** [One sentence — most important question, named manager, named account, specific ask]
```

**3. Full briefing sections** (in order after the TL;DR):
- Use `#` for section headers
- Use `**bold**` for account names, scores, and key labels
- Use `-` for bullet points
- Use `>` for customer verbatim quotes
- Preserve all Staircase links as clickable hyperlinks
- Include all six sections in full
- Append the correct emoji status indicator directly after each account name in Section 2

Capture the Canvas URL returned by Slack.

#### Step 5b — Send Slack DM

Using the Slack MCP connector, send a Slack DM to the recipient's Slack email (from config):

> Your weekly CS briefing is ready — [Canvas URL from Step 5a]

After sending, update the current briefing entry in the history file:
- Set `delivery_timestamp` to the current ISO timestamp
- Set `canvas_url` to the Canvas URL

If Slack delivery fails for any reason, save the full briefing to `./output/[first-name]-[last-name]-[YYYY-MM-DD].md` and note the delivery failure in the history entry.

---

## STATUS CHECK

When user says "briefing status", "list briefings", or "briefing recipients":

1. List all files matching `./config/*.json` (excluding `briefing-setup-draft.json`)
2. For each, read the corresponding history file and show:
   - Recipient name and title
   - Last delivery date (from the most recent history entry's `delivery_timestamp`)
   - Next scheduled Monday
3. Flag any recipients where the last delivery was more than 8 days ago (possible scheduling gap)

---

## Rules

- **Never run execution mode without a valid config file.** If no config matches, redirect to setup mode.
- **Never invent account data, CSM names, health scores, or sentiment scores.** Surface only what Staircase returns.
- **Never re-list "Still Watching" accounts as new risks.** History continuity is the whole point of this skill.
- **Never pass a raw numeric owner ID into the briefing as a CSM name.** Always resolve via the ID map or flag for manual lookup.
- **A renewal signing alone is not Recovering** if material blockers remain open.
- **Fail gracefully on Staircase errors.** If a query times out or fails, note the failure in the briefing footer rather than silently omitting the section.
- **If Slack delivery fails**, save to `./output/` and notify the user — never silently discard the briefing.
- **Apply brevity rules to every section.** The exec has 60 seconds. Every word must earn its place.
- **Week 1 is clean.** No continuity language, no "Still Watching" section, no references to prior briefings.
