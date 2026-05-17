---
name: td-plan
description: Use when the user invokes /td-plan or asks to plan a day in detail (tomorrow, today, or any specific day). A 10-15 min structured walk-through that locks in an anchor task, supporting cast, client commitments, time blocks, and a mental rehearsal — then saves a plan file the morning ritual can read.
---

# /td-plan

A 10-15 minute structured planning session that walks the user through their day in detail. Defaults to **tomorrow** but accepts any day.

**Invocation:** `/td-plan` (plans tomorrow), or with a target day mentioned in the user's message: "plan today", "plan Friday", "plan 2026-05-20", "plan next Monday".

**Best runtime: Claude Desktop** — has both Todoist MCP and Google Calendar MCP. In Claude Code (CLI mode, no Calendar), the calendar step is skipped gracefully.

## Prelude

1. Resolve this skill's directory. Bundle root is two levels up.
2. Read `<bundle>/skills/td-gtd/SKILL.md`. Internalize system map, filters, labels, growth contract.
3. Read `<bundle>/skills/_shared/environment.md`. Run detection. Halt if no Todoist runtime. If MCP-unauthed and user declines auth → halt.
4. **Calendar detection** (separate from Todoist detection): scan visible tools for any name matching `Google_Calendar` OR containing `*Calendar*list_events*`. If found → calendar mode ON. Else → calendar mode OFF (degrade gracefully; skip the calendar pull, ask the user to type any fixed meetings instead).
5. Proceed.

## Target day resolution

Parse the user's message for a day reference. Resolve to a `YYYY-MM-DD` date (local time):

- No day mentioned → tomorrow.
- "today" / "for today" → today.
- "tomorrow" → tomorrow.
- A weekday ("Friday", "next Monday") → the nearest upcoming match (today counts if it matches and current time is before noon, else next week).
- A date ("2026-05-20", "May 20", "20th") → that date.
- Ambiguous → ask once: "Plan for tomorrow (2026-05-18) by default — different day?"

Compute and bind:

```
TARGET_DATE = the resolved YYYY-MM-DD (local time)
TARGET_LABEL = a human label like "tomorrow (Friday, 2026-05-18)" or "today (2026-05-17)"
```

Use both throughout the ritual.

## Flow

### Step 1: Frame the day

> Planning for **[TARGET_LABEL]**. What kind of day is it — heads-down / meeting-heavy / recovery / travel / mixed?

Record the day-type. It informs the time-block proposal in Step 7.

### Step 2: Pull state

In **CLI mode (Claude Code)**:

```bash
td filter view "🎯 Today" --json
td filter view "📅 This week" --json
td task list --project "id:6gg3XRqxXPWPMwH7" --json   # All of LangRelay; filter client-side by section_id
```

In **MCP mode (Claude Desktop)**: equivalent operations via the Todoist MCP tools. Warn upfront if it's the slower runtime.

Look up LangRelay's `🔥 In Progress` section_id from `~/dev/skills/todoist-skills/section-ids.json` and filter the LangRelay task list to surface in-progress items.

Also identify tasks already dated for `TARGET_DATE` in the JSON — those are pre-existing commitments to honor.

### Step 3: Pull the calendar (calendar mode only)

If calendar mode is ON:

