# todoist-skills — design spec

_Drafted: 2026-05-17. Status: design approved by Chris, pending implementation plan._

## Purpose

A bundle of Claude skills that turn a freshly-migrated Todoist (GTD + Kanban) into a daily-use system. The skills work in both Claude Code (`td` CLI) and Claude Desktop (Todoist MCP connector). They are designed to actively shift the user's revenue mix — currently 85% client / 15% own work — toward a target of 50/50 within 12 months.

## Context

Chris is a solo freelance dev. Revenue: ~€6500/mo, currently ~85% from client work (Leat, Hairgivers, Mediastap), ~15% from own work (LangRelay, Selektable, Side Projects). LangRelay is the primary growth bet. The Todoist system was set up via the `todoist-migration` project on 2026-05-17 and follows GTD + Kanban principles documented in `/Users/chris/Downloads/todoist-migration/04-execution-log.md`.

Without active defense, client work expands to fill all available capacity. These skills are the active defense — and the active offense — for growing own-work share.

## Bundle structure

Canonical source: `~/dev/skills/todoist-skills/` (git repo, mirrors the `parsew-skills` pattern).

```
~/dev/skills/todoist-skills/
├── .git/
├── README.md
├── SPEC.md                         ← this file
├── journal/                        ← daily own/client time logs (one md per day)
└── skills/
    ├── td-gtd/
    │   └── SKILL.md                ← always-loaded rules + growth contract
    ├── td-morning/
    │   └── SKILL.md                ← /td-morning ritual
    ├── td-shutdown/
    │   └── SKILL.md                ← /td-shutdown ritual
    └── _shared/
        └── environment.md          ← CLI-vs-MCP detection snippet, referenced by each skill
```

Symlink chain (per individual skill):

```
~/.claude/skills/<name>   →  ../../.agents/skills/<name>
~/.agents/skills/<name>   →  ~/dev/skills/todoist-skills/skills/<name>
```

Editing once in the bundle dir propagates everywhere; git-tracked.

Slash command convention: skill name = slash command, prefixed `td-` (matches the `td` CLI binary). So `/td-morning`, `/td-shutdown`. `td-gtd` is auto-loaded, not invoked.

## MVP scope

Three skills:

1. `td-gtd` — auto-loaded rules doc. The system map, filter/label semantics, growth contract, capture conventions.
2. `/td-morning` — start-of-day ritual (~2-3 min). Locks in the day's deep_work block, surfaces client commitments, suggests quick wins.
3. `/td-shutdown` — end-of-day ritual (~2-3 min). Cleans the board, captures the day's own/client time split into the journal.

Out of scope for MVP (tracked for later): `/td-capture`, `/td-inbox-zero`, `/td-triage`, `/td-weekly-review`, `/td-client-status`, `/td-project-walk`, `/td-standup`.

## Skill: `td-gtd` (always-loaded)

**Frontmatter description** (the trigger): includes terms `Todoist`, `GTD`, `task system`, `weekly review`, `daily planning`, so Claude Code/Desktop loads it whenever the user mentions any of those.

**Contents — sections in this order:**

1. **System map** — projects with IDs (LangRelay, Selektable, the Clients parent + children, the Side Projects parent + children including Job Boards sub-parent, 🔧 Admin, 🏠 Personal, ⏳ Waiting For, 💭 Someday / Maybe, Inbox). Project IDs included so the rituals can act without re-lookup.

2. **Filter map** — the 7 filters with queries and intent. Source-of-truth for what filter to use for each purpose.

3. **Labels & semantics** — the 9 labels (`deep_work`, `quick`, `errand`, `calls`, `email`, `waiting`, `two_minutes`, `agent`, `review`) with one-line "when to apply" rules.

4. **Growth contract** — explicit:
   - Baseline (2026-05-17): ~€6500/mo client revenue, 0% own-product revenue, ~85% client time
   - 6-month target (2026-11-17): 70% client / 30% own (time + revenue)
   - 12-month target (2027-05-17): 50% client / 50% own
   - Primary growth bet: LangRelay
   - Containment style: soft — note when over weekly client cap, do not refuse
   - Ratchet: `/td-shutdown` logs daily split; `/td-weekly-review` (future) measures the rolling ratio against target and names the corrective action.

