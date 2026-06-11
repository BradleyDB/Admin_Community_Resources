# Owner Attribution & CSM Presence Detection

Two distinct jobs: (1) figure out **who owns** each account, (2) figure out **who is actually communicating** on it. The gap between the two is the point — a $500K account in active churn handled entirely by Renewals with no CSM on the thread is exactly the pattern this exists to surface.

## CSM attribution (who owns it)

The CSM is the **Owner field** returned by Staircase on the account record. If an Owner name is present, use it — never report attribution as unknown when Staircase provides a name.

Resolution order:

1. **Owner name present** → cross-reference the recipient's org tree (config) for the manager. If the name isn't in the org tree, keep the name and flag "not in known org tree" — never leave the CSM field blank.
2. **Alias check** → if config lists name aliases (e.g., name changes after marriage), treat former and current names as one person and attribute to the mapped pod.
3. **Numeric ID** → Staircase sometimes returns a numeric owner ID instead of a name (common for accounts created during a POC or data migration). Resolve via the `id_map` in config. If the ID isn't in the map, output: *"Owner ID {number} — not in known ID map, manual lookup required."* **Never pass a raw numeric ID through to the briefing as a CSM name.**
4. **Direct ICs** → if the owner is listed in config as reporting directly to the recipient, attribute as "Direct IC — {Recipient Name}" (no pod manager exists; their questions in Section 6 go to the recipient's own list).
5. **Exclusions** → people marked excluded in config (ops roles with no accounts) should never be queried or appear in output.

## CSM presence detection (who's actually on the thread)

For every at-risk account, inspect the Staircase thread participants and classify:

| Classification | Criteria | Output guidance |
|---|---|---|
| **CSM present** | Account owner appears in recent thread activity | No flag needed |
| **CSM absent — Renewals present** | Owner absent; member(s) of a `renewals`-type pod on thread | Name the contact(s) and title(s) |
| **CSM absent — Sales present** | Owner absent; Sales Rep listed in Staircase's "Sales Rep Name" field | Name the rep |
| **CSM absent — Renewals and Sales present** | Both, no CSM | Name everyone with roles |
| **CSM absent — Support present** | A `support`-org member on thread, or the org's support address (from `_org.json`) is a participant | Name if identifiable, else "Support contact ({support org})"; flag support@ appearances explicitly as support-routed |
| **CSM absent — Partner Success / Training present** | A `partner_success`-type team member on thread | Name and title; this team never owns accounts — their presence means a training/enablement touchpoint, classify accordingly |
| **CSM absent — no internal contact detected** | No internal participants visible | Flag as unowned communication |

Team types come from the pod definitions in the recipient config — don't guess a person's team from their title alone when config already classifies them.
