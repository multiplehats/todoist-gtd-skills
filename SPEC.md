# todoist-skills — design spec

_Drafted: 2026-05-17. Status: design approved by Chris (with independent-review revisions), pending implementation plan._

## Purpose

A bundle of Claude skills that turn a freshly-migrated Todoist (GTD + Kanban) into a daily-use system. The skills work in both Claude Code (`td` CLI) and Claude Desktop (Todoist MCP connector).

**Phase 1 (MVP — this spec)**: instrumentation. The skills establish the rituals and produce the daily data the growth contract needs. Active enforcement of the growth target is **not** in this phase — that arrives in Phase 2 when `/td-weekly-review` ships and starts reading the journal.

**Phase 2 (future)**: closed-loop growth — `/td-weekly-review` reads the journal, computes the rolling own/client ratio against the target, and names a corrective action. `/td-capture` enforces soft client caps. Other rituals join as needed.

Being explicit about the phase split because the growth contract without a reader is performative.

## Context

Chris is a solo freelance dev. Revenue: ~€6500/mo, currently ~85% from client work (Leat, Hairgivers, Mediastap), ~15% from own work (LangRelay, Selektable, Side Projects). LangRelay is the primary growth bet. The Todoist system was set up via the `todoist-migration` project on 2026-05-17 and follows GTD + Kanban principles documented in `/Users/chris/Downloads/todoist-migration/04-execution-log.md`.

Without active defense, client work expands to fill all available capacity. These skills are the eventual active defense — and active offense — for growing own-work share. MVP instruments; Phase 2 enforces.

## Bundle structure

Canonical source: `~/dev/skills/todoist-skills/` (git repo, mirrors the `parsew-skills` pattern).

```
~/dev/skills/todoist-skills/
├── .git/
├── README.md
├── SPEC.md                         ← this file
└── skills/
    ├── td-gtd/
    │   └── SKILL.md                ← rules doc; Read'd by the ritual skills, not auto-triggered
    ├── td-morning/
    │   └── SKILL.md                ← /td-morning ritual
    ├── td-shutdown/
    │   └── SKILL.md                ← /td-shutdown ritual
    └── _shared/
        └── environment.md          ← CLI-vs-MCP detection; explicitly Read'd by each skill
```

**User-state location** (separate from the bundle):

```
~/.local/share/todoist-skills/
└── journal/
    └── YYYY-MM-DD.md               ← one file per day, written by /td-shutdown
```

Rationale: the bundle is code (shareable, distributable, potentially read-only when installed); the journal is per-machine user state. Conflating them breaks portability.

**Symlink chain** (per individual skill):

```
~/.claude/skills/<name>   →  ../../.agents/skills/<name>
~/.agents/skills/<name>   →  ~/dev/skills/todoist-skills/skills/<name>
```

Claude Code and Claude Desktop both read skills from `~/.claude/skills/` on macOS, so this layout serves both runtimes.

Slash command convention: skill name = slash command, prefixed `td-` (matches the `td` CLI binary). So `/td-morning`, `/td-shutdown`. `td-gtd` has no slash command — it exists as a reference file the rituals Read.

## MVP scope

Three skill folders:

1. **`td-gtd`** — rules doc. Read by the ritual skills at invocation time. Contains: system map, filter/label semantics, growth contract, capture conventions, capability table (CLI vs MCP).

2. **`/td-morning`** — start-of-day ritual (~2-3 min). Locks in the day's deep_work block, surfaces client commitments, suggests quick wins. Has a **fast-path variant** for low-friction days (just "today + one deep_work suggestion").

3. **`/td-shutdown`** — end-of-day ritual (~2-3 min). Cleans the board, captures the day's own/client time split + day-type into the journal.

Out of scope for MVP, sequenced for Phase 2+: `/td-weekly-review` (reads journal, closes growth loop), `/td-capture` (soft cap enforcement), `/td-inbox-zero`, `/td-triage`, `/td-client-status`, `/td-project-walk`, `/td-standup`.

## How the rules doc is actually loaded

The independent review flagged that "auto-loaded via SKILL.md frontmatter keywords" is wishful — skills don't continuously scan conversation; they're matched against user requests at invocation time. Acknowledged.

