---
name: grill-me
description: Interview the user relentlessly about a plan, design, or topic, checkpointing every answer to a durable capture file so nothing is lost. Use when the user wants to stress-test or pressure-test a plan, get grilled on a design, run a brainstorm, discovery, or requirements-gathering session, do a design review, extract what's in their head into a doc, dump their brain, walk through their thinking, or says "grill me" or "interview me".
---

# Grill Me

Relentlessly interview the user about every aspect of the topic until you reach shared understanding. Walk down each branch of the decision tree, resolving dependencies one by one. The real goal is to **extract what's in their head into a durable, organized markdown capture so nothing is lost as context fills up.**

## The capture is the whole point

Long interviews fill context. If you hold answers only in your head, you will eventually misremember, conflate, or drop something. So you **checkpoint after every single answer**. The capture, not your context, is the source of truth. Never make the user ask you to save progress.

## Setup — pick the environment first

Before anything else, determine where you are running. Each environment has different persistence rules.

### Claude Code (filesystem + Bash)
- Capture path: `./brainstorms/{YYYY-MM-DD}-{topic-slug}.md` relative to cwd.
- Create the `brainstorms/` folder if missing.
- Get the date with `date +%F`.

### Cowork (filesystem, user-selected folder)
- Capture path: `{user-selected-folder}/brainstorms/{YYYY-MM-DD}-{topic-slug}.md`.
- If no folder is selected, fall back to the outputs scratchpad and warn the user that the capture will not persist after the session unless they save it.
- Get the date from the conversation `<env>` block (or `date +%F` via Bash if available).

### Claude.ai chat (no filesystem)
- Create a single markdown **artifact** at session start. Update it in place after every answer.
- Tell the user they can download the artifact at the end.
- Get the date from conversation context.
- Note: resume across sessions is not available in chat — only Code and Cowork support that.

## Resume check (Code and Cowork only)

Before creating a new capture, glob `brainstorms/*{topic-slug}*.md`. If a match exists:
1. Surface the file path and a one-line summary.
2. Offer three options: **resume** (continue appending Q&A in the same file), **new section** (add a dated section to the existing file), or **fresh start** (create a new file with a `-v2` suffix).
3. Never overwrite silently.

## Initial setup (before the first question)

1. Create or open the capture per the rules above.
2. Write a header: title, date, the goal of the session, and empty "Open flags" and "Next Steps" sections.
3. Tell the user where you are saving in one line. Then ask Q1.

## The checkpoint rule (non-negotiable)

After EVERY user answer, BEFORE you ask the next question:
- Append a structured entry to the capture: the question topic, the key facts and decisions from their answer (in their words where the wording matters), and any flags (things they could not answer plus who should).
- Update or correct earlier entries if a later answer changes them.
- **Never overwrite the file silently — always append or edit in place.**
- Only then ask the next question.

Never batch multiple answers into one write. Checkpoint one answer at a time. If context is lost at any moment, the capture already holds everything said so far.

## Interview method

- Ask **one question at a time**. For each, provide your **recommended answer** (your best inference from context) so the user can simply confirm, correct, or redirect.
- Resolve dependencies in order: settle the upstream decision before the ones that depend on it.
- If a question can be answered by **exploring the codebase or reading a file/doc**, do that instead of asking. If the user hands you a doc, read it and only surface what's net-new.
- When the user **can't answer** something, capture it as a flag with the right owner and move on. Don't stall.
- Keep going until the user says you're done, or you've covered every branch. Offer a completeness backstop near the end ("anything we haven't touched?").

## Capture file structure

```
# {Topic}: Brainstorm / Discovery Notes
Date: {date} · Goal: {one line}

## Summary / key decisions
(running synthesis, updated as you go)

## Q&A log
### Q1 — {topic}
- Asked: {question}
- Captured: {facts, decisions, in their words where it matters}
- Flags: {open item -> owner}
...

## Open flags (pending input)
| Item | Owner | Notes |
| --- | --- | --- |

## Next Steps
| Action | Owner | Due | Source Q# |
| --- | --- | --- | --- |
```

## At the end

1. Do a final read of the capture for contradictions or gaps and reconcile them.
2. Populate the **Next Steps** table — every decision that implies an action should have an owner and a due date (or `TBD` flagged for follow-up).
3. Give the user a short recap: what's captured, what's still flagged, and the top 1–3 next steps.
4. In chat, remind them to download the artifact. In Code/Cowork, surface the file path.
