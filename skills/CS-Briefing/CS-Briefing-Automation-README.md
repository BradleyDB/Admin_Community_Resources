# CS Executive Briefing Skill

A weekly automated briefing for CS leaders — delivered to Slack every Monday morning via Staircase AI + Claude.

Instead of manually compiling a weekly update, this skill:

1. Queries **Staircase AI** for live signals across the leader's full org
2. Cross-references a **rolling history** of past briefings to track what's new, worsening, or resolved
3. Pulls **verbatim customer quotes** to bring the voice of the customer into the narrative
4. Drafts a **thematic executive summary** — not an account inventory, but a story of what's happening in the business
5. **Delivers automatically to Slack** every Monday morning, timed before weekly manager syncs

---

## Quickstart

### Step 1 — Install requirements

This skill requires two MCP connectors:
- **Staircase AI MCP** — for account signals, health scores, and sentiment data
- **Slack MCP** — for Canvas creation and DM delivery

Connect both in Cowork before running setup.

### Step 2 — Run the setup wizard

In Cowork, type:

```
configure briefing for [Recipient Full Name]
```

The wizard walks you through:
- Recipient details (name, title, Slack email, delivery time, timezone)
- Company email format
- Full org structure — managers, pods, team types, and direct ICs
- Staircase numeric ID map (for accounts created during POC migrations)
- Name aliases (for CSMs who've changed names)

It creates two files automatically:
- `./config/[first-name]-[last-name].json` — configuration
- `./history/[first-name]-[last-name].json` — history (starts empty, Week 1 ready)

No manual file editing or `[CONFIGURE]` placeholder replacement required.

### Step 3 — Test it

After setup, run the briefing once manually to validate output and pre-approve tool permissions:

```
run briefing for [Recipient Full Name]
```

### Step 4 — Schedule it

In Cowork, create a Scheduled Task:
- **Frequency:** Weekly, every Monday
- **Time:** 8:00 AM recipient local time (or whatever you set during setup)
- **Task prompt:** `run briefing for [Recipient Full Name]`
- **Enable "Keep awake"** for reliable unattended execution

Pre-approve these tool permissions when prompted:
- Staircase AI MCP — read access
- Filesystem — read/write to `./history/` and `./config/`
- Slack MCP — Canvas create, DM send

---

## What the Briefing Contains

Each briefing delivers seven components, in order:

**⚡ 3 Things You Need to Know This Week**
TL;DR at the top — top risk, top opportunity, and one critical question for the week.

**📊 What's Happening Across the Segment**
Aggregated themes across the full book. Patterns in customer conversations, product friction, and sentiment trends — not account-by-account.

**🔴 New Risks This Week**
Only accounts where something materially changed. Includes ARR, health score, sentiment score, CSM name, manager, CSM presence detection, and a verbatim customer quote.

**⚠️ Still Watching** *(Week 2 onwards)*
Previously flagged accounts with no resolution. Notes how many weeks each issue has been open and whether it's improving, worsening, or static.

**🌑 Accounts Gone Dark**
Accounts with no customer-initiated engagement for 2+ weeks. Surfaced as early churn warnings.

**🌱 Expansion Signals**
New locations, upsell interest, NPS promoters, positive sentiment spikes. Flags if a signal has been sitting unactioned from a prior week.

**✅ Questions to Ask This Week**
Specific questions to raise with each manager, tied to named accounts. Each manager gets a single pre-written email ready to send in one click.

---

## Status Indicators

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

For every at-risk account, the skill inspects Staircase thread participants and classifies who is actually communicating on the account — not just who owns it:

- **CSM present** — account owner is active in communications
- **CSM absent — Renewals present** — Renewals team is handling without CSM involvement
- **CSM absent — Sales present** — Sales rep is on thread without CSM
- **CSM absent — Support present** — support@ email or support team is the only internal contact
- **CSM absent — Partner Success / Training present** — training or enablement team is on thread
- **CSM absent — no internal contact** — no internal participants visible

---

## History and Continuity

Each recipient has a JSON history file at `./history/[first-name]-[last-name].json` that stores every past briefing. Before drafting, the skill reads this file to understand:

- Which accounts were previously flagged and are still unresolved
- How many weeks each issue has been open
- What's improved since last week
- What was resolved and can be closed out

This prevents the briefing from re-presenting known issues as new surprises, and enables continuity language like:
- *"Flagged for the third consecutive week — still no re-engagement plan confirmed."*
- *"Sentiment improved from 10 → 34 since last week's flag — recovery call appears to be working."*

---

## Adding a Second Recipient

Run setup again for each additional recipient. Each gets their own config and history file. All scheduled tasks run independently.

```
configure briefing for [Second Recipient Full Name]
```

---

## Checking Status

To see all configured recipients and last delivery dates:

```
briefing status
```

---

## Tech Stack

| Component | Tool |
|---|---|
| Relationship signals and account health | Staircase AI MCP |
| Customer quote retrieval | Staircase AI `staircase_fetch_evidence` |
| Briefing generation | Claude (via Cowork) |
| Delivery | Slack Canvas + DM (via Slack MCP) |
| Scheduling | Cowork scheduled tasks |
| Configuration | JSON per recipient in `./config/` |
| History store | JSON per recipient in `./history/` |

---

## Known Limitations

**Staircase owner ID resolution**
Staircase sometimes returns a numeric owner ID for accounts created during a POC migration. The skill resolves these using the ID map you configure during setup. Any unresolved IDs are flagged explicitly rather than passed through as raw numbers.

**Duplicate IDs from POC migrations**
If your org ran a Staircase POC before going live, some users may have two IDs. Add any known mappings during setup. Flag unresolved duplicates to Staircase support for cleanup.

**Slack Canvas mailto links**
The "Questions to Ask" section uses mailto links with pre-written email bodies. These work correctly in Slack Canvas — click a manager's name to open a pre-written email ready to send.

**Microsoft 365 delivery**
The M365 MCP connector currently supports read-only operations. Email send and OneDrive write require `Mail.Send` and `Files.ReadWrite` OAuth scopes added to the connector's Azure AD app registration. Until those are granted, Slack Canvas is the recommended delivery method.

**Local scheduling dependency**
Scheduled tasks in Cowork require the local machine to be awake with "Keep awake" enabled. Cloud-based scheduling would remove this dependency and is listed as a future extension.

---

## Ideas for Extension

- **Manager-level cascade** — generate a scoped version for each manager showing only their pod's signals
- **Calendar integration** — auto-time delivery before the exec's weekly manager sync (requires Google Calendar or Outlook MCP)
- **Salesforce enrichment** — pull ARR and renewal pipeline data from SFDC for cleaner commercial figures
- **Renewal horizon view** — add a 30/60/90-day forward-looking renewal section
- **Action tracking** — integrate with Asana or Jira so suggested actions become trackable tasks
- **VP roll-up** — aggregate across multiple director-level briefings for an exec-level view
- **Cloud scheduling** — remove the "machine must be awake" dependency with cloud-based execution

---

*Built with [Claude](https://claude.ai) + [Staircase AI](https://staircase.ai). Requires Staircase AI MCP and Slack MCP connectors. Compatible with Claude Cowork.*