5. **Capture conventions** — Inbox is the only valid landing zone for unclarified items. Everything else must have project + (date OR Someday). No floating tasks.

6. **Date discipline** — date = commitment. Don't date for sorting. Use `This Week` section + filter for "soon-ish, not today."

7. **When to ask vs. when to act** — heuristics for the rituals. Ambiguous routing = ask one question; clear routing = do and confirm.

8. **Capability table** — operations in plain English with a "CLI mode" and "MCP mode" column. Concrete tool names left blank in the spec; filled in during implementation after a `tool_search` inventory.

**Length target**: 150–250 lines. Mostly tables.

**Excluded**: full CLI syntax (lives in `todoist-cli` skill), exhaustive MCP tool reference (self-describing), step-by-step ritual flows (live in the ritual skills).

## Skill: `/td-morning`

**Purpose**: 2-3 minute ritual to lock in the day. Bias toward defending and growing the 15%.

**Flow** (SKILL.md walks Claude through these in order):

1. **Pull state**
   - 🎯 Today contents
   - 📅 This week contents
   - ⏳ Waiting check (anything to nudge?)
   - LangRelay specifically: `🔥 In Progress` and `This Week`

2. **Triage Today**
   - Stale carry-overs from yesterday → push or keep?
   - Vague items missing context → clarify or move to Inbox?

3. **Lock the deep_work slot** (load-bearing)
   - Default: 1 LangRelay deep_work block, ~90 min, from LangRelay's `This Week` or `🔥 In Progress`. Skill names the specific task.
   - If LangRelay has nothing actionable: propose Selektable, then a Side Project, in that order. Explain why.
   - If rejected: ask what kind of day this is (client emergency, admin, rest) and adjust.

4. **Client commitments check**
   - Per active client (Leat, Hairgivers, Mediastap): any overdue or due-today? Surface.
   - If today's client load already looks ≥4h: flag "deep_work slot may get squeezed — move it earlier?"

5. **Quick wins shortlist**
   - 2-3 `@quick` items shown as candidates for between-block windows. Optional, just visible.

6. **3-line plan output**
   - `Today: [1] deep_work on X. [2] client commitments: A, B. [3] quick wins available: P, Q, R.`
   - Nothing persisted — the plan is 🎯 Today itself.

**Non-goals**: auto-add tasks, auto-reschedule, calendar time-blocking, weekly-ratio computation.

**Failure modes designed against**:
- Empty 🎯 Today → default to "pick 3-5 from This Week" instead of nagging.
- Skipped yesterday → batch overdue items into the triage step rather than per-task interruptions.

## Skill: `/td-shutdown`

**Purpose**: 2-3 minute end-of-day close. Clean the board AND capture signal for the weekly review.

**Flow**:

1. **Walk today**
   - Each task on 🎯 Today (or completed today): done / in-progress / untouched. Confirm completion, drag started-but-not-done items to `Doing` on their project board, reschedule or move-to-This-Week the untouched.

2. **Capture loose ends**
   - "Anything in your head that didn't make it into Todoist today?" → mentioned items land in Inbox.

3. **Log own/client time split** (ratchet input)
   - Rough estimate, not stopwatch: `"2h own / 5h client"`, `"all client"`, `"off"`.
   - Persisted to `~/dev/skills/todoist-skills/journal/YYYY-MM-DD.md` as one line: `2026-05-17 | own:2h | client:5h | note: shipped CJ-40`.
   - Reason for external journal: Todoist doesn't track time and the growth contract needs raw data. Single-file-per-day, plain markdown, git-trackable.

4. **Tomorrow setup (optional)**
   - If 🎯 Today for tomorrow has 0 items and 📅 This week has ≥3: "want to pick tomorrow's anchor while it's fresh?" One prompt, skippable.

5. **Waiting/blocked nudge**
   - Any ⏳ Waiting check item >7 days old → surface for a nudge tomorrow.

