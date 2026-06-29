---
name: cs-sentiment
description: >
  Runs the CS Sentiment drafting workflow for CSMs. Finds Gainsight accounts that are due for a sentiment update, researches each one (Timeline, CTAs, health scores, Success Plans), and presents fully drafted sentiments directly in the Claude chat for review and approval. Once approved, posts to Gainsight Timeline.

  Use whenever the user mentions updating sentiments, CS Sentiment being due or stale, drafting a sentiment, or checking which accounts need a sentiment update. Triggers for: "I need my sentiments updated", "run my sentiments", "check my sentiment", "sentiment update", "which accounts need sentiment", "post it", "skip", or any CS Sentiment drafting or approval request.
---

## ORG CONFIGURATION
<!-- Admin: run "admin setup" to populate these values automatically, or fill them in directly.
     All fields except sf_severity_field and sf_severity_value are required before distributing to CSMs. -->

org_name:
gainsight_url:
staleness_field:
staleness_values:
sentiment_activity_type:
custom_timeline_fields: none
sf_severity_field: none
sf_severity_value: none

---

# CS Sentiment Drafting Agent

Find Gainsight accounts due for a CS Sentiment update, research each one, draft a proposed sentiment, and present all drafts in this chat for review. Reply to approve, skip, or edit — approved sentiments post directly to Gainsight Timeline.

The skill's base directory is provided at the top of this context as `Base directory for this skill: <path>`.

---

## STEP 0 — SETUP CHECK

Read the **ORG CONFIGURATION** section at the top of this file.

- **If any required field is blank** (`org_name`, `gainsight_url`, `staleness_field`, `staleness_values`, `sentiment_activity_type`): run **Admin Setup** below. Do not proceed until all required fields are populated.
- **If all required fields are populated and no `csm-profile` exists** in `.auto-memory/`: run **CSM Onboarding** below.
- **If all required fields are populated and `csm-profile` exists:** load it into context and proceed to Step 1.

---

## ADMIN SETUP

*This runs once per installation. The admin completes it before distributing the skill to their team.*

1. Tell the user: "Let's configure the CS Sentiment skill for your org — I have a few questions."

2. Collect and validate each value **one at a time** in the order below. On a validation failure, explain what was found and re-ask — do not proceed until each critical field is confirmed.

   **a. Org name**
   Ask: "What is your organization's name?"
   No validation required.

   **b. Gainsight instance URL**
   Ask: "What is your Gainsight instance URL? (e.g., `yourorg.us2.gainsightcloud.com`)"
   No validation required — used for constructing links only.

   **c. Staleness field** *(critical)*
   Ask: "What is the API name of the field on the Gainsight Company object that identifies accounts due for a sentiment update?"
   **Validate:** Call `get_object_metadata` for the Company object. Check whether the provided name exists in the schema.
   - If found: confirm ("✅ Found `<field>` on the Company object.") and proceed.
   - If not found: tell the admin ("⚠️ I couldn't find `<field>` on the Company object. Here are fields that may be relevant: [list any fields from the metadata whose names suggest staleness, sentiment, or recency].") and re-ask. Do not proceed until a valid field is confirmed.

   **d. Staleness values** *(critical)*
   Ask: "What values on that field indicate an update is needed? List them exactly as they appear in Gainsight, comma-separated."
   **Validate:** If the field is a picklist type, call `get_picklist_values` for it and compare each provided value against the available options.
   - If all values match: confirm and proceed.
   - If any value doesn't match: tell the admin ("⚠️ I didn't find `<value>` in the picklist. Available values are: [list]. Did you mean one of these?") and re-ask.
   - If the field is not a picklist type: skip validation and proceed.

   **e. Sentiment activity type** *(critical)*
   Ask: "What is the name of your CS Sentiment activity type in Gainsight Timeline?"
   **Validate:** Call `get_activity_types_config` and check whether the provided name matches a configured type.
   - If found: confirm and proceed.
   - If not found: tell the admin ("⚠️ I couldn't find an activity type named `<name>`. Configured types are: [list all names].") and re-ask. Do not proceed until a valid type is confirmed.

   **f. Custom Timeline fields** *(optional)*
   Ask: "Do you have custom fields on that activity type you'd like populated when a sentiment is posted? If yes, provide each field's API name and a brief description of what value to set. If none, say 'none'."
   **Validate (if not "none"):** Cross-reference each provided field name against the fields on the confirmed activity type from step (e).
   - If any field name isn't found: warn the admin and ask them to confirm or correct before proceeding.

   **g. Salesforce case severity** *(optional)*
   Ask: "Do you filter Salesforce support cases by a severity field? If yes, provide the field API name and the highest-severity value (e.g., `MySeverityField__c`, `Critical`). If not, say 'none'."
   **Validate (if not "none"):** Run a test query — `SELECT Id FROM Case WHERE <sf_severity_field> = '<sf_severity_value>' LIMIT 1` — and check for errors.
   - If the query succeeds (even with zero results): confirm and proceed.
   - If the query errors: tell the admin ("⚠️ That field or value returned an error — please double-check the API name and value.") and re-ask.