**Mechanism instead**: each ritual SKILL.md begins with an explicit step:

```
1. Read `<bundle>/skills/td-gtd/SKILL.md` for the system map, filter/label rules,
   growth contract, and capability table.
2. Read `<bundle>/skills/_shared/environment.md` for runtime detection.
3. ...
```

`<bundle>` resolves to whatever path the skill is installed at. Each SKILL.md begins by determining its own absolute path (Claude can read its own location from context) and resolving siblings from there.

Cost: two extra `Read` tool calls per ritual invocation. Acceptable. Benefit: single source of truth for rules + detection.

## Skill: `td-gtd` (rules reference)

**Contents — sections in this order:**

1. **System map** — projects with IDs (LangRelay, Selektable, the Clients parent + children, the Side Projects parent + children incl. Job Boards sub-parent, 🔧 Admin, 🏠 Personal, ⏳ Waiting For, 💭 Someday / Maybe, Inbox). Each project entry includes a **validation hint**: "if a referenced ID returns 404, halt and ask the user to re-run discovery."

2. **Filter map** — the 7 filters with queries and intent.

3. **Labels & semantics** — the 9 labels (`deep_work`, `quick`, `errand`, `calls`, `email`, `waiting`, `two_minutes`, `agent`, `review`) with one-line "when to apply" rules.

4. **Growth contract**:
   - Baseline (2026-05-17): ~€6500/mo client revenue, 0% own-product revenue, ~85% client time
   - 6-month target (2026-11-17): 70% client / 30% own (time + revenue)
   - 12-month target (2027-05-17): 50% client / 50% own
   - Primary growth bet: LangRelay
   - Containment style: soft — note when over a future per-client weekly cap, do not refuse
   - Ratchet: instrumented in MVP via daily journal; activated in Phase 2 by `/td-weekly-review`

5. **Capture conventions** — Inbox is the only valid landing zone for unclarified items. Everything else must have project + (date OR Someday). No floating tasks.

6. **Date discipline** — date = commitment. Don't date for sorting. Use `This Week` section + filter for "soon-ish, not today."

7. **When to ask vs. when to act** — heuristics for the rituals. Ambiguous routing = ask one question; clear routing = do and confirm.

8. **Capability table** — operations in plain English with CLI-mode and MCP-mode notes. Filled in during implementation after a `tool_search` inventory; degradation notes per operation ("MCP mode: no bulk move, loop one-by-one").

9. **Journal schema** — referenced by `/td-shutdown` and (eventually) `/td-weekly-review`. See "Persistence" below for the format.

**Length target**: 150–250 lines, mostly tables.

**Excluded**: full CLI syntax (lives in `todoist-cli` skill), exhaustive MCP tool reference (discovered at runtime), step-by-step ritual flows (live in the ritual skills).

## Skill: `/td-morning`

**Purpose**: 2-3 minute ritual to lock in the day. Biases toward LangRelay/own-work for the deep_work slot.

**Default flow** (SKILL.md walks Claude through these in order):

1. Read `td-gtd/SKILL.md` and `_shared/environment.md`.
2. Detect environment; if unauthed MCP → run auth bootstrap (see Environment section). If no runtime → halt.
3. **Pull state**: 🎯 Today, 📅 This week, ⏳ Waiting check, LangRelay's `🔥 In Progress` and `This Week`. (In MCP mode, this is batched; user is warned if it takes >10s.)
4. **Triage Today**: stale carry-overs (push or keep?), vague items (clarify or move to Inbox?).
5. **Lock the deep_work slot** — load-bearing step.
   - Default: 1 LangRelay deep_work block, ~90 min, from LangRelay's `This Week` or `🔥 In Progress`. Skill names the specific task.
   - If LangRelay has nothing actionable: propose Selektable, then a Side Project. Explain why.
   - If rejected: ask what kind of day this is (client emergency, admin, rest) and adjust.
6. **Client commitments check**: per active client, overdue or due-today? Surface. If today's client load ≥4h: flag "deep_work slot may get squeezed — move it earlier?"
7. **Quick wins shortlist**: 2-3 `@quick` items, optional, just visible.
8. **3-line plan output**: `Today: [1] deep_work on X. [2] client commitments: A, B. [3] quick wins available: P, Q, R.`