6. **2-line close output**
   - `Closed: [N] done, [M] pushed, [K] new in Inbox. Own/client today: 2h/5h. Tomorrow's anchor: Y.`

**Non-goals**: compute weekly ratio (that's `/td-weekly-review` later), write diary entries, force time logging.

**Failure modes**:
- Skipped 3 days → on next run: "log the last 3 days, or skip ahead?" One prompt, no guilt.
- "Rough" or "skip" accepted for the split — weekly review will note the gap, not block.

## Cross-cutting: environment detection

**Problem**: skills must work in both Claude Code (`td` CLI) and Claude Desktop (Todoist MCP connector). MCP tools are named `Todoist:*` in Desktop and `mcp__claude_ai_Todoist__*` in Claude Code. Tools may be deferred (schemas not loaded until probed).

**Solution**: shared snippet in `_shared/environment.md`, read by each skill at the top of its run.

**Detection order**:

1. Bash `command -v td` succeeds → **CLI mode** (uses `td` for everything; references `todoist-cli` skill for syntax).
2. Else look for any tool named `Todoist:*` or `mcp__*Todoist*` → **MCP mode**.
3. Else call `tool_search` with query `todoist` (deferred tools may exist) → if results, load schemas and proceed in MCP mode.
4. Else probe anchor tool (`Todoist:user-info` or equivalent) → if succeeds, MCP mode.
5. Else halt with: "I need either the `td` CLI or the Todoist MCP connector. In Claude Desktop, add it via Connectors. In Claude Code, install `td`."

**Tool surface is discovered, not hard-coded**. Skills describe operations functionally ("list tasks for today"); the dispatcher runs `tool_search` once per turn in MCP mode and maps operations to whatever tool names exist.

**Auth probe**: first MCP call in a session hits a lightweight anchor (`Todoist:user-info`); if it fails with auth error, tell the user to re-auth via Connectors. Do not retry.

**Performance note**: bulk operations (e.g., walking 🎯 Today, pulling 5 filters) are one shell call in CLI mode, but 5-10 tool calls in MCP mode. Skills batch where possible and warn if Desktop feels slow: "MCP mode takes ~20s for this — for speed, run in Claude Code."

**Capability table** (filled in during implementation after a `tool_search` inventory). Operations to cover at minimum:

- List tasks by filter name
- List tasks by project / section
- Add task
- Complete / uncomplete task
- Update task (date, priority, labels, section)
- Move task between projects/sections
- Read project hierarchy

For each: note CLI command, MCP tool name, and any degradation (e.g., "MCP mode lacks bulk move — loop one-by-one").

## Persistence layer (the journal)

`~/dev/skills/todoist-skills/journal/YYYY-MM-DD.md`. One file per day. Single line:

```
2026-05-17 | own:2h | client:5h | note: shipped CJ-40
```

Append-only via `/td-shutdown`. Read by future `/td-weekly-review` to compute the 7-day own/client ratio against the growth-contract target.

Why not in Todoist: Todoist doesn't track time, and putting per-day numbers into task descriptions is fragile and unsearchable. Plain markdown, one file per day, in the bundle's git repo — durable, greppable, version-controlled.

Why not a CSV or sqlite: friction. The shutdown ritual writes one line; the weekly review reads ~7 files. Markdown is human-editable if the skill writes a wrong line.

## What this spec does NOT cover

- Implementation order (writing-plans skill produces that next)
- The other 7 skills (`/td-capture`, `/td-inbox-zero`, etc.) — designed later once MVP usage data exists
- A scheduler/cron layer (manual invocation only for MVP; cron deferred)
- Calendar integration (Todoist + Google Calendar correlation is a separate future concern)
- Client billing / invoicing flows (Moneybird MCP exists but is out of scope here)

## Open questions deferred to implementation

- Exact MCP tool names for each operation (resolve via `tool_search` during build)
- Per-client weekly hour caps for the soft-containment hint in future `/td-capture` (set when that skill is designed)
- Whether the journal should also record completed-task IDs for later analysis (default: no, keep it small — revisit if weekly review wants richer data)