3. Write all confirmed values into the **ORG CONFIGURATION** section of this file using `str_replace`, replacing each blank field with the validated input. The path to this file is `<base_dir>/SKILL.md` where `<base_dir>` comes from the `Base directory for this skill:` context variable. For optional fields where the admin said "none", write `none`.

4. Call `present_files` with `<base_dir>/SKILL.md` to surface the configured skill file as a download.

5. Confirm in chat:
   > ✅ **Org configuration saved.** Your configured skill file is ready to download above.
   >
   > **To distribute:** share the `SKILL.md` file with your CSMs and have each one install it in their Cowork workspace. The first time each CSM runs it, they'll be asked for their name, email, and timezone only — no schema knowledge required on their end.
   >
   > To reconfigure at any time, say *"admin setup"* again.

---

## CSM ONBOARDING

*This runs once per CSM, on their first use of the distributed skill.*

1. Tell the user: "Welcome! I just need a couple of details to get you set up."

2. Ask the following **one at a time**, waiting for a reply after each:
   - "What is your full name and work email address?"
   - "What timezone are you in? (e.g., US/Eastern, US/Pacific, Europe/London)"

3. Using the email provided, resolve the CSM's **Gainsight GSID** via `resolve_user`.

4. Determine the queue file path: resolve the current workspace folder's absolute path and set `queue_file_path` to `<workspace_path>/pending_sentiments.json`. Create the file as `[]` if it doesn't exist.

5. Save the profile to `.auto-memory/csm-profile.md`, combining personal details with the org config values from the ORG CONFIGURATION section:
   ```
   csm_name: <full name>
   csm_email: <email>
   csm_gsid: <resolved GSID>
   timezone: <timezone>
   queue_file_path: <absolute path>

   org_name: <from ORG CONFIGURATION>
   gainsight_url: <from ORG CONFIGURATION>
   staleness_field: <from ORG CONFIGURATION>
   staleness_values: <from ORG CONFIGURATION>
   sentiment_activity_type: <from ORG CONFIGURATION>
   custom_timeline_fields: <from ORG CONFIGURATION>
   sf_severity_field: <from ORG CONFIGURATION>
   sf_severity_value: <from ORG CONFIGURATION>
   ```
   Add a pointer to `.auto-memory/MEMORY.md`.

6. Create a scheduled task named `cs-sentiment-daily-drafting` with cron `0 8 * * 1-5` (notifyOnCompletion: false) and prompt:
   > "Load csm-profile from .auto-memory/csm-profile.md and run the CS Sentiment drafting workflow: identify stale accounts, research each in parallel, draft sentiments, and present them in this chat for approval. Use the field names and values in the profile throughout — do not substitute your own. Use queue_file_path from the profile for pending_sentiments.json. When querying Gainsight, filter only by Csm = <csm_gsid> and <staleness_field> IN <staleness_values>. No other filters."

7. Confirm in chat:
   > ✅ **You're all set!** Each weekday at 8:00 AM I'll draft sentiments for any due accounts and post them here for your review. Reply *"Post all"* to publish everything, *"Post [Account]"* for one, or *"Skip [Account]"* to hold. You can also say *"Run my sentiments"* any time to kick off a draft on demand.

---

## STEP 1 — HANDLE PENDING APPROVALS

Read `pending_sentiments.json` at `queue_file_path`. If entries exist with `status: "awaiting_approval"` **and** the user's current message expresses approval, skip, or edit intent, handle it now and exit.

