---
name: td-morning
description: Use when the user invokes /td-morning or asks for a morning planning ritual. Pulls 🎯 Today + 📅 This week, triages stale carry-overs, locks in a LangRelay deep_work block, surfaces client commitments, suggests quick wins.
---

# /td-morning

A 2-3 minute ritual to lock in the day. Biases toward LangRelay for the deep_work slot.

**Default invocation:** `/td-morning`.

**Quick variant:** when the user invokes `/td-morning` and then says "quick", "fast", "skip the triage", or similar — OR includes any of those words in the same message — run the **Fast path** (see end of this file). Do NOT rely on slash-command arguments (`/td-morning fast`) — Claude Code parses that as a lookup for a skill literally named `td-morning fast` and won't match. The trigger is natural-language in the user's message, not a flag.

## Prelude (runs every time)

1. Resolve this skill's directory. The bundle root is two levels up from this file: `<this-file>/../../`. Resolve to `~/dev/skills/todoist-skills/`.

2. Use the Read tool to load `<bundle>/skills/td-gtd/SKILL.md`. Internalize the system map, filter map, label semantics, growth contract, and capability table.

3. Use the Read tool to load `<bundle>/skills/_shared/environment.md`. Run the runtime detection. If detection halts, stop here.

4. If MCP mode and unauthed → run the auth bootstrap from `_shared/environment.md`. If the user declines auth → halt.

5. Once detection completes, you have a runtime and an operation→tool mapping. Proceed.

## Default flow

Run these steps in order. If anything errors, stop and report — do not silently skip.

### Step 1: Pull state

In **CLI mode**, run these in parallel (multiple Bash calls in one tool call):

```bash
td filter view "🎯 Today" --json
td filter view "📅 This week" --json
td filter view "⏳ Waiting check" --json
td task list --project "id:6gg3XRqxXPWPMwH7" --section "🔥 In Progress" --json
td task list --project "id:6gg3XRqxXPWPMwH7" --section "This Week" --json
```

In **MCP mode**, perform equivalent calls. Warn the user upfront: "Pulling state in MCP mode — this takes ~15s."

### Step 2: First-run check (handle empty system)

If 🎯 Today, 📅 This week, AND LangRelay sections are ALL empty:

> Your Todoist system looks freshly set up — nothing on Today or This Week yet. Want me to: (a) walk you through picking 3-5 tasks from a project's Backlog for the week, (b) just show all projects so you can plan manually, or (c) skip and come back tomorrow?

Wait for the user's pick. Then exit the morning ritual — there's no day to triage on a true first-run.

### Step 3: Triage 🎯 Today

For each task in 🎯 Today:
- **Stale carry-over** (due >1 day ago, still on Today): "X is from N days ago — still today, or push?" Accept: today / tomorrow / This Week / Someday.
- **Vague** (one-word title, no label, no description): "Y has no context — clarify now, or send to Inbox?"
- **Clear and current** → no action.

Batch the questions: if 3+ tasks need triage, present as a list and accept a single response.

If 🎯 Today is empty but other filters have items: "🎯 Today is empty. Want to date 3-5 items from 📅 This week for today?" Show top items from This Week, let user multi-select.

### Step 4: Lock the deep_work slot

This is the load-bearing step. Default = LangRelay.

1. From the LangRelay pull (Step 1), pick the strongest candidate in this order:
   - Anything in `🔥 In Progress` (continue what's started)
   - p1 or p2 tasks in `This Week`
   - Any task in `This Week`
   - If nothing in `This Week` → "LangRelay's This Week is empty. Promote one from Backlog now, shift to Selektable, or skip the LangRelay default?"

2. Propose: "Deep work block: **<task title>** (~90 min). OK, or pick another?"

3. If user rejects: one diagnostic — "What kind of day is this? client-heavy / admin / rest / something else?" Adjust once, move on.

### Step 5: Client commitments check (count-based heuristic)

For each active client, list tasks due today or overdue:

```bash
td task list --project "id:6gg3XVMc99VMGxMV" --json   # Leat
td task list --project "id:6gg3XVq73jRXwxHR" --json   # Hairgivers
td task list --project "id:6gg3XVQrrCgRV9Qv" --json   # Mediastap
```

Filter the JSON client-side for `due.date <= today's date` (use local time, `date +%Y-%m-%d`). Do NOT chain a `--filter` flag onto `td task list` — that flag is not documented for this command and may silently no-op.

**Heaviness flag**: if total client tasks due today/overdue ≥ 4 (count, not duration — Todoist `duration` is rarely populated and a sum-based heuristic fires constantly):

> Heavy client day (N client tasks queued). The deep_work block on **<task>** may get squeezed — move it to earlier in the day?

Otherwise just show the list.

### Step 6: Quick wins shortlist

```bash
td filter view "⚡ Quick wins" --json
```

Show 2-3 candidates. No action required — visible for between-block windows.

### Step 7: Output the 3-line plan

```
Today: deep_work on <task>.
Client commitments: <a>, <b>.
Quick wins available: <p>, <q>, <r>.
```

Skill exits here.

## Fast path

Triggered when the user invokes `/td-morning` and the same message (or the next user turn) contains "quick", "fast", "skip the triage", "just the essentials", or similar.

1. Run the Prelude (still required — must detect runtime).
2. Pull only:
   - `td filter view "🎯 Today" --json`
   - `td task list --project "id:6gg3XRqxXPWPMwH7" --section "🔥 In Progress" --json` (LangRelay's in-progress section)
3. Show: `🎯 Today (N items): [...]. Proposed deep_work: <LangRelay task or "no LangRelay in-progress, pick one from This Week?">`.
4. User confirms or rejects in one turn. If rejected, accept a one-line override ("do Selektable demo today instead"). No follow-up questions.
5. Skill exits.

Total runtime in CLI: ~2 seconds. In MCP: ~5 seconds.

## Failure modes (handled in-skill)

- **Empty 🎯 Today, non-empty This Week** — offer multi-select from This Week (Step 3 handles this).
- **First-run: everything empty** — Step 2 catches this and offers the on-ramp.
- **Skipped yesterday, many overdue items** — batch into Step 3 as a single "These 8 are overdue: bulk-push to This Week, or pick which to keep on Today?"
- **MCP >10s** — show "running in MCP mode, hold on" before the user thinks it's hung.
- **Filter doesn't exist** (e.g., user renamed it) — halt with: "Filter `<name>` not found. Was it renamed? Check the system map in `td-gtd` section 2."
- **Project ID returns 404** — halt with: "Project `<name>` (id:<id>) doesn't resolve. Was it deleted? Update `td-gtd` section 1."