**Fast-path variant** (invoked via `/td-morning fast` or when the user says "quick version"):

1. Read shared files + detect environment.
2. Show 🎯 Today and one proposed LangRelay deep_work task. That's it.
3. User picks or rejects in one turn; skill exits.

Fast path is for client-emergency days when the full triage is friction. The default is still the full flow.

**Non-goals**: auto-add tasks, auto-reschedule, calendar time-blocking, ratio computation.

**Failure modes designed against**:
- Empty 🎯 Today → default to "pick 3-5 from This Week" instead of nagging.
- Skipped yesterday → batch overdue items into the triage step.
- MCP slow / >10s probe → show a "running in Desktop, hold on" message before the user thinks the skill is hung.

## Skill: `/td-shutdown`

**Purpose**: 2-3 minute close. Clean the board AND capture signal for Phase 2's weekly review.

**Flow**:

1. Read `td-gtd/SKILL.md` and `_shared/environment.md`.
2. Detect environment; halt or auth-bootstrap as needed.
3. **Walk today**: for each task on 🎯 Today (or completed today), confirm done / drag started-not-done to `Doing` on its project board / reschedule or move-to-This-Week the untouched.
4. **Capture loose ends**: "Anything in your head that didn't make it into Todoist today?" → mentioned items land in Inbox.
5. **Day type + own/client split** (ratchet input):
   - Day type: `workday | partial | off` (asked first; if `off`, skip the hours question).
   - Hours: rough estimate. Accept `"2h own / 5h client"`, `"all client"`, `"4h total off-balance"`, `"skip"`.
6. **Write/update journal entry** — see Persistence for format and concurrency rules.
7. **Tomorrow setup (optional)**: if tomorrow's 🎯 Today has 0 items and 📅 This week has ≥3: "want to pick tomorrow's anchor while it's fresh?" One prompt, skippable.
8. **Waiting nudge**: any ⏳ Waiting check item >7 days old → surface for tomorrow.
9. **2-line close**: `Closed: [N] done, [M] pushed, [K] new in Inbox. Day: workday | own:2h client:5h. Tomorrow's anchor: Y.`

**Non-goals**: compute weekly ratio (Phase 2), write diary entries, force time logging.

**Failure modes**:
- Skipped 3 days → "log the last 3 days, or skip ahead?" One prompt, no guilt.
- "Rough" or "skip" accepted for the split — Phase 2 will note the gap, not block.
- Day type `off` doesn't pollute the ratio when `/td-weekly-review` later computes it.

## Cross-cutting: environment detection

**Problem**: skills must work in both Claude Code (`td` CLI) and Claude Desktop (Todoist MCP connector). MCP tool names differ across runtimes and tools may be deferred (schemas not loaded until probed). The independent review flagged that hard-coding specific MCP tool names (`Todoist:user-info`, `mcp__claude_ai_Todoist__find-tasks-by-date`, etc.) is brittle and that the auth bootstrap was missing.

**Solution**: detection logic lives in `_shared/environment.md`, Read'd by each skill.

**Detection order** (each skill runs this once at start):

1. Bash `command -v td` succeeds → **CLI mode**. Use `td` for everything; reference `todoist-cli` skill for syntax. Exit detection.

2. Else scan currently-visible tools for any name matching `^Todoist:` OR containing `Todoist` after an `mcp__` prefix (covers both observed naming conventions). If matches found → **MCP mode**. Skip to capability discovery.

3. Else call `tool_search` with query `todoist` (deferred tools may exist but not be loaded). If results returned, load their schemas and re-check tool visibility.

4. Else assume no runtime → tell the user: "I need either the `td` CLI (Claude Code) or the Todoist MCP connector (Claude Desktop → Connectors menu). Stopping." Halt.

**Capability discovery** (MCP mode, runs once per session):

- Probe with whatever tool looks like a lightweight identity check (`*user-info*`, `*user*`, `*me*` by name match). If the probe succeeds → authed, proceed.
- If only `authenticate` / `complete_authentication` tools are visible (the unauthed state in Claude Code today): **auth bootstrap**. Walk the user: "Todoist MCP is installed but not authenticated. I'll trigger the auth flow now. Please complete the OAuth in your browser when it opens." Call `authenticate`, wait, then `complete_authentication`. Once authed, re-run `tool_search` to surface the real tools.
- Cache the discovered tool-name → operation mapping in working memory for this turn so each operation doesn't re-search.

