# CS Executive Briefing — Weekly Scheduled Task Prompt Template
## Version: 1.0
## Compatible with: Claude Cowork + Staircase AI MCP + Slack MCP

---

## HOW TO USE THIS TEMPLATE

This prompt is designed to run as a weekly scheduled task in Claude Cowork. Before using it:

1. Replace every value marked `[CONFIGURE]` with your recipient's actual information
2. Fill in the org structure tables with your recipient's managers and CSMs
3. Add any known Staircase numeric IDs to the ID map
4. Initialize an empty history JSON file at `./history/[first-name]-[last-name].json`
5. Run once manually to validate output before scheduling
6. Create a scheduled task in Cowork — weekly, Monday at 8AM recipient local time

Fields marked `[OPTIONAL]` can be left blank or removed if not applicable.

---

## TASK INSTRUCTIONS

You are generating a weekly CS executive briefing for **[CONFIGURE: Recipient Full Name]**, [CONFIGURE: Title] of Customer Success. This briefing is designed to give [CONFIGURE: First Name] a narrative view of what's happening across their org — not an account inventory, but a thematic summary of patterns, risks, momentum, and questions they should bring into their week.

Follow every step below in order. Do not skip steps.

---

## STEP 1 — READ HISTORY FILE

Read the history file at `./history/[CONFIGURE: first-name-last-name].json`.

- If the file exists and contains prior briefings, extract:
  - All accounts flagged in the most recent briefing and their current status
  - How many consecutive weeks each unresolved issue has been flagged
  - Any expansion signals previously noted that are still unactioned
  - Any accounts previously gone dark that have or haven't re-engaged
- If the file does not exist or `briefings` array is empty, treat this as **Week 1** — no prior context, no continuity language.

Store this context in memory. You will use it in Steps 3 and 4.

---

## STEP 2 — QUERY STAIRCASE AI

Using the Staircase AI MCP connector, pull current signals across the recipient's full org. Query at the account level for all CSMs listed in the org structure below.

For each account surfaced with a risk signal, attempt to retrieve a verbatim customer quote using `staircase_fetch_evidence`.

### CSM Attribution

For every account, the CSM is the **Owner field** returned directly by Staircase on the account record. Always read and use this field — do not treat CSM attribution as unknown or unresolved if an Owner name is present in Staircase.

Once you have the Owner name, cross-reference the org structure below to determine their manager. If the Owner name matches someone in the org structure, use that manager attribution. If the Owner name does not match anyone in the org structure, note them by name and flag that they are not in the known org tree — do not leave the CSM field blank.

**[OPTIONAL] Name aliases:** If any CSMs in your org have recently changed their name (e.g. marriage), add aliases here:
- [Former Name] and [Current Name] are the same person. Their email is [email]. Attribute to [Manager] pod.

**Numeric ID fallback:** Staircase occasionally returns a numeric owner ID instead of a name — particularly for accounts created during a POC or data migration. If a numeric ID appears, resolve it using the table below before declaring the owner unknown:

| Staircase Owner ID | Resolves To | Pod / Manager |
|---|---|---|
| `[CONFIGURE: numeric ID]` | [CONFIGURE: CSM Name] | [CONFIGURE: Pod / Manager] |
| `[CONFIGURE: numeric ID]` | [CONFIGURE: CSM Name] | [CONFIGURE: Pod / Manager] |

*Add rows as needed. If a numeric ID appears that is not in this table, output it as: "Owner ID [number] — not in known ID map, manual lookup required." Never pass a raw numeric ID through to the briefing output as a CSM name.*

**Direct IC attribution:** The following people report directly to [CONFIGURE: Recipient First Name] with no pod manager in the chain. If Staircase returns any of these names as account owner, attribute them as "Direct IC — [CONFIGURE: Recipient Name]":
- [CONFIGURE: IC Name] — [Title]
- [CONFIGURE: IC Name] — [Title]
- *Add or remove as needed*

### CSM Presence Detection

For every at-risk account, inspect the thread participants in Staircase and determine whether the assigned CSM is present in active communications. Classify each account using the following logic:

- **CSM present** — Account owner appears as a participant in recent thread activity. No flag needed.
- **CSM absent — Renewals present** — Account owner is not on the thread, but one or more members of the Renewals team are. Note the specific contact(s) by name and title.
- **CSM absent — Sales present** — Account owner is not on the thread, but a Sales Rep is listed under the "Sales Rep Name" field in Staircase. Note the sales rep by name.
- **CSM absent — Renewals and Sales present** — Both are on the thread but the CSM is not. Note all contacts by name and role.
- **CSM absent — Support present** — A member of the Support org is on the thread, or the thread involves [CONFIGURE: your support email address e.g. support@yourcompany.com]. Note the contact by name if identifiable, or flag as "Support contact ([CONFIGURE: Support Manager] org)" if not. Flag support@ appearances explicitly as support-routed communications.
- **[OPTIONAL] CSM absent — Partner Success / Training present** — A member of the Partner Success or Training team is on the thread. Note them by name and title.
- **CSM absent — no internal contact detected** — No internal participants visible on the thread. Flag as unowned communication.

### Email Construction

For all mailto links in the briefing, construct email addresses using the format:
`[CONFIGURE: firstname.lastname@yourcompany.com]`

For hyphenated last names, use the full hyphenated form.

*[OPTIONAL] Add any email format exceptions here, e.g. for name aliases or non-standard formats.*

---

### Org Structure

**[CONFIGURE: Recipient Full Name]** — [CONFIGURE: Title], Customer Success

#### [CONFIGURE: Pod Name e.g. "Strategic Customer Success"] — Manager: [CONFIGURE: Manager Name]
| Name | Title | Location |
|---|---|---|
| [CONFIGURE: CSM Name] | [CONFIGURE: Title] | [CONFIGURE: Location] |
| [CONFIGURE: CSM Name] | [CONFIGURE: Title] | [CONFIGURE: Location] |
*Add rows as needed*

#### [CONFIGURE: Pod Name e.g. "Renewals"] — Manager: [CONFIGURE: Manager Name]
| Name | Title | Location |
|---|---|---|
| [CONFIGURE: Name] | [CONFIGURE: Title] | [CONFIGURE: Location] |
| [CONFIGURE: Name] | [CONFIGURE: Title] | [CONFIGURE: Location] |
*Add rows as needed*

#### [OPTIONAL: Pod Name e.g. "Partner Success / Training"] — Manager: [CONFIGURE: Manager Name]
*Note: This team does not own customer accounts and will not appear as CSM or Owner in Staircase. Their presence on a thread indicates a training or enablement touchpoint. Classify accordingly.*
| Name | Title | Location |
|---|---|---|
| [CONFIGURE: Name] | [CONFIGURE: Title] | [CONFIGURE: Location] |
*Add rows as needed*

#### [OPTIONAL: Support Org] — Manager: [CONFIGURE: Support Manager Name]
*Note: This org handles support cases primarily via [CONFIGURE: your ticketing system e.g. Salesforce]. Their names will not appear as CSM or Owner in Staircase. If a member of this org appears on a thread, or if [CONFIGURE: support@yourcompany.com] is a participant, classify as Support present and flag as support-routed.*

#### [OPTIONAL: CS Operations — Excluded from Staircase queries]
- **[CONFIGURE: Name]** — [Title]. Internal/ops role with no customer-facing accounts. Do not query Staircase for this pod. Do not include in pod health output.

#### Individual Contributors (direct to [CONFIGURE: Recipient First Name], no pod)
- [CONFIGURE: Name] — [Title] ([Location])
- [CONFIGURE: Name] — [Title] ([Location])
*Add or remove as needed. Remove this section entirely if the recipient has no direct IC reports.*

---

## STEP 3 — DRAFT THE BRIEFING

Draft the briefing using the sections below. Write in an executive narrative voice — direct, specific, and actionable. Avoid bullet-point dumps. Each section should read like a well-informed colleague briefing the exec, not a system report.

Use continuity language from Step 1 wherever applicable:
- *"Flagged for the third consecutive week — no re-engagement plan confirmed."*
- *"Sentiment recovered from 10 → 34 since last week's flag — recovery call appears to be working."*
- *"Previously flagged — marking resolved."*

