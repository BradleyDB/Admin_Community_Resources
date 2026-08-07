# CS Executive Briefing v2 — Weekly Scheduled Task Prompt

## Recipient: [VP/Head of Customer Success — replace with your recipient's name and title]

## Status: TEMPLATE — customize the Org Structure section and all named individuals before deploying

## Version: 1.0 — Sanitized public template



---



## TASK INSTRUCTIONS



You are generating a weekly CS executive briefing for **[Recipient Name]**, [Title — e.g. VP of Customer Success]. This briefing is designed to give the recipient a narrative view of what's happening across their org — not an account inventory, but a thematic summary of patterns, risks, momentum, and questions they should bring into their week.



Follow every step below in order. Do not skip steps.



---



## STEP 1 — READ HISTORY FILE



Read the history file at `./history/marco-poggio-v2.json`.

If not found there, also check `./marco-poggio-v2.json` in the root working folder.

If still not found, treat this as **Week 1** — no prior context, no continuity language.



- If the file exists and contains prior briefings, extract:

  - All accounts flagged in the most recent briefing and their current status

  - How many consecutive weeks each unresolved issue has been flagged

  - Any expansion signals previously noted that are still unactioned

  - Any accounts previously gone dark that have or haven't re-engaged

- If the file does not exist or `briefings` array is empty, treat this as **Week 1** — no prior context, no continuity language.



Store this context in memory. You will use it in Steps 4 and 5.



---



## STEP 2 — QUERY STAIRCASE AI

Using the Staircase AI MCP connector, pull current signals across the recipient's full org. **Do not run a single org-wide query — the Staircase report API truncates at 500 rows, which silently drops accounts.** Instead, run scoped queries per pod/team and merge the results before proceeding.

Apply all Exclusion Rules below before processing any query results.

**⚠️ Admin action required:** The queries below are a template. Replace the bracketed pod names and team member lists with your organization's actual reporting structure — see the Org Structure section further down, which you'll also need to fill in with your real org chart.

### Query 2a — [Pod/Team Name] (e.g. "Strategic CS Pod")

Query all at-risk accounts owned by: [list every team member in this pod by name]. Include accounts with **Risk Level 4 or 5**, AND any account with an **active Churn Risk flag** regardless of whether a Risk Level score is populated.

### Query 2b — [Additional Pod/Team Name, if applicable]

Repeat the same query structure for each additional pod/team in your org. Add one Query 2-x block per pod.

### Query 2c — Individual Contributors / Direct Reports (no pod manager)

Query all at-risk accounts owned by: [list any team members who report directly to the briefing recipient with no intermediate manager]. Same Risk Level / Churn Risk flag criteria as above.

*Note: if your org has a group without a dedicated manager (e.g. during a hiring gap), attribute those accounts directly to the briefing recipient with no pod manager in the chain, and note this explicitly in the briefing so it's clear this is a temporary structure.*

**Renewal date note:** Check whether your Staircase instance's renewal date field has any known offset from your CRM's actual contract end date (this varies by tenant configuration). If there is a consistent offset, document it here and use the Staircase date consistently rather than trying to adjust it per-account.

### Query 2d — Gone Dark (all pods)

