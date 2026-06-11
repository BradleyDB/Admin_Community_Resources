# Dry-Run Validation Checklist

Use during setup (Step 6) and any time output quality is in question. Walk through each item with the user against the preview output. All items must pass before scheduling.

## Structure

- [ ] Title, Status Legend, and ⚡ TL;DR block appear in that order, before Section 1
- [ ] TL;DR has exactly three entries: Top Risk, Top Opportunity, Critical Question — one sentence each, with named accounts/managers and ARR where applicable
- [ ] All sections present in order (1, 2, 4, 5, 6 on Week 1; 1–6 from Week 2)
- [ ] **Week 1 only:** Section 3 (Still Watching) is completely absent — no header, no placeholder, no mention of continuity
- [ ] Footer present and correctly states next run day/time/timezone

## Content rules

- [ ] Section 1 is thematic — no account-by-account listing; max 3 sentences per theme
- [ ] Section 2 contains only accounts that materially changed this week
- [ ] Every Section 2 account has: status emoji, CSM + manager, ARR, health, sentiment, one-line change summary, presence classification, at most one quote
- [ ] Status emojis follow the criteria (explicit intent to leave = 🔴 regardless of scores; renewal signed with open blockers = 🟠, not 🟢)
- [ ] Quotes express emotional state or stated intent — nothing procedural
- [ ] Section 4 entries are max 2 lines; Section 5 entries are one line
- [ ] Section 6: max 2 sentences per question; friendly, curious tone

## Attribution

- [ ] No raw numeric Staircase IDs anywhere in the output (unresolved IDs appear as "Owner ID {n} — not in known ID map, manual lookup required")
- [ ] Every account's CSM and manager match the org tree in config; aliases resolved to one person
- [ ] Direct ICs attributed as "Direct IC — {Recipient}", not to a pod manager
- [ ] Excluded people do not appear anywhere
- [ ] Presence classifications name specific people and roles, not just "CSM absent"

## Email & delivery

- [ ] Exactly one mailto link per manager, consolidating all of that manager's questions
- [ ] Email addresses follow the org format, honoring exceptions and hyphenated names
- [ ] Email bodies: friendly open, one short paragraph per question, sign-off with recipient's first name, 6–8 sentences max
- [ ] Canvas delivery includes the 💡 click-instruction line; DM-text delivery omits it

## State (verify after first scheduled run, not the dry run)

- [ ] Preview/dry runs wrote **nothing** to the history file
- [ ] After the first scheduled run: one new history entry with `delivery_timestamp` and `canvas_url` populated