If this is Week 1, omit continuity language entirely.

### Brevity Rules — apply to every section
- **Section 1:** Maximum 3 sentences per theme. Lead with the pattern, state the ARR exposure, give the one-line implication.
- **Section 2:** One verbatim quote per account maximum. Select the quote that most clearly expresses the customer's emotional state or stated intent — preference for language that includes a direct consequence, a decision, or explicit sentiment. Avoid quotes that are procedural or logistical.
- **Section 4:** Maximum 2 lines per account entry.
- **Section 5:** One-line signal summary per account.
- **Section 6:** Maximum 2 sentences per manager question — the setup and the ask.

---

### Section 1 — 📊 What's Happening Across the Segment

Aggregate themes across the full book. Do not list accounts one by one. Answer: what patterns are showing up in customer conversations this week? What product areas, topics, or sentiment trends are appearing across multiple CSMs or pods? What is the overall tone of the business right now?

Maximum 3 sentences per theme. Follow brevity rules.

---

### Section 2 — 🔴 New Risks This Week

Only accounts where something **materially changed this week**. Do not re-list previously flagged accounts here (those belong in Section 3).

For each account include:
- Account name and Staircase link
- CSM name (account owner) and manager name
- ARR, health score, and sentiment score
- One-sentence summary of what changed
- **CSM presence status** — use the classification from Step 2
- One verbatim customer quote (most impactful only — follow brevity rules)

**Status indicators:** For every account, append an emoji status indicator directly after the account name using these criteria:

| Status | Emoji | When to use |
|---|---|---|
| Confirmed Churn / Churning | 🔴 | Formal non-renewal submitted, deactivation confirmed, or customer has explicitly stated intent to leave |
| At Risk | 🟠 | Active escalation, explicit evaluation of alternatives, sentiment below 20, health below 20, or renewal within 90 days with unresolved blockers |
| Recovering | 🟢 | Renewal recently signed AND no material blockers remaining |
| Watching | 🟡 | Previously flagged, no new deterioration, static signal |

*A renewal signing alone does not qualify an account as Recovering if material blockers remain open. Use 🟠 until all post-close items are resolved. Use 🔴 any time a customer has stated explicit intent to leave, regardless of health or sentiment score.*

---

### Section 3 — ⚠️ Still Watching

Accounts flagged in prior briefings with no resolution. For each:
- State how many consecutive weeks the issue has been open
- Note whether the signal is **improving**, **worsening**, or **static**
- Include updated health/sentiment scores if they changed
- Include a verbatim quote if a new one is available (most impactful only)

If no prior history exists (Week 1), **omit this section entirely** — do not include the header, placeholder text, or any mention of continuity tracking.

---

### Section 4 — 🌑 Accounts Gone Dark

Accounts with no customer-initiated engagement for 2+ weeks. Surface as early churn warnings.

For each (maximum 2 lines per entry):
- Account name, Staircase link, ARR, CSM name, manager, last known engagement date
- Note if previously flagged and for how many weeks

---

### Section 5 — 🌱 Expansion Signals

Positive signals: new locations, upsell interest, NPS promoters, positive sentiment spikes. Note if any signal has been sitting unactioned from a prior week.

For each (one line per account):
- Account name, CSM name, manager, one-line signal summary, flag if unactioned from prior week

---

### Section 6 — ✅ Questions to Ask This Week

Generate 2–3 specific, grounded questions for the exec to raise with each relevant manager. Questions must be:
- Tied to a specific account or signal from this briefing
- Maximum 2 sentences — the setup and the ask
- Friendly and curious in tone — not punitive or alarming

**Email consolidation rule:** Generate ONE mailto link per manager. All questions for a given manager are combined into a single pre-written email body — one click, one email, all questions included.

For each consolidated manager email:
- **Subject line:** Use the most critical account as the subject, followed by "— and a couple of other things" if multiple questions
- **Body:** Open with a friendly one-liner, list each question as a short paragraph, close with "Thanks, [CONFIGURE: Recipient First Name]." Maximum 6–8 sentences total.
- **Tone:** Friendly and curious — checking in, not escalating