Run a separate gone-dark query across all CSMs in the org (excluding any automated/digital-CSM accounts and any business units you've deliberately scoped out — see Exclusion Rules below). Return all accounts with no customer engagement regardless of how long they have been dark. Include: account name, CSM owner, ARR, last engagement date, renewal date, health score.

### Query 2e — Expansion Signals (by pod)

Run one expansion query per pod to avoid the 500-row truncation limit, mirroring the pod structure from Query 2a/2b above. Query accounts with expansion readiness 4 or 5 for each pod separately, then merge results.

Merge all results before proceeding. Exclude any automated/digital-CSM accounts and any excluded business units.

After all queries complete, merge the results into a single unified account list. For each account surfaced with a risk signal, attempt to retrieve a verbatim customer quote using `staircase_fetch_evidence`.

### Exclusion Rules — Apply Before Any Processing

**1 — Digital/Automated CSM accounts**

If your organization has an automated or "digital" CSM covering long-tail accounts (a bot/system rather than a human), exclude that entity from all org attribution, manager questions, and main briefing sections. Surface risk signals in Section 2c only, as lightweight visibility — not full escalation treatment. [Replace with your actual digital-CSM name/identifier, if applicable, or delete this rule if you don't have one.]

**2 — Excluded business units / account categories**

If your organization has any account category that should be excluded from this briefing entirely (e.g. accounts from a recent acquisition still being integrated, a business unit tracked separately, test/demo accounts), define the exclusion criteria here — typically a Business Unit field value or a specific list of owner names to exclude. [Replace this placeholder with your actual exclusion criteria, or delete this section if not applicable.]

### CSM Attribution

**UNIVERSAL RULE:** The CSM is always and only the **Owner field** returned directly on the Staircase account record. Pod query grouping is for scoping only — never for attribution.

**Name alias table:** If anyone in your org goes by a different name in Staircase/Gainsight than their common name (maiden name, nickname, etc.), document the mapping here so attribution and email construction don't break. Starts empty — populate with real examples from your own team as you find them. [Example format: "Jane Smith (may appear as Jane Doe in some systems) — email: jane.smith@example.com"]

**Known CSM attribution overrides:** If you find specific accounts that consistently misattribute to the wrong owner in Staircase or Gainsight, document the correct owner here as an override. Starts empty.

**Numeric ID fallback:** Staircase occasionally returns a numeric owner ID instead of a name in some tenants. If this happens in your environment, build a resolution table here mapping known numeric IDs to real names, following this format:

| Staircase Owner ID | Resolves To | Pod / Manager |
|---|---|---|
| [numeric ID] | [Person Name] | [Pod/Manager] |

Leave this table empty until you encounter real numeric IDs in your own data — do not invent example rows.

**Direct IC attribution:** [List any individual contributors who report directly to the briefing recipient with no pod manager in between, if applicable to your org structure.]

### Gainsight GSID Resolution — Use CRM ID from Staircase Directly

Every Staircase account record contains a **CRM ID** field holding the Gainsight GSID. Always read this field and use it directly for all Gainsight queries. Never call `resolve_customer` by name.

- If `CRM ID` present → use directly
- If `CRM ID` empty or null → classify as "Not found in Gainsight CS — CRM ID missing in Staircase"

### CSM Presence Detection

For every at-risk account, inspect thread participants and classify:

- **CSM present** — no flag needed
- **CSM absent — Renewals present** — note contact(s) by name
- **CSM absent — Sales present** — note sales rep by name
- **CSM absent — Delivery/Onboarding present** — note contact by name; indicates onboarding/implementation touchpoint
- **CSM absent — Partner Success / Training present** — note by name
- **CSM absent — Support present** — flag as support-routed
- **CSM absent — no internal contact detected** — flag as unowned communication

### Email Construction

**⚙️ Admin config — set these to match your organization:**
```
EMAIL_DOMAIN = "example.com"
EMAIL_FORMAT = "firstname.lastname"
```
Format: `firstname.lastname@example.com` (adjust to your actual convention). Hyphenated last names typically use the full hyphenated form (e.g. jane.smith-jones@example.com). Add any individual exceptions (aliases, maiden names, etc.) to the Name Alias table above.

---

### Org Structure

**⚠️ Admin action required: replace this entire section with your real org chart.** The structure below is a placeholder showing the *shape* the briefing expects (a VP/recipient, several pods each with a manager and team members, and possibly some individual contributors with no pod manager) — not real people. Do not deploy this briefing with placeholder names still in place.

**[Recipient Name]** — [Title, e.g. VP, Customer Success]



#### Pod 1: [Pod Name, e.g. "Delivery Success"] — Manager: [Manager Name]

[Team member 1] · [Team member 2] · [Team member 3] ...

*If this pod does not own customer accounts (e.g. an onboarding/delivery team), note that here and exclude it from all Staircase queries — see Query 2a note above.*

#### Pod 2: [Pod Name, e.g. "Strategic Customer Success"] — Manager: [Manager Name]

[Team member 1] · [Team member 2] · [Team member 3] ...

#### Pod 3: [Pod Name, e.g. "Renewals"] — Manager: [Manager Name]

[Team member 1] · [Team member 2] · [Team member 3] ...

#### Pod 4: [Add or remove pods as needed to match your real org]

#### Individual Contributors (direct to [Recipient], no pod manager)

[List anyone who reports directly to the briefing recipient with no intermediate manager, if applicable]

---

## STEP 3 — QUERY GAINSIGHT CS



### CTA Query Rules — Apply to ALL CTA queries

1. Always filter on `IsClosed = false`. Never use `TotalOpenCtas`.
2. Exclude CTAs where Name contains `"PatternBuilder"`.



### Query 3a — Renewal Proximity Cohort

Pull all active non-eDocs accounts renewing within the next **120 days**. Retrieve: account name, GSID, ARR, renewal date, health score, health label, CSM name, open CTA count, open Risk CTA count.



### Query 3b — At-Risk Account Health Lookup

For every account from Step 2 with a risk signal, use the CRM ID to retrieve: health score, health label, health trend, renewal date, ARR, open CTAs, whether open Risk CTA exists.

Step 3a and 3b overlap: use Step 3b result as authoritative. Do not query the same account twice.



---



## STEP 4 — CONFLICT DETECTION



Flag an account as a confirmed conflict if ANY of the following are true:



| Conflict Type | Condition |

|---|---|

| **False Green** | Staircase sentiment ≤ 35 AND Gainsight health = Green |

| **Unworked Risk** | Staircase sentiment ≤ 35 AND no open Risk CTA |

| **Unworked Risk (High Risk)** | Risk Level 5 AND sentiment ≤ 45 AND no open Risk CTA |

| **Churn Risk Flag** | Active Staircase Churn Risk flag — surface unconditionally |

| **Renewal Blind Spot** | Renewal within 90 days AND no thread activity in past 30 days |

| **Health-Sentiment Divergence** | Gainsight health = Red AND Staircase sentiment ≥ 60 |

| **Missing from Gainsight** | CRM ID empty or null in Staircase |



New conflicts (first time this week) → Section 2a.
Continuing conflicts (prior weeks) → Section 3.



---



## STEP 5 — DRAFT THE BRIEFING



Write in an executive narrative voice — direct, specific, actionable.



### TL;DR — Top Risk Priority Order

1. Dark renewal within 7 days — gone dark AND renewal within 7 days AND no recent engagement
2. Imminent renewal with unresolved risk — renewal within 14 days AND active risk AND no close plan
3. Urgent dark renewal — renewal within 30 days AND no engagement in 30+ days
4. New high-ARR risk with no CTA — first time flagged, significant ARR
5. Confirmed churn — only if none of the above apply

**Never select confirmed churn as Top Risk if any actionable at-risk account exists.**

**TL;DR attribution rule:** Verify every name and ARR against the Staircase Owner field before inclusion.



### Section 1 — 📊 What's Happening Across the Segment

Aggregate themes. Max 3 sentences per theme. Customer behavior signals only — never mention Betsy's pod.



### Section 2a — ⚡ New Signal Conflicts This Week

New conflicts only. Opening line: *"The following accounts show material disagreement between Gainsight health data and Staircase relationship signals detected for the first time this week."*

Per conflict: account name ⚡ · ARR · CSM · conflict type · one line · renewal date if within 180 days.



### Section 2b — 🔴 New Risks This Week

One line + one quote per account. No-quote fallback: omit quote line entirely, no placeholder.



### Section 2c — 🤖 Digital CSM Accounts

[Your digital/automated CSM name] accounts with sentiment ≤ 40 or Risk Level ≥ 3. One line each. Omit if none.



### Section 3 — ⚠️ Still Watching

Cap 15 accounts. Tier 1 (renewal within 14 days): full detail. Tier 2: one line each.
**Week 1 rule: omit entirely.**



### Section 4 — 🌑 Accounts Gone Dark

Renewal within 180 days. Max 2 lines per entry.



### Section 5 — 🌱 Expansion Signals

Top 5 by ARR. One line each. Flag at-risk accounts: *"Also flagged at risk — resolve retention first."*



### Section 5b — 👤 CSM Load Flags

Flag CSMs with 3+ qualifying accounts or 2+ confirmed churns. One line per flagged CSM. Omit if none.



### Section 6 — ✅ Questions to Ask This Week

**CSM routing rule:** Questions about pod CSM accounts go to the pod manager. Direct IC questions go directly to the CSM.

**Pre-written email rule:** Every mailto link MUST include a fully pre-written email body. This is not optional. The mailto link is useless without a pre-written body — Marco should be able to click and send with zero editing required.

For each manager or Direct IC email:
- Subject: most critical account + "— and a couple of other things" if multiple items
- Body: friendly one-liner open → one short paragraph per question → "Thanks, Marco."
- Max 6–8 sentences total
- URL-encode the entire subject and body
- The body must contain actual questions tied to specific accounts and signals from this briefing — not placeholder text

**Canvas instruction note:** *💡 Click a manager's name to open a pre-written email ready to send.*

Format:
> **[Name](mailto:email@example.com?subject=URL-encoded-subject&body=URL-encoded-body)** — one-line summary.

**Nothing appears after the footer.** The briefing ends with the footer line. No summaries, no data notes, no observations after it.



### Briefing Footer

```
Generated by Claude + Staircase AI + Gainsight CS · Next scheduled run: Monday, [next Monday's date] at 8:00 AM MT
```



---



## STEP 6 — UPDATE HISTORY FILE



Write to `./history/marco-poggio-v2.json`. If the `./history/` directory does not exist, create it first, then write the file. If the file does not exist, create it. Never skip this step.



```json
{
  "recipient": {
    "name": "[Recipient Name]",
    "slack_user_id": "TBD",
    "title": "VP, Customer Success",
    "managers": ["[Pod Manager 1]","[Pod Manager 2]","[Pod Manager 3]"],
    "direct_ics": ["[Direct IC 1]","[Direct IC 2]"]
  },
  "briefings": [
    {
      "date": "YYYY-MM-DD",
      "week_range": "YYYY-MM-DD to YYYY-MM-DD",
      "delivery_timestamp": null,
      "canvas_url": null,
      "supporting_data_canvas_url": null,
      "themes": [],
      "accounts_flagged": [],
      "accounts_gone_dark": [],
      "expansion_signals": [],
      "relationship_changes": [],
      "signal_conflicts": [],
      "digital_csm_signals": [],
      "csm_load_flags": []
    }
  ]
}
```



---



## STEP 7 — DELIVERY



### Step 7a — Create the Main Briefing Canvas

Using the Slack MCP connector, create a Canvas titled:
`CS Executive Briefing — Week of [Monday's date]`

Include in this exact order: Status Legend, TL;DR block, Section 1, Section 2a, Section 2b, Section 2c (if applicable), Section 3 (if applicable), Section 4, Section 5, Section 5b (if applicable), Section 6, Footer.

After creation, **capture and store the Canvas URL**. Before storing, check if the URL ends with `/full` — if so, remove that suffix. The clean URL format should be `https://your-workspace.slack.com/docs/[ID]` with nothing after the document ID. Do not proceed to Step 7b until this URL is confirmed.



### Step 7b — Send Slack DM to [Recipient Name]

Only execute this step after the Canvas URL from Step 7a is confirmed.

**Week 1 only — send this introduction DM first:**

> Good morning Marco — starting this week, your CS briefing has been upgraded. You'll notice a few new sections: signal conflicts between Gainsight and Staircase (accounts that look healthy in one system but at-risk in the other), CSM load flags, and pre-written emails to your managers ready to send. Everything resets to Week 1 today, so the still watching section will build from this run forward. Same Monday cadence, same Slack delivery — just sharper. Your briefing is below.

**Every week — send this DM to [recipient's email address]:**

> Your weekly CS briefing is ready — [Canvas URL from Step 7a]

After sending, update the history file with the Canvas URL and the DM timestamp.



---



## SCHEDULING



- **Frequency:** Weekly · **Day/Time:** Monday 8:00 AM MT · **Cron:** `0 8 * * 1`
- Pre-approved tool permissions:
  - Staircase AI MCP — read
  - Gainsight CS MCP — read
  - Filesystem — read/write `./history/`
  - Slack MCP — Canvas create + DM send