**Operation → tool mapping** is by name match, not hard-coded literal: when the skill needs "list tasks by filter", it picks the tool whose name best matches `*find*task*` / `*list*task*` / `*filter*`. Adapts to either runtime's naming.

**Capability table** (in `td-gtd`) lists operations in plain English with one column per mode and a degradation note per operation. Filled in during implementation after running `tool_search` and inspecting both surfaces.

**Performance**: bulk operations are one shell call in CLI mode, but 5-10 tool calls in MCP mode. Skills batch where possible and warn explicitly: "MCP mode: this takes ~20s; for daily speed, prefer Claude Code." `/td-morning` has a fast-path variant for slow-runtime days.

## Persistence: the journal

**Location**: `~/.local/share/todoist-skills/journal/YYYY-MM-DD.md` (outside the skill bundle).

**Format** (one file per day; one canonical line, optional free-text below):

```
2026-05-17 | type:workday | own:2h | client:5h | note: shipped CJ-40
```

Fields:
- `type:` — `workday | partial | off` (mandatory; `off` days are excluded from ratio math)
- `own:` — rough hours on own-work (LangRelay, Selektable, Side Projects, own admin)
- `client:` — rough hours on client work (Leat, Hairgivers, Mediastap)
- `note:` — optional free-text (what shipped, what blocked, etc.)

`own:` and `client:` are optional on `partial` days and skipped on `off` days.

**Concurrency rule** (addresses #10 in the review): `/td-shutdown` does **read-modify-write**. If `YYYY-MM-DD.md` exists, read it, replace the canonical first line, preserve any free-text body. Never blind-append. Eliminates duplicate-lines from double-runs and same-day corrections.

**Read access**: Phase 2 `/td-weekly-review` reads the last 7 (or 30) journal files, parses the canonical line via regex, excludes `type:off` from the denominator, computes the rolling own/client ratio vs the growth contract.

**Why not Todoist itself**: Todoist doesn't track time. Putting per-day numbers into task descriptions is fragile and unsearchable.

**Why not CSV/sqlite**: friction. The ritual writes one line; markdown is human-editable when the skill writes a wrong line. SQLite would require a schema, migrations, and tool support across both runtimes for a thing the user can type by hand.

## What this spec does NOT cover

- Implementation order (writing-plans skill produces that next)
- The other 7 future skills — designed later once MVP usage data exists
- Closed-loop growth enforcement — that's Phase 2's job, by definition
- A scheduler/cron layer (manual invocation only)
- Calendar integration (Todoist + Google Calendar correlation is a separate future concern)
- Client billing / invoicing flows (Moneybird MCP exists but is out of scope here)

## Open questions deferred to implementation

- Exact MCP tool names per operation — discovered via `tool_search`, not specified here
- Per-client weekly hour caps for `/td-capture` (set when that skill is designed)
- Whether the journal should also record completed-task IDs for richer Phase 2 analysis (default: no; revisit when `/td-weekly-review` lands)
- Exact phrasing for the auth-bootstrap walk-through (write during implementation)

## Acknowledged limitations of this MVP

Surfacing what the independent review correctly flagged, so it's not buried:

- **MVP is instrumentation, not enforcement.** The growth contract is documented in `td-gtd`, the daily data is captured in the journal, but nothing in MVP closes the loop. That's Phase 2's job — and the spec is honest about it rather than pretending MVP grows the 15% by itself.
- **The "active defense" of own-work in MVP is limited to**: `/td-morning` defaulting the deep_work block to LangRelay, and `/td-morning` flagging client-heavy days. No weekly cap, no decline-suggestion, no Inbox-routing bias. Stronger defense ships in Phase 2.
- **Project IDs are hard-coded in `td-gtd`.** Stable in normal use; vulnerable if a project is deleted+recreated. The validation hint in section 1 of `td-gtd` is the mitigation — not perfect, but cheap.
- **Capability table is finalized during implementation, not now.** The spec approves the architecture without proving each MCP primitive exists. If a critical capability is missing on the MCP side, that's a discovered constraint to handle in the implementation plan.
