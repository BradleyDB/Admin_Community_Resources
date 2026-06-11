# File Schemas

Three files, all in the working folder. Config is static (edited by setup mode or by hand); history is mutable state (written only by scheduled runs, only after successful delivery). Never mix the two.

## 1. `./recipients/_org.json` — org-wide constants

```json
{
  "company_name": "Acme Corp",
  "email_format": "firstname.lastname@acme.com",
  "email_exceptions": [
    { "name": "Sam Lee-Osei", "email": "sam.lo@acme.com" }
  ],
  "support_email": "support@acme.com",
  "ticketing_system": "Salesforce"
}
```

## 2. `./recipients/{first-last}.json` — per-recipient config

```json
{
  "recipient": {
    "name": "Dana Whitfield",
    "first_name": "Dana",
    "title": "VP, Customer Success",
    "slack_email": "dana.whitfield@acme.com",
    "timezone": "America/Chicago"
  },
  "delivery": {
    "method": "slack_canvas_dm",
    "day": "Monday",
    "time": "08:00"
  },
  "history_lookback_weeks": 12,
  "pods": [
    {
      "name": "Strategic Customer Success",
      "manager": "Priya Nair",
      "team_type": "csm",
      "members": [
        { "name": "Jordan Fox", "title": "Sr. CSM", "location": "Austin, TX" }
      ]
    },
    {
      "name": "Renewals",
      "manager": "Chris Okafor",
      "team_type": "renewals",
      "members": []
    }
  ],
  "direct_ics": [
    { "name": "Lena Park", "title": "Principal CSM", "location": "Remote" }
  ],
  "exclusions": [
    { "name": "Morgan Diaz", "reason": "CS Operations — no customer-facing accounts; do not query or report" }
  ],
  "id_map": [
    { "staircase_id": "48211", "name": "Jordan Fox", "pod": "Strategic Customer Success" }
  ],
  "aliases": [
    { "former_name": "Jordan Smith", "current_name": "Jordan Fox", "email": "jordan.fox@acme.com", "pod": "Strategic Customer Success" }
  ]
}
```

Notes:
- `delivery.method`: `slack_canvas_dm` (default) or `slack_dm_text`. Email delivery is intentionally **not** a supported value — see `troubleshooting.md` (M365 scopes).
- `team_type`: `csm` | `renewals` | `sales` | `support` | `partner_success`. Drives presence classification — `support` and `partner_success` pods never own accounts.
- `history_lookback_weeks`: how many recent briefings a run loads for continuity. Default 12; set during setup.
- Optional sections (`direct_ics`, `exclusions`, `id_map`, `aliases`, non-CSM pods) may be empty arrays or omitted.

## 3. `./history/{first-last}.json` — briefing history

```json
{
  "recipient": {
    "name": "Dana Whitfield",
    "slack_email": "dana.whitfield@acme.com",
    "title": "VP, Customer Success",
    "managers": ["Priya Nair", "Chris Okafor"],
    "direct_ics": ["Lena Park"]
  },
  "briefings": [
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
  ]
}
```

Notes:
- The file keeps **every** briefing ever written (no pruning); runs read only the last `history_lookback_weeks` entries. `first_flagged` preserves true issue age even beyond the read window.
- `delivery_timestamp` and `canvas_url` are filled in when delivery succeeds — a `null` timestamp on the latest entry means something went wrong and the entry shouldn't exist (history is only written post-delivery).
- Setup mode initializes the file with the `recipient` block and `"briefings": []`. An empty array = Week 1 semantics.