**Canvas instruction note:** At the top of Section 6 in the Canvas, include this line in italics:
*💡 Click a manager's name to open a pre-written email ready to send.*

Format:
> **[Manager Name](mailto:manager.email@yourcompany.com?subject=Account%20Name%20%E2%80%94%20and%20a%20couple%20of%20other%20things&body=Email%20body%20here)** — Brief one-line summary of what the email covers.

---

### Briefing Footer

At the very end of the briefing, include exactly this footer:

```
Generated by Claude + Staircase AI · Next scheduled run: Monday, [next Monday's date] at [CONFIGURE: delivery time] [CONFIGURE: timezone]
```

*[OPTIONAL] If any Staircase admin notes are relevant (e.g. unresolved duplicate IDs), include them on a separate line below the footer.*

---

## STEP 4 — UPDATE HISTORY FILE

After drafting the briefing, update `./history/[CONFIGURE: first-name-last-name].json` with this week's data.

If the file does not exist, create it using the schema below. If it exists, append a new entry to the `briefings` array.

```json
{
  "recipient": {
    "name": "[CONFIGURE: Full Name]",
    "slack_email": "[CONFIGURE: email@yourcompany.com]",
    "title": "[CONFIGURE: Title]",
    "managers": ["[CONFIGURE: Manager Name]"],
    "direct_ics": ["[CONFIGURE: IC Name]"]
  },
  "briefings": [
    {
      "date": "YYYY-MM-DD",
      "week_range": "YYYY-MM-DD to YYYY-MM-DD",
      "delivery_timestamp": null,
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
  ]
}
```

---

## STEP 5 — DELIVERY

Deliver the briefing using the following two-step Slack process:

### Step 5a — Create the Slack Canvas

Using the Slack MCP connector, create a Slack Canvas with the following:

- **Title:** `CS Executive Briefing — Week of [Monday's date]`
- **Content:** The full briefing formatted in Slack Canvas markdown, in this exact order:

**1. Status Legend** — immediately after the title:
```
---
**Status Legend**
🔴 Confirmed Churn / Churning · 🟠 At Risk · 🟡 Watching · 🟢 Recovering · 🌑 Gone Dark · 🌱 Expansion Signal
---
```

**2. TL;DR block** — immediately after the legend:
```
## ⚡ 3 Things You Need to Know This Week

🔴 **Top Risk:** [One sentence — the single most critical risk, named account, ARR, and why it matters now]

🌱 **Top Opportunity:** [One sentence — the single most actionable expansion signal, named account or CSM, ARR, and what's ready to close]

✅ **Critical Question This Week:** [One sentence — the single most important question, named manager, named account, specific ask]
```

**3. Full briefing sections** — in order after the TL;DR:
- Use `#` for section headers
- Use `**bold**` for account names, scores, and key labels
- Use `-` for bullet points
- Use `>` for customer verbatim quotes
- Preserve all Staircase links as clickable hyperlinks
- Include all six sections in full

**Status indicators:** For every account in Section 2, append the correct emoji status indicator directly after the account name using the criteria defined in Step 3.

After creation, capture the Canvas URL returned by Slack.

### Step 5b — Send Slack DM to Recipient

Using the Slack MCP connector, send a Slack DM to **[CONFIGURE: recipient-email@yourcompany.com]** with the following message:

> Your weekly CS briefing is ready — [Canvas link from Step 5a]

- Log the Canvas URL and DM timestamp to the history file under `delivery_timestamp`.

---

## SCHEDULING

When this task is set up as a scheduled run in Cowork:
- **Frequency:** Weekly
- **Day/Time:** Every Monday at [CONFIGURE: 8:00 AM recipient local time]
- Tool permissions should be pre-approved for unattended execution:
  - Staircase AI MCP — read access
  - Filesystem — read/write access to `./history/`
  - Slack MCP — Canvas create and DM send access

---

*Built with Claude + Staircase AI. See CS-Briefing-Automation-README.md for full documentation.*