- Call the calendar `list_events` tool for `TARGET_DATE` (00:00 to 23:59 in user's local timezone — CET/CEST).
- Extract: start time, end time, title for each event.
- Sort chronologically.

If calendar mode is OFF:

> Calendar isn't available in this runtime. Any fixed meetings or calls on [TARGET_LABEL]? Format like "09:30-10:00 standup, 14:00-15:30 Hairgivers call", or "none".

Record the fixed events list.

### Step 4: The anchor

The most important step. Single most important task for [TARGET_LABEL].

> What's the **ONE thing** that, if it ships [TARGET_LABEL], makes the day successful?

Help the user pick. Surface candidates in this priority order (the growth-contract default):

1. **LangRelay** — `🔥 In Progress` tasks first, then `This Week`. This is the primary growth bet — bias toward it.
2. **Selektable** — user's company, second priority for own-work.
3. **Side Projects** (Parsew, Kopplio, EmitKit, etc.) — third own-work tier.
4. **Clients** (Leat, Hairgivers, Mediastap) — high external pressure but lower long-term equity.
5. **🔧 Admin / 🏠 Personal** — only if something time-critical.

If the user picks a non-LangRelay anchor when LangRelay has actionable items: note gently — "LangRelay has [N] in-progress and [M] in This Week. Sure you want [other] as anchor instead?" — accept their answer, don't push.

Once chosen, **set the anchor in Todoist**:

```bash
td task update "id:<anchor_task_id>" --due "<TARGET_DATE>" --priority p1 --labels "deep_work,<existing labels>"
td task move "id:<anchor_task_id>" --section "id:<This_Week_section_id_for_that_project>"
```

(Section ID looked up from `section-ids.json`.)

### Step 5: The supporting cast

> Two more tasks you'd like to ship on [TARGET_LABEL].

Same priority order as the anchor (LangRelay → Selektable → ...). Don't require deep_work labels — supporting tasks can be `@quick`, `@email`, `@calls`, whatever's honest.

Set each in Todoist: `--due "<TARGET_DATE>"`, appropriate priority (p2 default), preserve existing labels.

If the user struggles to pick two: that's a signal. Ask: "Is [TARGET_LABEL] mostly going to be reactive client work + meetings? In that case, just one supporting task or skip — better to honor reality than overload."

### Step 6: Client commitments walk

For each active client, pull tasks:

```bash
td task list --project "id:6gg3XVMc99VMGxMV" --json   # Leat
td task list --project "id:6gg3XVq73jRXwxHR" --json   # Hairgivers
td task list --project "id:6gg3XVQrrCgRV9Qv" --json   # Mediastap
```

For each, identify:
- Tasks already dated for `TARGET_DATE`
- Tasks overdue as of `TARGET_DATE` (`due.date < TARGET_DATE`)
- Tasks in `This Week` that might warrant promotion to `TARGET_DATE`

Present concisely, per-client: "Leat — 1 task already dated for [TARGET_LABEL] (Vendomat follow-up), 0 overdue, 3 in This Week. Promote any?"

The user confirms / adjusts dates. Don't over-prompt — if a client has 0 commitments for the target, just say "Leat — nothing for [TARGET_LABEL]" and move on.

### Step 7: Time-block sketch

Compose a time-blocked agenda. Inputs:
- Fixed events (from Step 3): non-negotiable, exact times.
- Anchor (Step 4): needs a 90-min deep_work block.
- Supporting tasks (Step 5): each gets ~30-60 min depending on label.
- Client commitments (Step 6): each gets a slot, sized by judgment.
- Day-type (Step 1): meeting-heavy days have less work-block room; heads-down days have more.

Propose a sketch and let the user adjust. Sample for a "heads-down" day with one meeting:

```
08:00–09:30  Deep work — [Anchor title]
09:30–10:00  Standup (fixed)
10:00–12:00  Supporting tasks: [Task 2], [Task 3]
12:00–13:00  Lunch (hard break)
13:00–14:00  Client: [Leat — Vendomat]
14:00–15:30  Quick wins window + email
15:30–17:00  Buffer / wrap
```

For meeting-heavy days, shrink the deep_work block to 60 min or move it before the first meeting. For recovery days, propose 1 task + lots of buffer.

This sketch is a **suggestion**, not enforced. The user adjusts; the skill writes whatever they end up with.

### Step 8: Quick-wins reserve

```bash
td filter view "⚡ Quick wins" --json
```

Pick 3-5 candidates. These don't get dated for `TARGET_DATE` (they live in the reserve, not the locked plan). Just list them so they're visible.

### Step 9: Mental rehearsal

Verbalize the day forward to the user. Concrete, time-anchored, second-person:

> When you sit down at 08:00, you open [anchor] and ship it before standup. Right after standup at 10:00, you do [task 2]. Lunch is a hard break — close the laptop. After [meeting] at 15:30, you triage email and clear two quick wins, then close down by 17:00.

The rehearsal is short — 3-5 sentences. Its job is mental priming, not narration of every minute.

### Step 10: Save the plan

Write to `~/.local/share/todoist-skills/plans/<TARGET_DATE>.md`. Format:

```markdown
# Plan for <TARGET_DATE> (<weekday>) — <day-type>

_Created: <YYYY-MM-DD HH:MM> (local time)_

## Anchor
- **<anchor title>** — <project> / <section> / @deep_work / p1
  Todoist id: <task_id>

## Supporting
- <task 2 title> — <project> / <section> / <labels> / <priority>
- <task 3 title> — <project> / <section> / <labels> / <priority>

## Client commitments
- <client>: <task title> — <due> / <labels>
- (or "No client commitments for this day" if zero)

## Calendar fixed points
- HH:MM–HH:MM  <event title>
- (or "None" / "Calendar not available" if applicable)

## Time blocks
- HH:MM–HH:MM  <block description>
- HH:MM–HH:MM  <block description>

## Quick wins reserve
- <task> — <project> / <labels>
- <task> — <project> / <labels>

## Mental rehearsal
<3-5 sentence rehearsal text>
```

**Concurrency rule**: if a file already exists at that path (e.g., user re-runs `/td-plan` for the same day to adjust), **overwrite it entirely**. There's no canonical-line preservation here — the file is a single coherent plan, not an append log. The user is intentionally replacing the prior plan.

### Step 11: Output a 4-line summary

```
Plan for <TARGET_LABEL> saved.
Anchor: <anchor title>.
Supporting: <task 2>, <task 3>.
Plan file: ~/.local/share/todoist-skills/plans/<TARGET_DATE>.md
```

Skill exits.

## Failure modes

- **Target day already has many dated tasks** (>5) — the planning impulse may be misguided; surface this before Step 4: "[TARGET_LABEL] already has N tasks dated. Want to triage what's there before adding more, or replan from scratch?"
- **User says all kinds of "I don't know"** — accept it. The plan file can have placeholder content like "Anchor: TBD — pick during /td-morning." A half-plan is better than abandoning.
- **Calendar tool fails / returns no events** — treat as no fixed events. Continue.
- **Todoist update fails on a task** (e.g., section ID stale) — regenerate `section-ids.json` for that project, retry. If still failing, halt and report.
- **The user invokes `/td-plan` mid-morning for "today"** — works fine. Effectively a deeper version of `/td-morning`. No special handling needed.
- **Re-run for the same day** — overwrites the plan file. Tasks already in Todoist are updated in place (date stays, labels may add).

## Notes for /td-morning integration

`/td-morning` checks for `~/.local/share/todoist-skills/plans/<today>.md` at the start of its Prelude. If it exists:

- Read it.
- Use the saved anchor + supporting + time blocks as the **draft**.
- The morning ritual confirms/adjusts rather than building from scratch.
- The fast path is even faster — usually one "confirm" turn.

If no plan file exists, `/td-morning` proceeds with its existing flow unchanged.
