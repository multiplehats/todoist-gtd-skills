---
name: td-shutdown
description: Use when the user invokes /td-shutdown or asks for an end-of-day ritual. Cleans 🎯 Today, captures own/client time split + day-type into the journal, surfaces tomorrow's anchor and waiting-for nudges.
---

# /td-shutdown

A 2-3 minute close. Two jobs: leave Todoist clean for tomorrow, and capture today's signal for Phase 2's weekly review.

**Invocation:** `/td-shutdown`.

## Prelude

1. Resolve this skill's directory. Bundle root is two levels up.
2. Read `<bundle>/skills/td-gtd/SKILL.md`. Internalize rules, growth contract, journal schema (section 9), and capability table (section 8).
3. Read `<bundle>/skills/_shared/environment.md`. Run detection. Halt if no runtime. If MCP-unauthed and user declines auth → halt.
4. Proceed.

## Flow

### Step 1: Walk today

Pull `🎯 Today` AND tasks completed today (use **local time**, not UTC):

```bash
td filter view "🎯 Today" --json
td completed list --since "$(date +%Y-%m-%d)T00:00:00" --json
```

For each task on `🎯 Today` (excluding ones already completed):

- **Done?** → confirm completion via `td task complete "id:<task_id>"`. If it should recur, ask: "Recur weekly/daily/monthly, or one-shot?" and apply via `td task update "id:<task_id>" --due "every <interval>"`.
- **Started but not done?** → drag to `Doing` section on its project board. Look up the section ID for "Doing" within that task's project (from the cached `section-ids.json` map — see Task 7.5; if the cache is stale, regenerate via `td section list --project "id:<project_id>"`):
  ```bash
  td task move "id:<task_id>" --section "id:<doing_section_id>"
  ```
  Then ask: "Keep dated today, or push to tomorrow / This Week / unschedule?"
- **Untouched?** → "Reschedule (when?), move back to `This Week`, or move to `💭 Someday / Maybe`?"

Batch prompts: if 5+ tasks fall in the same bucket, present as a list with bulk-actions.

### Step 2: Capture loose ends

One prompt:

> Anything in your head that didn't make it into Todoist today?

For each item the user mentions, route to Inbox:

```bash
td task quickadd "<user's text>"
```

If user says "nothing" / "no" / silence → skip.

### Step 3: Day type + own/client split

**Prompt 1:** "Day type — workday, partial, or off?"
- `off` → skip Prompt 2. Omit `own:` and `client:` from the journal line.
- `partial` → Prompt 2 is optional ("hours, or skip?"). Allow either both fields, neither, or one.
- `workday` → Prompt 2 is asked.

**Prompt 2:** "Rough split — own/client hours? (e.g., '2h own / 5h client' or '3 client, no own' or 'skip')."

Parse forms:
- `"2h own / 5h client"` → `own:2h client:5h`
- `"all client, 6h"` → `own:0h client:6h`
- `"3 own"` → `own:3h client:0h`
- `"a few hours each, mostly client"` → `own:fuzzy client:fuzzy` + the original text as the note
- `"skip"` → omit both fields

### Step 4: Write the journal entry (data-loss-safe)

Journal path: `~/.local/share/todoist-skills/journal/YYYY-MM-DD.md` (local date).

**Use this exact Bash recipe.** It handles all edge cases: existing canonical-only file, canonical + body, free-text-only file (no canonical), single-line file, missing trailing newline, double-run in the same day:

```bash
set -euo pipefail
JOURNAL_DIR="$HOME/.local/share/todoist-skills/journal"
mkdir -p "$JOURNAL_DIR"
DATE="$(date +%Y-%m-%d)"   # LOCAL time, NOT date -u
FILE="$JOURNAL_DIR/$DATE.md"

# Build NEW_LINE from the user's answers in Step 3. Example:
NEW_LINE="$DATE | type:workday | own:2h | client:5h | note: shipped CJ-40"
# If type:off → "$DATE | type:off"
# If user gave no note → drop the " | note: ..." suffix

TMP="$FILE.tmp.$$"

if [ -f "$FILE" ]; then
  # Read first line safely (even if file has no trailing newline)
  FIRST_LINE="$(head -n 1 "$FILE")"
  if printf '%s' "$FIRST_LINE" | grep -qE '^[0-9]{4}-[0-9]{2}-[0-9]{2} \|'; then
    # First line is a canonical entry — REPLACE it, preserve the rest
    {
      printf '%s\n' "$NEW_LINE"
      tail -n +2 "$FILE"
    } > "$TMP"
  else
    # First line is NOT canonical — PREPEND new line, keep all original content
    {
      printf '%s\n' "$NEW_LINE"
      cat "$FILE"
    } > "$TMP"
  fi
  # Atomic rename on the same filesystem — survives crashes mid-write
  mv "$TMP" "$FILE"
else
  printf '%s\n' "$NEW_LINE" > "$FILE"
fi
```

Why this recipe and not the simpler version:

- `printf '%s\n'` guarantees a trailing newline, so the next `cat` / `tail` doesn't smush content together.
- The regex check on the first line prevents `tail -n +2` from silently deleting a user's free-text first line on files with no canonical line.
- `mv` is atomic on the same filesystem — a crash mid-write either leaves the old file intact or the new one complete, never a half-written file.
- `$$` in the tmp filename avoids collisions if two shells run simultaneously. NOTE: this does NOT provide true concurrency safety — for a single-user, single-machine system this is acceptable; `flock` would be the upgrade if needed later.

**MCP mode fallback**: if no filesystem write tool is available in MCP mode, tell the user:

> Here's today's journal line — paste it manually into `~/.local/share/todoist-skills/journal/YYYY-MM-DD.md`:
>
>     <NEW_LINE>

Continue the rest of the shutdown.

### Step 5: Tomorrow setup (always ask, don't precompute)

Ask once:

> Want to pick tomorrow's anchor task now while it's fresh? (Or skip — fine to do it in /td-morning.)

If yes, pull `📅 This week`:

```bash
td filter view "📅 This week" --json
```

Show 3-5 candidates, let user pick one. Set its due date to tomorrow:

```bash
td task reschedule "id:<task_id>" "tomorrow"
```

If skipped / "no" → move on. (We don't auto-detect tomorrow's count because that requires a query whose syntax — `td filter view --filter "tomorrow"` — does NOT work, and we don't want a fragile branch on a feature the user can just say yes/no to.)

### Step 6: Waiting nudge

```bash
td filter view "⏳ Waiting check" --json
```

For each item, check creation date (or last-update if available). If >7 days old, list and ask "Nudge tomorrow?" — if yes:

```bash
td task quickadd "Nudge <person/thing> on <waiting item> tomorrow"
```

### Step 7: Output the 2-line close

```
Closed: <N> done, <M> pushed, <K> new in Inbox. Day: workday | own:2h client:5h. Tomorrow's anchor: <task or "none">.
```

Skill exits.

## Failure modes

- **Skipped 3 days** — first prompt: "Last entry was N days ago. Log the last N days now, or skip ahead?" If "log": loop day-type + split prompts for each missing day, using the appropriate past date in `$DATE`. If "skip": just do today.
- **Ambiguous split** ("a few hours own, mostly client") — accept as fuzzy: `own:fuzzy client:fuzzy | note: <quote>`. Phase 2 ignores fuzzy days for ratio math.
- **Journal write fails** (permission, disk full) — tell the user the path that failed; ask if they want the line printed for manual paste. Don't halt the rest.
- **Filesystem write unavailable (MCP)** — fallback in Step 4.
- **`td completed list` returns nothing today** — fine, no completed-task review needed.
- **Section ID lookup miss** (section name not found in cache) — regenerate the cache for that project: `td section list --project "id:<project_id>" --json`; retry the move.
