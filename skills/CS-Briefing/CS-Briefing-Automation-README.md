# Automated Weekly CS Executive Briefing with Claude + Staircase AI

A lightweight automation that generates contextually-aware, data-driven weekly briefings for CS leaders — automatically delivered to Slack every Monday morning.

---

## What It Does

Instead of a CSM ops person manually compiling a weekly update, this system:

1. Queries **Staircase AI** for live signals across a VP or Director's full org
2. Cross-references against a **rolling history** of past briefings to track what's new, what's worsening, and what's been resolved
3. Pulls **verbatim customer quotes** to bring the voice of the customer into the briefing
4. Drafts a **thematic executive summary** — not an account inventory, but a narrative of what's happening in the business
5. **Delivers automatically to Slack** every Monday morning, timed before weekly manager syncs

---

## The Problem It Solves

Most CS leaders get reports that are either:
- Too tactical (a spreadsheet of health scores with no narrative)
- Too stale (manually compiled, so signals are days old by the time they arrive)
- Too repetitive (same accounts flagged week after week with no "what changed")

This briefing answers: *"What do I need to know right now, and what questions should I be asking my managers this week?"*

---

## Briefing Structure

Each briefing contains the following sections:

**⚡ 3 Things You Need to Know This Week**
A TL;DR at the very top — one top risk, one top opportunity, and one critical question for the week. Written for an exec who has 60 seconds before their week starts.

**📊 What's Happening Across the Segment**
Aggregated themes across the whole book — not account by account. What patterns are showing up in customer conversations? What product areas are generating friction or momentum this week?

**🔴 New Risks This Week**
Only accounts where something materially changed this week. Each entry includes ARR, health score, sentiment score, CSM name, manager attribution, CSM presence detection (who is actually on the thread), and one verbatim customer quote — selected for emotional impact, not procedural detail.

**⚠️ Still Watching** *(Week 2 onwards)*
Previously flagged accounts with no resolution. Language explicitly notes how many weeks the issue has been open and whether it's improving, worsening, or static.

**🌑 Accounts Gone Dark**
Accounts with no customer-initiated engagement for 2+ weeks. Surfaced as early churn warnings before they become active churn cases.

**🌱 Expansion Signals**
Positive signals: new locations, upsell interest, NPS promoters, positive sentiment spikes. Notes if a signal has been sitting unactioned from a prior week.

**✅ Questions to Ask This Week**
Specific, grounded questions the leader should bring into each manager's week — attributed by manager name, tied to specific accounts or signals, with a pre-written email ready to send in one click.

---

## Status Indicators

Each at-risk account in the briefing is tagged with a color-coded emoji:

| Emoji | Status | When Applied |
|---|---|---|
| 🔴 | Confirmed Churn / Churning | Formal non-renewal submitted or explicit intent to leave stated |
| 🟠 | At Risk | Active escalation, evaluation of alternatives, or renewal within 90 days with unresolved blockers |
| 🟡 | Watching | Previously flagged, no new deterioration, static signal |
| 🟢 | Recovering | Renewal signed AND no material blockers remaining |
| 🌑 | Gone Dark | No customer-initiated engagement for 2+ weeks |
| 🌱 | Expansion Signal | Positive upsell, expansion, or NPS signal |

---

## CSM Presence Detection

For every at-risk account, the system inspects thread participants in Staircase and classifies who is actually communicating on the account — not just who owns it. Classifications include:

- **CSM present** — account owner is active in communications
- **CSM absent — Renewals present** — Renewals team is handling without CSM involvement
- **CSM absent — Sales present** — Sales rep is on thread without CSM
- **CSM absent — Support present** — Support team or support@ email address is the only internal contact
- **CSM absent — Partner Success / Training present** — Training or enablement team is on thread
- **CSM absent — no internal contact** — No internal participants visible

This surfaces situations where a $500K+ account in active churn is being handled entirely by Renewals with no CSM in the conversation — a pattern that's easy to miss without this kind of thread-level inspection.

---

## Voice of the Customer

For every at-risk account, the system attempts to retrieve a verbatim customer quote from recent Staircase communications. Quote selection prioritizes language that expresses emotional state or stated intent — not procedural or logistical detail.

A customer saying *"I do not trust you"* or *"we will evaluate continuity of the service"* hits differently than "Sentiment Score: 7."

---

## History & Continuity

Each recipient has a JSON history file that stores every past briefing. Before drafting a new briefing, the system reads this file to understand:

- Which accounts were previously flagged and are still unresolved
- How many weeks each issue has been open
- What's improved since last week
- What was resolved and can be closed out

This prevents the briefing from re-presenting known issues as new surprises, and enables continuity language like:
- *"Flagged for the third consecutive week — still no re-engagement plan confirmed."*
- *"Sentiment improved from 10 → 34 since last week's flag — recovery call appears to be working."*

History files are stored as JSON at `./history/{first-name}-{last-name}.json` in the Cowork working folder.

---

## Org Hierarchy Awareness

The system respects the reporting structure of the recipient. It:

- Pulls signals from all CSMs across the recipient's full org tree
- Attributes each account to the correct CSM **and** their manager
- Frames weekly questions to the right person
- Handles Individual Contributors (ICs) who report directly to the exec with no pod manager in the chain
- Supports multiple team types: CSM pods, Renewals teams, Support orgs, Partner Success / Training teams — each classified correctly when they appear in Staircase threads

---

## Known Limitations & Workarounds

**Staircase owner ID resolution**
Staircase sometimes returns a numeric owner ID instead of a name — particularly for accounts created during a POC or data migration. The prompt includes a configurable ID map to resolve known IDs to named CSMs. Any unresolved IDs are flagged explicitly rather than passed through as raw numbers.

**Duplicate IDs from POC migrations**
If your org ran a Staircase POC before going live, some users may have two IDs — one from the POC import and one from production. The prompt handles ambiguous IDs by checking email signatures in the account thread to resolve attribution. Flag duplicates to Staircase support for cleanup.

**Slack Canvas mailto links**
The briefing's "Questions to Ask This Week" section uses mailto links with pre-written email bodies. These work correctly in Slack Canvas — click the manager's name to open a pre-written email ready to send.

**Microsoft 365 delivery**
The M365 MCP connector currently supports read-only operations. Email send and OneDrive write require `Mail.Send` and `Files.ReadWrite` OAuth scopes, which must be added to the connector's Azure AD app registration by your IT team. Until those are granted, Slack Canvas is the recommended delivery method.

---

## Tech Stack

| Component | Tool |
|---|---|
| Relationship signals & account health | Staircase AI (via MCP) |
| Customer quote retrieval | Staircase AI `staircase_fetch_evidence` |
| Briefing generation | Claude (via Cowork) |
| Delivery | Slack Canvas + DM (via Slack MCP) |
| Scheduling | Claude Cowork scheduled tasks |
| History store | JSON file per recipient (local filesystem) |

---

## Setting It Up for a New Recipient

1. **Map their org** — identify their direct manager-level reports, the CSMs under each, and any Individual Contributors reporting directly to the exec
2. **Identify team types** — note which pods are CSM-facing, which are Renewals, Support, Partner Success, etc.
3. **Create a history file** — initialize an empty `{first-name}-{last-name}.json` in the history folder (see schema in the prompt template)
4. **Resolve known Staircase IDs** — if your org has had a POC migration, identify any numeric IDs that may appear and add them to the ID map
5. **Configure the prompt** — fill in the recipient's name, Slack email, org structure, and team type classifications
6. **Run once manually** to validate output and pre-approve tool permissions
7. **Create a scheduled task** in Cowork — weekly, Monday at 8AM recipient local time
8. **Enable Keep awake** in Cowork scheduled tasks so the task fires reliably

---

## History File Schema

```json
{
  "recipient": {
    "name": "...",
    "slack_email": "...",
    "title": "...",
    "managers": ["..."],
    "direct_ics": ["..."]
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

## What's Next / Ideas to Extend

- **Manager-level cascade** — generate a scoped version for each manager that only shows their pod's signals
- **Calendar integration** — auto-time delivery before the exec's weekly manager sync (requires Google Calendar or Outlook MCP)
- **Salesforce enrichment** — pull ARR and renewal pipeline data from SFDC for cleaner commercial figures
- **Renewal horizon view** — add a 30/60/90-day forward-looking renewal section
- **Action tracking** — integrate with Asana or Jira so suggested actions become trackable tasks
- **VP roll-up** — aggregate across multiple director-level briefings for an exec-level view
- **Cloud scheduling** — currently requires local machine to be awake; cloud-based execution would remove this dependency

---

*Built with [Claude](https://claude.ai) + [Staircase AI](https://staircase.ai). Requires Staircase AI MCP connector and Slack MCP connector. Tested on Claude Cowork.*
