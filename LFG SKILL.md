---
name: lfg
description: Session initialization. Load memory, surface active work, surface transcript inbox + open todos + Jira board + Slack + Gmail + Calendar + Friday recap prompt. Detects first-run setups and switches to extended onboarding when appropriate.
---

# LFG — Session Init

Invoked when the user types `lfg`, `let's go`, `start session`, `boot up`, or anything that signals "I am starting a new working session."

## Step 1: Detect first run vs ongoing use

Read `.claude/memory/MEMORY.md`. Count the entries listed.

- If the index has **only one entry** (`user_role.md`) and no other memory files exist → **first run mode** (Step 2A).
- Otherwise → **normal mode** (Step 2B).

## Step 2A: First-run onboarding

This is the user's very first session. Read `.claude/memory/user_role.md` to learn their name and role.

Output something close to this, tailored to their role:

```
Welcome, <first-name>.

I know you as <one-line role summary from user_role.md>. If that's off, tell me and I'll fix it.

Here's what we have:
  - Memory in .claude/memory/ that I'll grow every time we wrap up
  - Eight skills you can invoke: lfg, wrap-up, anti-slop, html-report,
    meeting-notes, pull-transcripts, process-transcripts, weekly-recap
  - Daily rhythm: type "lfg" to start, "wrap up" to close

Three good first moves:
  1. Tell me about one project you're working on. I'll scaffold projects/<name>/.
  2. Set up transcript ingestion (Gemini and/or Granola). See docs/transcript-sources-setup.md.
  3. Paste in a draft and I'll run the anti-slop skill on it.

What would you like to do?
```

If they pick option 1, scaffold the folder. If 2, walk them through the transcript setup doc. If 3, invoke anti-slop.

## Step 2B: Normal-mode ready status

### Historical context (read in parallel)
- Every memory file listed in `MEMORY.md`
- `.claude/project-memory.md` (top 5 entries)
- Any `projects/*/STATUS.md` (up to 5)
- `BACKLOG.md` if it exists

### Local work surfaces
1. **Transcript inbox count**: count files in `projects/transcripts/inbox/gemini/` and `projects/transcripts/inbox/granola/`
2. **Open todos**: read the most recent `projects/todos/YYYY-MM-DD.md`, count `[ ]` items. Also check the prior 2 todo files for carryover open items.
3. **Day of week**: if Friday (or weekend with no recap yet for the week), prep the weekly-recap prompt.

### Live work surfaces (query in parallel, fail gracefully)

All four queries below can fail (MCP not connected, auth expired, no results). If any one fails, skip that section in the output. Never block the ready status on a live query.

4. **Jira board** (see `.claude/memory/reference_jira_board.md` for the canonical URL + JQL):
   - First call `getAccessibleAtlassianResources` to get the cloudId for `tempo-io.atlassian.net`.
   - Then `searchJiraIssuesUsingJql` with `assignee = currentUser() AND statusCategory != Done ORDER BY priority DESC, updated DESC`, limit 10.
   - Surface up to 5 by priority. Show key, summary, status, due date if set.

5. **Gmail** (via Google Workspace MCP, the gmail tools):
   - `search_threads` with `is:unread in:inbox newer_than:2d` for unread recent threads.
   - `search_threads` with `label:starred is:unread` for starred-and-unread (typically her actually-important pile).
   - Surface counts and the top 3 thread subjects + senders.

6. **Google Calendar** (via Google Workspace MCP, the calendar tools):
   - `list_events` for today and tomorrow on her primary calendar.
   - Surface count of meetings today, the next one with start time, and any meetings tomorrow before 10am that she might want to prep for tonight.

7. **Slack** (via Slack MCP):
   - `slack_search_public_and_private` for `@Rhiannon` (or her Slack handle if stored in user_role) in the last 24h.
   - Surface count of mentions and the channels they came from.
   - Don't fetch DM content here, that's a privacy/noise tradeoff. Just flag if there are unread DMs by checking recent activity.

Output a brief ready status. Use voice rules from `.claude/skills/anti-slop/SKILL.md`: no em-dashes (use periods, colons, or parens instead), no banned words, no filler phrases.

```
LFG. Ready to work.

Role: <one-line from user_role.md>

Active:
  <project-name>. Next: <next action from STATUS.md>
  <project-name>. Next: <next action>

Today's calendar: <N meetings>. Next: <title> at <time>.
Tomorrow before 10am: <title at time>, if any.

Jira (assigned, open): <N items>. Top by priority:
  <KEY>: <summary> (<status>, due <date if set>)
  <KEY>: <summary> (<status>)

Gmail: <N unread> in inbox, <M starred unread>. Top:
  <subject> from <sender>
Slack: <N mentions in last 24h> across <channels>.

Transcripts: 3 in inbox (unprocessed). Say "process transcripts".
Todos: 5 open (oldest from 2026-05-08). Say "show my todos".

Feedback applied today:
  <one-line summary of relevant feedback memory>

Backlog: <N open items>

It's Friday. Want a weekly recap? Say "weekly recap".

What are we working on?
```

Drop any section that has nothing to report. Keep the output tight. If a live source errored, omit it silently (don't say "Jira: error") unless the user explicitly asks why a section is missing.

## Rules

- Do not ask clarifying questions before outputting the ready status. Output it, then take the user's next message.
- Do not dump full memory contents into the response. The status is a summary.
- If a project's STATUS.md hasn't been touched in 30+ days, flag it: "Note: <project> hasn't moved in 30+ days, still active?"
- Never invent project names, transcripts, todos, next actions, or Jira/Gmail/Slack items. Only surface what's actually in the files or returned by the live queries.
- For live sources (Jira, Gmail, Calendar, Slack), fail gracefully: if a query errors or returns nothing, drop the section. Don't surface error messages to Rhiannon unless she asks.
- Don't repeat items across surfaces. If a Jira ticket is already in her todo file, only show it once (prefer the todo surface).
- Pre-load voice rules from `.claude/skills/anti-slop/SKILL.md` and apply them to your output.