**Approval triggers** (case-insensitive): "post it", "post all", "post [company]", "send it", "go ahead", "approved", "lgtm", "yes", "looks good", 👍, or any clear affirmative.
- "Post all" or a general affirmative with no company specified: approve all `"awaiting_approval"` entries.
- Otherwise: match the company name to its queue entry.

**Skip triggers**: "skip", "hold", "not yet", "pass", "skip [company]".

**Edit**: any other reply is treated as an edit request. Apply changes to `content_html`, update `subject` and `sentiment` if the color changed, re-present the revised draft in chat for re-approval.

**For each approved entry**, post to Gainsight Timeline:
- Call `create_timeline_activity` with `company_id`, `subject`, `content` (`content_html`), `activity_type` (`sentiment_activity_type` from profile), `activity_date`: today (YYYY-MM-DD).
- If `custom_timeline_fields` is not `none`, include them in `custom_field_values`, resolving each field's value from the draft content as described. Omit `custom_field_values` entirely if `none`.
- On success: set `status: "posted"`, add `posted_at` and `gainsight_activity_id`. Confirm in chat:
  > <color_emoji> **[Sentiment] CS Sentiment for [Company]** posted to Gainsight Timeline.
  > [View entry](<gainsight_url>/v1/ui/timeline#/activities/<gainsight_activity_id>)
- On failure: keep `status: "awaiting_approval"`, report the error in chat, suggest retrying.

**For each skipped entry**: set `status: "skipped"`, add `skipped_at`. Confirm: "↩️ Held the draft for **[Company]** — say *"Run my sentiments"* to get a fresh draft."

Save the updated queue. **Exit — do not proceed to Step 2.**

If no approval intent is detected, proceed to Step 2.

---

## STEP 2 — IDENTIFY STALE ACCOUNTS

Query Gainsight for all companies where `Csm = <csm_gsid>` and `<staleness_field>` is any of `<staleness_values>` (all from `csm-profile.md`). Retrieve: company name, GSID, last sentiment date, and current staleness value.

**Do not add any other filters** — additional filters can silently drop matching accounts.

If no accounts found: "✅ All your accounts are up to date — no CS Sentiment updates needed right now." Exit.

---

## STEP 3 — RESEARCH ALL ACCOUNTS IN PARALLEL

Issue **all** research calls for **all** accounts simultaneously.

**Research window:** Use the later of `last_sentiment_date` and 90 days ago as `research_start_date`. If `last_sentiment_date` is null, use 90 days ago.

**Gainsight:** Recent Timeline activities, open CTAs (especially Risk type), health/scorecard data, Success Plans.

**Salesforce** (use `soqlQuery`):
- Opportunities: `SELECT Id, Name, StageName, Amount, CloseDate, LastModifiedDate FROM Opportunity WHERE AccountId = '<SF_ID>' AND LastModifiedDate >= <research_start_date> ORDER BY LastModifiedDate DESC LIMIT 5`
- Cases: If `sf_severity_field` and `sf_severity_value` are set in the profile, query high-severity cases only: `SELECT Id, Subject, Status, LastModifiedDate FROM Case WHERE AccountId = '<SF_ID>' AND <sf_severity_field> = '<sf_severity_value>' AND LastModifiedDate >= <research_start_date> ORDER BY LastModifiedDate DESC LIMIT 10`. If not set, query all recent open cases: `SELECT Id, Subject, Status, LastModifiedDate FROM Case WHERE AccountId = '<SF_ID>' AND Status != 'Closed' AND LastModifiedDate >= <research_start_date> ORDER BY LastModifiedDate DESC LIMIT 10`.
- Include high-priority cases individually. For lower-priority cases, only surface if 3+ repeated issues of the same type are evident — describe the trend, not each case.
- Find Salesforce Account ID via: `FIND {"<Company Name>"} IN NAME FIELDS RETURNING Account(Id, Name)`

**Microsoft 365:**
- Recent emails mentioning the company (`outlook_email_search`, after research start date)
- Recent Teams messages mentioning the company (`chat_message_search`, after research start date)

**Slack:** Recent mentions across channels (`slack_search_public_and_private` with company name, after research start date).

**Zoom** (if available): Search recent meetings (`search_meetings`). Extract key signals from transcripts/summaries (`get_meeting_assets`) — customer tone, renewal or budget discussions, escalations, feature requests, product value statements. Prioritize transcripts as highest-signal source.

Draft each sentiment using the template and rubric below, then proceed to Step 4.

**Personnel mentions — two rules:**
1. Titles must come from account-specific sources only. Never infer a role from messages in other account channels.
2. Only call someone "new" if their assignment falls within the sentiment window. If it predates the last sentiment, describe them neutrally.

---

## STEP 4 — PRESENT DRAFTS IN CHAT

**Duplicate check:** If an entry already exists in `pending_sentiments.json` with `status: "awaiting_approval"` for a given company, skip drafting that company. If all are already queued: "All due sentiments are already pending your review — reply to approve, skip, or edit."

Convert `content_html` to readable markdown: `<strong>` → `**bold**` · `<em>` → `_italic_` · `<p>` → blank line · `<ul><li>` → `• `

Present all drafts in chat:

---
📋 **CS Sentiment Drafts — [Today's Date]**

*Reply with:* **"Post all"** · **"Post [Company]"** · **"Skip [Company]"** · or send edits and I'll revise before posting.

---
**[Company Name]** · <color_emoji> [Sentiment] · Last updated: [date] · Status: [staleness value]

[Full draft in markdown]

---
*(repeat for each account)*

---

Append each draft to `pending_sentiments.json`:

```json
{
  "company_name": "...",
  "company_id": "...",
  "subject": "...",
  "sentiment": "Red|Amber|Green",
  "content_html": "...",
  "risk_cta_gsids": ["..."],
  "drafted_at": "<ISO timestamp>",
  "status": "awaiting_approval"
}
```

---

## SENTIMENT COLOR RUBRIC

| Color | Renewal Posture | Required Conditions |
|-------|----------------|---------------------|
| 🟢 **Green** | Little or no risk to renewal. | Customer has stated they will renew, **or** CS feels strongly this will occur. No significant contraction expected. |
| 🟡 **Amber** | Renewal with contraction. | Customer likely to renew but deal size will shrink. **Must be supported by** a Medium or High Risk CTA documenting the contraction risk. |
| 🔴 **Red** | Full churn risk. | Customer has stated they will not renew, **or** CS feels strongly they won't. **Must be supported by** a High or Urgent Risk CTA documenting churn risk. |

- **Gainsight CTAs** are the primary evidence source for Amber and Red. If no matching CTA exists but evidence warrants the color, assign it, note the missing CTA in Risks, and recommend the CSM open one.
- **Salesforce opportunity data** validates the renewal outlook — stalled or contracting renewals support Amber or Red.
- **High-priority cases, escalations, and negative signals** across sources act as risk amplifiers. Document the source explicitly.
- **Positive signals** (executive engagement, strong adoption, expansion conversations) support Green or stabilize Amber.
- **If a customer has stated non-renewal** in any source but no CTA exists: assign Red and flag the missing CTA.
- **Never assign Amber or Red without documenting** the specific signals in the Risks section.

---

## SENTIMENT DRAFT TEMPLATE

**Subject:** [Red/Amber/Green] CS Sentiment for [Company Name] — [Today's Date]

**Overview:** Lead with the renewal posture and color rationale. Synthesize signals across all sources — Gainsight Timeline, open CTAs, health scores, Salesforce activity, emails, Teams, and Slack. Highlight what's driving the sentiment color, recent momentum or stalls, and strategic initiatives underway.

**Risks:** One bullet per open Risk CTA or significant risk signal (escalations, support cases, negative signals, stalled opportunities). Include the source, description, and mitigation plan. **Required for Amber or Red** — must contain the CTA or signal that justifies the color.

**System of Record Status:** Which product modules are systems of record, how they're used, competing tools present, and next steps to deepen adoption.

**Formatting (strictly required):**
- Write as valid HTML using `<p>`, `<strong>`, `<ul>`/`<li>`, and `<em>` tags — required for correct rendering in Gainsight Timeline.
- Convert to clean markdown before presenting in chat — no raw HTML visible in the preview.
- End with: `<em>This entry was created via Claude (Anthropic AI).</em>`
