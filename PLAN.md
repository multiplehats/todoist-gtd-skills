# todoist-skills Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a 3-skill bundle (`td-gtd`, `/td-morning`, `/td-shutdown`) that supports a daily GTD ritual in both Claude Code (`td` CLI) and Claude Desktop (Todoist MCP connector), with a journal-based instrumentation layer that Phase 2's weekly review will consume.

**Architecture:** Single git repo at `~/dev/skills/todoist-skills/`. Three skill folders + one `_shared/` folder under `skills/`. Each individual skill is symlinked into `~/.agents/skills/<name>` which is in turn symlinked into `~/.claude/skills/<name>`. The ritual skills `Read` the rules doc (`td-gtd`) and detection snippet (`_shared/environment.md`) at invocation time. Journal lives at `~/.local/share/todoist-skills/journal/` — outside the bundle.

**Tech Stack:** Markdown only. No code beyond what skills instruct Claude to run via Bash / MCP tools. Source-of-truth: `~/dev/skills/todoist-skills/SPEC.md` (commit `594b737` or later).

---

## File Structure

Final layout when implementation is complete:

```
~/dev/skills/todoist-skills/
├── .git/
├── .gitignore
├── README.md
├── SPEC.md                                    (already exists)
├── PLAN.md                                    (this file)
└── skills/
    ├── _shared/
    │   └── environment.md                     environment detection logic
    ├── td-gtd/
    │   └── SKILL.md                           rules + system map + capability table
    ├── td-morning/
    │   └── SKILL.md                           /td-morning ritual
    └── td-shutdown/
        └── SKILL.md                           /td-shutdown ritual

~/.agents/skills/td-gtd          → ~/dev/skills/todoist-skills/skills/td-gtd
~/.agents/skills/td-morning      → ~/dev/skills/todoist-skills/skills/td-morning
~/.agents/skills/td-shutdown     → ~/dev/skills/todoist-skills/skills/td-shutdown

~/.claude/skills/td-gtd          → ../../.agents/skills/td-gtd
~/.claude/skills/td-morning      → ../../.agents/skills/td-morning
~/.claude/skills/td-shutdown     → ../../.agents/skills/td-shutdown

~/.local/share/todoist-skills/journal/                  user-state directory
```

**Why this split:**
- `_shared/` is referenced by every ritual; placing it as a sibling under `skills/` keeps related files together.
- Each skill folder contains only its `SKILL.md` — no additional scripts. Skill content is markdown-only; Claude executes via existing tools (Bash, MCP).
- Symlinks chain through `~/.agents/skills/` to match the established convention (`parsew-skills`, `todoist-cli`). Editing in the canonical bundle propagates everywhere.
- Journal is **outside** the bundle so the bundle can be installed read-only / shared across machines without polluting it with per-machine state.

---

## Task Ordering Rationale

The plan is ordered so each task can be verified before the next begins:

1. **Skeleton** before content (a place for files to live)
2. **Detection snippet** before the rituals that reference it
3. **`td-gtd`** before the rituals that Read it
4. **Rituals** in any order (independent)
5. **Symlinks + journal dir** before smoke testing
6. **Smoke tests** in Claude Code (the runtime we're in)
7. **Smoke test in Claude Desktop** (user runs manually)
8. **MCP capability inventory** (deferred until a real authed MCP session exists — final pre-ship task)

---

### Task 1: Bundle skeleton (.gitignore, README, dir structure)

**Files:**
- Create: `~/dev/skills/todoist-skills/.gitignore`
- Create: `~/dev/skills/todoist-skills/README.md`
- Create: `~/dev/skills/todoist-skills/skills/_shared/` (directory)
- Create: `~/dev/skills/todoist-skills/skills/td-gtd/` (directory)
- Create: `~/dev/skills/todoist-skills/skills/td-morning/` (directory)
- Create: `~/dev/skills/todoist-skills/skills/td-shutdown/` (directory)

- [ ] **Step 1: Create the directory tree**

```bash
mkdir -p ~/dev/skills/todoist-skills/skills/{_shared,td-gtd,td-morning,td-shutdown}
ls ~/dev/skills/todoist-skills/skills/
```

Expected: four directories listed (`_shared td-gtd td-morning td-shutdown`).

- [ ] **Step 2: Write `.gitignore`**

```
.DS_Store
*.swp
.idea/
.vscode/
node_modules/
```

Save to `~/dev/skills/todoist-skills/.gitignore`. Rationale: keep OS junk and editor metadata out; we have no build artifacts.

- [ ] **Step 3: Write `README.md`**

```markdown
# todoist-skills

A bundle of Claude skills wrapping a GTD + Kanban Todoist setup. Works in both Claude Code (`td` CLI) and Claude Desktop (Todoist MCP connector).

## Skills

- **`td-gtd`** — rules + system map + filter/label semantics. Read by the rituals; not a slash command.
- **`/td-morning`** — start-of-day ritual (~2-3 min). Locks in the deep_work block, surfaces client commitments.
- **`/td-shutdown`** — end-of-day ritual (~2-3 min). Cleans the board, logs own/client time split to the journal.

## Install

```bash
# Already cloned to ~/dev/skills/todoist-skills/. Then:
for s in td-gtd td-morning td-shutdown; do
  ln -s ~/dev/skills/todoist-skills/skills/$s ~/.agents/skills/$s
  ln -s ../../.agents/skills/$s ~/.claude/skills/$s
done
mkdir -p ~/.local/share/todoist-skills/journal
```

## Status

Phase 1 (MVP): instrumentation only. See `SPEC.md` and `PLAN.md`.
Phase 2 (next): `/td-weekly-review` reads the journal, closes the growth loop.
```

Save to `~/dev/skills/todoist-skills/README.md`.

- [ ] **Step 4: Verify the structure**

```bash
find ~/dev/skills/todoist-skills -type f -o -type d | sort
```

Expected output (modulo `.git/` internals):
```
~/dev/skills/todoist-skills
~/dev/skills/todoist-skills/.git
~/dev/skills/todoist-skills/.gitignore
~/dev/skills/todoist-skills/PLAN.md
~/dev/skills/todoist-skills/README.md
~/dev/skills/todoist-skills/SPEC.md
~/dev/skills/todoist-skills/skills
~/dev/skills/todoist-skills/skills/_shared
~/dev/skills/todoist-skills/skills/td-gtd
~/dev/skills/todoist-skills/skills/td-morning
~/dev/skills/todoist-skills/skills/td-shutdown
```

- [ ] **Step 5: Commit**

```bash
cd ~/dev/skills/todoist-skills
git add .gitignore README.md skills/
git commit -m "Scaffold bundle directory structure and README"
```

(Note: empty directories don't get committed by git. If you want them tracked before content lands, add a `.gitkeep` to each. Otherwise the directories will be committed implicitly when Task 2-5 add files inside them — fine either way.)

---

### Task 2: `_shared/environment.md` — detection snippet

**Files:**
- Create: `~/dev/skills/todoist-skills/skills/_shared/environment.md`

- [ ] **Step 1: Write the file with the full detection logic**

````markdown
# Environment detection

The skill calling this file MUST run these steps before any Todoist operation.

## Runtime detection

Determine the runtime by trying each in order:

1. **CLI mode** — Run `command -v td` via the Bash tool. If it returns a path, the `td` CLI is available. Use `td` for everything. Reference the `todoist-cli` skill (already installed at `~/.claude/skills/todoist-cli/SKILL.md`) for syntax.

2. **MCP mode (tools already visible)** — If no `td` CLI, scan the currently-visible tool names. If any name matches `^Todoist:` OR contains `Todoist` after an `mcp__` prefix (e.g., `mcp__claude_ai_Todoist__find-tasks-by-date`), the MCP connector is loaded. Proceed to "Capability discovery."

3. **MCP mode (tools deferred)** — Many runtimes hide MCP tool schemas until `tool_search` is called. Run `tool_search` with query `todoist` and `max_results: 20`. If results return, fetch their schemas, then re-check tool visibility. If now visible → proceed to "Capability discovery."

4. **No runtime** — If none of the above succeed, tell the user verbatim:

   > I need either the `td` CLI (Claude Code) or the Todoist MCP connector (Claude Desktop → Connectors menu → add Todoist). Stopping here. Once you've installed one, re-invoke me.

   Stop the skill. Do not proceed.

## Capability discovery (MCP mode only)

Once MCP tools are visible:

1. **Auth probe.** Find a tool whose name matches `*user-info*` or `*user*` or `*me*` (a lightweight identity check). Call it with no arguments. If it succeeds, you are authenticated — proceed. If it fails with an auth error or no such tool exists, fall through to "Auth bootstrap" below.

2. **Auth bootstrap.** Only the `authenticate` and `complete_authentication` tools are visible in the unauthed state. Tell the user:

   > Todoist MCP is installed but not authenticated. I'll trigger the auth flow now. Please complete the OAuth in your browser when it opens, then tell me when you're done.

   Then call the `authenticate` tool (its exact name will be discoverable via `tool_search`). Wait for the user to confirm browser completion. Call `complete_authentication`. Then re-run `tool_search` for `todoist` to surface the real tools. Re-run the auth probe.

3. **Operation → tool mapping.** Skills describe operations functionally ("list tasks for today"). Map each operation to whatever tool name best matches by pattern:

   | Operation | Match pattern |
   |---|---|
   | List tasks by filter | `*find*task*` or `*list*task*` with a filter/query argument |
   | List tasks by project | `*find*task*` or `*list*task*` with a project argument |
   | Add task | `*add*task*` or `*create*task*` |
   | Complete task | `*complete*task*` or `*close*task*` |
   | Update task (date, labels, priority, section) | `*update*task*` |
   | Move task | `*update*task*` with `project_id` / `section_id` |
   | List projects | `*list*project*` or `*find*project*` |
   | Get project | `*get*project*` |

   Cache the mapping in working memory for this turn so each operation doesn't re-search.

## Capability table

The `td-gtd` SKILL.md contains the populated capability table (filled in after a real MCP inventory). When acting, prefer reading that table; this file only describes the discovery mechanism.

## Performance note

In MCP mode, bulk operations (e.g., walking 🎯 Today + 5 filters) cost 5-10 tool calls. Skills should:
- Batch where possible (one MCP call that lists all tasks beats five filtered calls).
- Warn explicitly if the user is in MCP mode and an operation will be slow: "Running this in Desktop takes ~20s; for daily speed, prefer Claude Code."
- Offer the fast-path variant of `/td-morning` for slow-runtime days.
````

Save to `~/dev/skills/todoist-skills/skills/_shared/environment.md`.

- [ ] **Step 2: Verify file exists and is non-empty**

```bash
wc -l ~/dev/skills/todoist-skills/skills/_shared/environment.md
```

Expected: ≥40 lines.

- [ ] **Step 3: Commit**

```bash
cd ~/dev/skills/todoist-skills
git add skills/_shared/environment.md
git commit -m "Add _shared/environment.md detection snippet"
```

---

### Task 3: `td-gtd/SKILL.md` — rules doc

**Files:**
- Create: `~/dev/skills/todoist-skills/skills/td-gtd/SKILL.md`

The system map / filter map / labels data comes verbatim from `/Users/chris/Downloads/todoist-migration/04-execution-log.md`. Cross-reference it while writing.

- [ ] **Step 1: Write the frontmatter + opening**

```markdown
---
name: td-gtd
description: Rules, system map, growth contract, and capability reference for the user's Todoist GTD setup. Read by /td-morning and /td-shutdown. Not invoked directly.
---

# td-gtd — rules reference

This file is **Read by the ritual skills**, not invoked as a slash command. It is the single source of truth for the user's Todoist structure, filters, labels, growth contract, and CLI-vs-MCP capability differences.

When a ritual skill (`/td-morning`, `/td-shutdown`) starts, it reads this file once and applies the rules below for the rest of the turn.
```

- [ ] **Step 2: Add the system map section**

````markdown
## 1. System map

Top-level projects and their IDs. **Validation hint**: if any of these IDs returns 404 when accessed, halt and tell the user "Project ID for `<name>` no longer resolves — please re-run discovery."

| Project | ID | View | Notes |
|---|---|---|---|
| Inbox | (system) | list | Only valid landing zone for unclarified items |
| LangRelay | `6gg3XRqxXPWPMwH7` | board | The big bet. Area sections, not kanban. |
| Selektable | `6gg3XV44fmp4qp6j` | board | User's company. Kanban sections. |
| Clients | `6gg3gv5f4fj5M46g` | list | Parent — no tasks of its own |
| ├─ Leat | `6gg3XVMc99VMGxMV` | board | Active client |
| ├─ Hairgivers | `6gg3XVq73jRXwxHR` | board | Active client |
| └─ Mediastap | `6gg3XVQrrCgRV9Qv` | board | Active client (sections: WP Content Agent, Wisa Export Plugin) |
| Side Projects | `6gg3hr9xM2F69qq7` | list | Parent — no tasks of its own |
| ├─ Parsew | `6gg3XV25WQFjCWwQ` | board | Solo SDK |
| ├─ Kopplio | `6gg3XV4C9xjPpjM2` | board | Stripe→Moneybird |
| ├─ Landing Gallery | `6gg3XW32cC6MRwgG` | board | Inspiration gallery |
| ├─ AutomagicWP | `6gg3XVGJjR3w6hJq` | board | WP distribution |
| ├─ EmitKit | `6gg3XV95224CRHJF` | board | Event kit |
| └─ Job Boards | `6gg3hrH2JQQx74QV` | list | Sub-parent (under Side Projects) |
|     ├─ JobBoardStarter | `6gg3XV8P3CM7RWQH` | board | |
|     └─ Fashion Workplace | `6gg3XVxhfgWC559g` | board | |
| 🔧 Admin | `6gg3XW67r44qWM45` | board | Business admin |
| 🏠 Personal | `6gg3XWCqFVr6rgXP` | board | Non-work tasks |
| ⏳ Waiting For | `6gg3XWGPrg27g2QP` | list | Anything blocked on someone else |
| 💭 Someday / Maybe | `6gg3XWMfpCwFjr7X` | list | Parked ideas |

**LangRelay sections** (area-based, not kanban): `🔥 In Progress`, `Visual Editor`, `WP Plugin`, `Switcher & Translations`, `Other`.

**Standard kanban sections** (used by every board project except LangRelay and Mediastap): `Backlog`, `This Week`, `Doing`, `Review`, `Done`.

**Mediastap sections**: `WP Content Agent`, `Wisa Export Plugin`.
````

- [ ] **Step 3: Add the filter map section**

````markdown
## 2. Filter map

| Filter | Query | Intent |
|---|---|---|
| 🎯 Today | `(today \| overdue) & !@waiting` | Single source of truth for "what am I doing today" |
| 🧠 Deep work | `@deep_work & (today \| overdue \| no date) & (p1 \| p2)` | Pick when you have a 90-min block |
| ⚡ Quick wins | `@quick & !@waiting` | Between-meetings windows |
| 🤖 Agent queue | `@agent & !@waiting` | Tasks delegatable to AI agents |
| ⏳ Waiting check | `@waiting` | Anything blocked on someone else |
| ❓ Unclarified | `no date & !#"💭 Someday / Maybe" & !@waiting` | Backlog hygiene (over-inclusive in MVP; see SPEC follow-up) |
| 📅 This week | `(7 days \| overdue) & !@waiting` | What's queued for the coming week |
````

- [ ] **Step 4: Add labels & semantics**

````markdown
## 3. Labels & semantics

Labels are context (HOW the work happens), not category (WHAT it's about). Category is the project.

| Label | When to apply |
|---|---|
| `@deep_work` | Needs a 90+ min uninterrupted block. Honest application — not "important." |
| `@quick` | Fits in 5-15 min. Doesn't need deep focus. |
| `@errand` | Requires leaving the desk |
| `@calls` | Phone call required |
| `@email` | Outbound email or response |
| `@waiting` | Blocked on someone else (also lands on ⏳ Waiting For project for parking) |
| `@two_minutes` | GTD 2-min rule — do immediately if encountered |
| `@agent` | Delegatable to an AI agent without supervision |
| `@review` | You're a reviewer/approver, not the doer |
````

- [ ] **Step 5: Add the growth contract**

````markdown
## 4. Growth contract

Source: SPEC.md, section "Growth contract."

- **Baseline (2026-05-17)**: ~€6500/mo client revenue, 0% own-product revenue, ~85% client time.
- **6-month target (2026-11-17)**: 70% client / 30% own work (time + revenue).
- **12-month target (2027-05-17)**: 50% client / 50% own work.
- **Primary growth bet**: LangRelay. When `/td-morning` proposes a deep_work block, default to LangRelay unless it has no actionable task.
- **Containment style**: soft. When a future `/td-capture` lands client work that pushes the user over a weekly cap, the skill notes it — never refuses.
- **Ratchet**: instrumented now (`/td-shutdown` writes the journal). Activated in Phase 2 when `/td-weekly-review` reads the journal, computes the rolling own/client ratio against the target, and names the corrective action.

**MVP enforcement** (limited by design):
- `/td-morning` defaults the deep_work proposal to LangRelay.
- `/td-morning` flags client-heavy days.
- Nothing else. Stronger defense lands in Phase 2.
````

- [ ] **Step 6: Add capture conventions, date discipline, ask-vs-act**

````markdown
## 5. Capture conventions

- Inbox is the **only** valid landing zone for unclarified items.
- Every other task must have: project + (date OR `💭 Someday / Maybe`).
- No floating tasks anywhere outside Inbox.

## 6. Date discipline

- Date = commitment to do it that day. Not sort order.
- For "soon but not today" use the `This Week` section on the project board AND optionally the `📅 This week` filter.
- Reschedule freely — re-dating is the close-of-day move, not a failure mode.

## 7. When to ask vs. when to act

- **Clear routing** → act and confirm in the same step: "Moved Vendomat to Leat / This Week, due tomorrow. Undo?"
- **Ambiguous routing** → one question only: "Where does Vendomat go — Leat or somewhere else?"
- **Bulk** → batch into one question: "These 5 look like agent tasks — confirm all, or pick the exceptions?"

Never ask three questions in a row. If three would be needed, propose a default for all and let the user override the ones that are wrong.
````

- [ ] **Step 7: Add the capability table (placeholder until inventory completes)**

````markdown
## 8. Capability table (CLI vs MCP)

Filled in by Task 10 of `PLAN.md` after a real MCP inventory. Until then, the rituals fall back to discovery via `_shared/environment.md`.

| Operation | CLI mode (`td`) | MCP mode (tool name) | Degradation in MCP |
|---|---|---|---|
| List tasks by filter | `td filter view "<name>" --json` | _TBD_ | _TBD_ |
| List tasks by project | `td task list --project "id:<id>" --json` | _TBD_ | _TBD_ |
| Add task | `td task add <flags>` / `td task quickadd "<text>"` | _TBD_ | _TBD_ |
| Complete task | `td task complete "id:<id>"` | _TBD_ | _TBD_ |
| Update task | `td task update "id:<id>" <flags>` | _TBD_ | _TBD_ |
| Move task | `td task move "id:<id>" --project "id:<id>" --section "id:<id>"` | _TBD_ | _TBD_ |
| List projects | `td project list --json` | _TBD_ | _TBD_ |

**Task 10 of `PLAN.md`** populates this table. Until then, in MCP mode the rituals will fall back to runtime `tool_search` discovery on every operation (slower, but functional).
````

- [ ] **Step 8: Add the journal schema section**

````markdown
## 9. Journal schema

Written by `/td-shutdown` to `~/.local/share/todoist-skills/journal/YYYY-MM-DD.md`. One file per day. The first line is canonical (machine-parseable); anything below is free-text.

```
2026-05-17 | type:workday | own:2h | client:5h | note: shipped CJ-40
```

Fields:

- `type:` — `workday | partial | off`. Mandatory. `off` days are excluded from ratio math.
- `own:` — rough hours on own-work. Optional on `partial`, omitted on `off`.
- `client:` — rough hours on client work. Optional on `partial`, omitted on `off`.
- `note:` — optional free-text.

**Concurrency rule** (`/td-shutdown`): if today's file already exists, **read-modify-write** the canonical first line. Preserve any free-text body. Never blind-append.
````

- [ ] **Step 9: Verify total length and commit**

```bash
wc -l ~/dev/skills/todoist-skills/skills/td-gtd/SKILL.md
cd ~/dev/skills/todoist-skills
git add skills/td-gtd/SKILL.md
git commit -m "Add td-gtd SKILL.md (rules, system map, growth contract)"
```

Expected: 150-250 lines, mostly tables.

---

### Task 4: `td-morning/SKILL.md` — start-of-day ritual

**Files:**
- Create: `~/dev/skills/todoist-skills/skills/td-morning/SKILL.md`

- [ ] **Step 1: Write the frontmatter + intro**

```markdown
---
name: td-morning
description: Start-of-day Todoist ritual (~2-3 min). Pulls 🎯 Today + 📅 This week, triages stale carry-overs, locks in a LangRelay deep_work block, surfaces client commitments, suggests quick wins. Has a fast-path variant for low-friction days.
---

# /td-morning

A 2-3 minute ritual to lock in the day. Biases toward LangRelay for the deep_work slot — that's the user's primary growth bet, and the system actively defends it.

**Invoked by the user:** `/td-morning` (default flow) or `/td-morning fast` (fast-path variant — see end of file).
```

- [ ] **Step 2: Write the prelude (read shared files, detect environment)**

````markdown
## Prelude (runs every time)

1. Resolve this skill's directory. The bundle root is two levels up from this file:
   `<this-file>/../../` → `~/dev/skills/todoist-skills/skills/` → `~/dev/skills/todoist-skills/`.

2. Use the Read tool to load `<bundle>/skills/td-gtd/SKILL.md`. Internalize the system map, filter map, label semantics, growth contract, and capability table.

3. Use the Read tool to load `<bundle>/skills/_shared/environment.md`. Run the runtime detection. If detection halts, stop here.

4. If MCP mode and unauthed → run the auth bootstrap from `_shared/environment.md`. Do not proceed until authenticated.

5. Once detection completes, you have a runtime (CLI or MCP) and an operation→tool mapping. Proceed.
````

- [ ] **Step 3: Write the default flow**

````markdown
## Default flow

Run these steps in order. If anything errors (e.g., a filter doesn't exist), stop and report — do not silently skip.

### Step 1: Pull state (batched)

In **CLI mode**, run these as parallel Bash calls if possible:

```bash
td filter view "🎯 Today" --json
td filter view "📅 This week" --json
td filter view "⏳ Waiting check" --json
td task list --project "id:6gg3XRqxXPWPMwH7" --section "🔥 In Progress" --json
td task list --project "id:6gg3XRqxXPWPMwH7" --section "This Week" --json
```

In **MCP mode**, perform equivalent calls one by one (use the operation→tool mapping). Warn the user upfront: "Pulling state in MCP mode — this takes ~15s."

### Step 2: Triage 🎯 Today

For each task in 🎯 Today:
- **Stale carry-over** (due >1 day ago, still on Today): "X is from N days ago — still today, or push?" Accept: today / tomorrow / This Week / Someday.
- **Vague** (one-word title, no label, no description): "Y has no context — clarify now, or send to Inbox?"
- **Clear and current** → no action.

Batch the questions: if 3+ tasks need triage, present them as a list and accept a single batched response ("push all, except Z which is today").

### Step 3: Lock the deep_work slot

This is the load-bearing step. Default = LangRelay.

1. From the LangRelay pull (Step 1), pick the strongest candidate in this order:
   - Anything in `🔥 In Progress` (continue what's started)
   - p1 or p2 tasks in `This Week`
   - Any task in `This Week`
   - If nothing in `This Week` → "LangRelay's This Week is empty. Promote one from Backlog now, shift to Selektable, or skip the LangRelay default?"

2. Propose: "Deep work block: **<task title>** (~90 min). OK, or pick another?"

3. If user rejects: ask one diagnostic question — "What kind of day is this? client-heavy / admin / rest / something else?" Adjust the proposal once based on the answer, then move on.

### Step 4: Client commitments check

For each active client (Leat, Hairgivers, Mediastap), list overdue or due-today tasks. Use CLI:

```bash
td task list --project "id:6gg3XVMc99VMGxMV" --filter "(today | overdue)" --json
td task list --project "id:6gg3XVq73jRXwxHR" --filter "(today | overdue)" --json
td task list --project "id:6gg3XVQrrCgRV9Qv" --filter "(today | overdue)" --json
```

(MCP mode: equivalent tool calls.)

If the total client load already looks ≥4h (sum task `duration` values where present, or estimate 1h per task without a duration), flag:

> Heavy client day (~Nh estimated). The deep_work block on **<task>** may get squeezed — move it to morning?

### Step 5: Quick wins shortlist

Pull `⚡ Quick wins` and list 2-3 candidates:

```bash
td filter view "⚡ Quick wins" --json
```

Just present them — no action required. They're visible for between-block windows.

### Step 6: Output the 3-line plan

```
Today: deep_work on <task>.
Client commitments: <a>, <b>.
Quick wins available: <p>, <q>, <r>.
```

Skill exits here.
````

- [ ] **Step 4: Write the fast-path variant**

````markdown
## Fast-path variant (`/td-morning fast`)

Triggered when the user invokes `/td-morning fast` OR says some form of "quick version" / "just the basics."

1. Run the Prelude (still required — must detect runtime).
2. Pull `🎯 Today` AND the top LangRelay `🔥 In Progress` task (one CLI call each, or two MCP calls). Skip everything else.
3. Show: `🎯 Today (N items): [...]. Proposed deep_work: <LangRelay task>.`
4. User confirms or rejects. If rejected, accept a one-line override ("do Selektable demo today instead"). No follow-up questions.
5. Skill exits.

Total runtime in CLI: <2 seconds. In MCP: ~5 seconds. Use this on client-emergency days.
````

- [ ] **Step 5: Write the failure modes section**

````markdown
## Failure modes (handled in-skill)

- **Empty 🎯 Today** — don't nag. Default to "Pick 3-5 from `This Week` to date for today" with a multi-select prompt.
- **Skipped yesterday, many overdue items** — batch them into Step 2 as a single "These 8 are overdue: bulk-push to This Week, or pick which to keep on Today?" prompt.
- **MCP >10s** — show a "running in MCP mode, hold on" message before the user thinks the skill is hung.
- **Filter doesn't exist** — halt and tell the user: "Filter `<name>` not found. Was it renamed? Run /td-discovery (future) or re-check `td-gtd` system map."
````

- [ ] **Step 6: Verify and commit**

```bash
wc -l ~/dev/skills/todoist-skills/skills/td-morning/SKILL.md
cd ~/dev/skills/todoist-skills
git add skills/td-morning/SKILL.md
git commit -m "Add /td-morning ritual SKILL.md"
```

Expected: ~100-150 lines.

---

### Task 5: `td-shutdown/SKILL.md` — end-of-day ritual

**Files:**
- Create: `~/dev/skills/todoist-skills/skills/td-shutdown/SKILL.md`

- [ ] **Step 1: Write the frontmatter + intro**

```markdown
---
name: td-shutdown
description: End-of-day Todoist ritual (~2-3 min). Cleans 🎯 Today, captures own/client time split + day-type into the journal, surfaces tomorrow's anchor and waiting-for nudges.
---

# /td-shutdown

A 2-3 minute close. Two jobs: leave Todoist clean for tomorrow, and capture today's signal for Phase 2's weekly review.

**Invoked by the user:** `/td-shutdown`.
```

- [ ] **Step 2: Write the prelude (same as morning)**

````markdown
## Prelude (runs every time)

1. Resolve this skill's directory. The bundle root is two levels up.
2. Read `<bundle>/skills/td-gtd/SKILL.md`. Internalize rules, growth contract, and journal schema (section 9).
3. Read `<bundle>/skills/_shared/environment.md`. Run detection. Halt if no runtime.
4. If MCP and unauthed → auth bootstrap.
5. Proceed.
````

- [ ] **Step 3: Write the flow**

````markdown
## Flow

### Step 1: Walk today

Pull `🎯 Today` AND tasks completed today:

```bash
td filter view "🎯 Today" --json
td completed list --since "$(date -u +%Y-%m-%dT00:00:00Z)" --json
```

For each task on `🎯 Today` (excluding ones already completed):

- **Done?** → confirm completion. If it's a one-shot, complete it. If it should recur, ask: "Recur weekly/daily/monthly, or one-shot?" Apply via `td task update` or MCP equivalent.
- **Started but not done?** → drag to `Doing` section on its project board:
  ```bash
  td task move "id:<task>" --section "Doing"
  ```
  Then ask: "Keep dated today, or push to tomorrow / This Week / unschedule?"
- **Untouched?** → "Reschedule (when?), move back to `This Week`, or move to `💭 Someday / Maybe`?"

Batch the prompts: if 5+ tasks fall in the same bucket, present as a list with bulk-actions ("push all to This Week, except Y which I want tomorrow").

### Step 2: Capture loose ends

Ask exactly one prompt:

> Anything in your head that didn't make it into Todoist today?

For each item the user mentions, route to Inbox via:

```bash
td task quickadd "<user's text>"
```

(MCP: equivalent add-task tool, no project specified → defaults to Inbox.)

If user says "nothing" / "no" / silence → skip.

### Step 3: Day type + own/client split

Two prompts, in order:

**Prompt 1:** "Day type — workday, partial, or off?"
- `off` → skip Prompt 2. Set `own:` and `client:` to omitted in the journal line.
- `partial` → Prompt 2 is optional ("hours, or skip?"). Allow either both fields, neither, or one.
- `workday` → Prompt 2 is asked.

**Prompt 2:** "Rough split — own/client hours? (e.g., '2h own / 5h client' or '3 client, no own' or 'skip')."

Parse the response. Accept any of these forms:
- `"2h own / 5h client"` → `own:2h client:5h`
- `"all client, 6h"` → `own:0h client:6h`
- `"3 own"` → `own:3h client:0h`
- `"skip"` → omit both fields from the journal line

### Step 4: Write the journal entry

Journal path: `~/.local/share/todoist-skills/journal/YYYY-MM-DD.md`.

Canonical line format (see `td-gtd` section 9):

```
2026-05-17 | type:workday | own:2h | client:5h | note: <optional>
```

**Read-modify-write semantics**:

1. Check if the file exists.
2. If yes → read it. Replace the first line (which must match the regex `^\d{4}-\d{2}-\d{2} \|`). Preserve everything after the first line verbatim.
3. If no → write a new file with just the canonical line.

Reference Bash recipe (CLI mode; same logic in MCP mode using whatever filesystem tool is available):

```bash
JOURNAL_DIR=~/.local/share/todoist-skills/journal
mkdir -p "$JOURNAL_DIR"
DATE=$(date -u +%Y-%m-%d)
FILE="$JOURNAL_DIR/$DATE.md"
NEW_LINE="$DATE | type:workday | own:2h | client:5h | note: shipped CJ-40"

if [ -f "$FILE" ]; then
  # Replace first line, preserve rest
  tail -n +2 "$FILE" > "$FILE.tmp"
  echo "$NEW_LINE" > "$FILE"
  cat "$FILE.tmp" >> "$FILE"
  rm "$FILE.tmp"
else
  echo "$NEW_LINE" > "$FILE"
fi
```

Skip writing the note suffix if the user gave none (omit `| note: ...`).

### Step 5: Tomorrow setup (optional)

Check tomorrow's `🎯 Today` count and current `📅 This week`:

```bash
td filter view "🎯 Today" --filter "tomorrow" --json
td filter view "📅 This week" --json
```

If tomorrow's count is 0 AND `📅 This week` has ≥3 items: ask once:

> Want to pick tomorrow's anchor while it's fresh? (Or skip — fine to do it in /td-morning.)

If yes, present the top 3-5 items from `📅 This week` and let the user pick one. Set its due date to tomorrow:

```bash
td task reschedule "id:<task>" tomorrow
```

If skipped or "no" → move on.

### Step 6: Waiting nudge

Pull `⏳ Waiting check`:

```bash
td filter view "⏳ Waiting check" --json
```

For each item, check its creation date (or last-update if available). If >7 days old: list it and ask "Nudge tomorrow?" — if yes, add a task to Inbox: `td task quickadd "Nudge <person/thing> on <waiting item>" --due tomorrow`. If no → skip.

### Step 7: Output the 2-line close

```
Closed: <N> done, <M> pushed, <K> new in Inbox. Day: workday | own:2h client:5h. Tomorrow's anchor: <task or "none">.
```

Skill exits.
````

- [ ] **Step 4: Write failure modes**

````markdown
## Failure modes (handled in-skill)

- **Skipped 3 days** — first prompt: "Last entry was N days ago. Log the last N days now, or skip ahead?" If "log": loop the day-type + split prompts for each missing day (use yesterday/2 days ago/etc. as the journal date). If "skip": just do today.
- **User gives ambiguous split** ("a few hours own, mostly client") — accept it as a fuzzy note: write `own:fuzzy client:fuzzy | note: <quote>`. Phase 2's parser will ignore fuzzy days for ratio math.
- **Journal file write fails** (permission denied, disk full) — tell the user the path that failed and ask if they want to paste the line manually elsewhere. Don't halt the rest of the shutdown.
- **Filesystem write tool unavailable in MCP mode** — fall back to: "Here's today's journal line — paste it manually into `~/.local/share/todoist-skills/journal/YYYY-MM-DD.md`."
````

- [ ] **Step 5: Verify and commit**

```bash
wc -l ~/dev/skills/todoist-skills/skills/td-shutdown/SKILL.md
cd ~/dev/skills/todoist-skills
git add skills/td-shutdown/SKILL.md
git commit -m "Add /td-shutdown ritual SKILL.md"
```

Expected: ~150-200 lines.

---

### Task 6: Install symlinks

**Files:**
- Create: `~/.agents/skills/td-gtd` (symlink)
- Create: `~/.agents/skills/td-morning` (symlink)
- Create: `~/.agents/skills/td-shutdown` (symlink)
- Create: `~/.claude/skills/td-gtd` (symlink)
- Create: `~/.claude/skills/td-morning` (symlink)
- Create: `~/.claude/skills/td-shutdown` (symlink)

- [ ] **Step 1: Verify no conflicting names exist**

```bash
ls -la ~/.agents/skills/ | grep -E "td-gtd|td-morning|td-shutdown" || echo "no conflicts"
ls -la ~/.claude/skills/ | grep -E "td-gtd|td-morning|td-shutdown" || echo "no conflicts"
```

Expected: `no conflicts` for both. (`todoist-cli` already exists but that's a different name.) If anything matches, stop and resolve (rename or delete the existing entry).

- [ ] **Step 2: Create the three `~/.agents/skills/` symlinks**

```bash
ln -s ~/dev/skills/todoist-skills/skills/td-gtd ~/.agents/skills/td-gtd
ln -s ~/dev/skills/todoist-skills/skills/td-morning ~/.agents/skills/td-morning
ln -s ~/dev/skills/todoist-skills/skills/td-shutdown ~/.agents/skills/td-shutdown
ls -la ~/.agents/skills/td-gtd ~/.agents/skills/td-morning ~/.agents/skills/td-shutdown
```

Expected: each line shows `→ /Users/chris/dev/skills/todoist-skills/skills/<name>`.

- [ ] **Step 3: Create the three `~/.claude/skills/` symlinks (relative target, matches existing convention)**

```bash
cd ~/.claude/skills
ln -s ../../.agents/skills/td-gtd td-gtd
ln -s ../../.agents/skills/td-morning td-morning
ln -s ../../.agents/skills/td-shutdown td-shutdown
ls -la td-gtd td-morning td-shutdown
```

Expected: each line shows `→ ../../.agents/skills/<name>`.

- [ ] **Step 4: Verify the full chain resolves**

```bash
readlink -f ~/.claude/skills/td-gtd
readlink -f ~/.claude/skills/td-morning
readlink -f ~/.claude/skills/td-shutdown
```

Expected: each resolves to `/Users/chris/dev/skills/todoist-skills/skills/<name>`.

- [ ] **Step 5: Verify the SKILL.md files are readable through the chain**

```bash
head -5 ~/.claude/skills/td-gtd/SKILL.md
head -5 ~/.claude/skills/td-morning/SKILL.md
head -5 ~/.claude/skills/td-shutdown/SKILL.md
```

Expected: each shows the frontmatter (`---`, `name:`, `description:`, `---`).

No commit — symlinks live outside the bundle.

---

### Task 7: Create the user-state journal directory

**Files:**
- Create: `~/.local/share/todoist-skills/journal/` (directory only)

- [ ] **Step 1: Create the directory**

```bash
mkdir -p ~/.local/share/todoist-skills/journal
ls -la ~/.local/share/todoist-skills/
```

Expected: the `journal` directory exists.

- [ ] **Step 2: Verify writability**

```bash
touch ~/.local/share/todoist-skills/journal/.test && rm ~/.local/share/todoist-skills/journal/.test && echo "writable"
```

Expected: `writable`.

No commit — user state lives outside the bundle.

---

### Task 8: Smoke test in Claude Code (CLI mode)

This task is the "test" for the previous tasks. It runs the three skills end-to-end in this runtime (Claude Code, where `td` CLI is available) and verifies basic behavior.

- [ ] **Step 1: Verify the skills are discoverable**

In the Claude Code session, invoke help on each:

```
/td-morning --help
```

(Or, if `--help` isn't recognized: just type the slash command at the prompt and see if Claude recognizes and offers to run it.)

Expected: Claude recognizes the skill and either runs it or describes it.

If Claude reports "no such skill" → re-check the symlinks (Task 6). The symlink chain may not be resolving; try running `ls -la ~/.claude/skills/td-morning/SKILL.md`.

- [ ] **Step 2: Run `/td-morning` end-to-end**

Invoke:

```
/td-morning
```

Expected behavior (in order):
1. Claude reads `td-gtd/SKILL.md` and `_shared/environment.md`.
2. Detects CLI mode (`td` is on PATH).
3. Runs the 5 `td filter view` / `td task list` calls from Default flow Step 1.
4. Walks through triage / deep_work / client commitments / quick wins / 3-line plan.

If something errors, capture the error and check:
- Filter names match exactly (emoji included)
- Project IDs in `td-gtd` resolve (run `td project view "id:6gg3XRqxXPWPMwH7"` directly)
- Section names match (`🔥 In Progress`, `This Week`)

- [ ] **Step 3: Run `/td-shutdown` end-to-end**

Invoke:

```
/td-shutdown
```

Expected behavior:
1. Reads shared files.
2. Walks 🎯 Today + completed-today.
3. Asks loose-ends, day-type, split.
4. Writes to `~/.local/share/todoist-skills/journal/YYYY-MM-DD.md`.
5. Outputs the 2-line close.

Verify the journal file:

```bash
ls ~/.local/share/todoist-skills/journal/
cat ~/.local/share/todoist-skills/journal/$(date -u +%Y-%m-%d).md
```

Expected: one file exists; its first line matches `^\d{4}-\d{2}-\d{2} \| type:`.

- [ ] **Step 4: Verify read-modify-write on a second invocation**

Re-run `/td-shutdown` in the same session, give different hours (e.g., `"3h own, 4h client"`).

```bash
cat ~/.local/share/todoist-skills/journal/$(date -u +%Y-%m-%d).md
```

Expected: only ONE line (the new one). Old line replaced, not appended.

If two lines appear: the read-modify-write logic in Task 5 Step 4 isn't being followed. Re-read `td-shutdown/SKILL.md` Step 4 and fix.

- [ ] **Step 5: Verify `/td-morning fast`**

Invoke:

```
/td-morning fast
```

Expected: runs in ≤5 seconds, shows 🎯 Today contents + one LangRelay proposal, then exits after one confirm/reject.

---

### Task 9: Smoke test in Claude Desktop (MCP mode) — user-run

This is a manual user task. The plan-executor cannot do this; the user opens Claude Desktop and verifies.

- [ ] **Step 1: Open Claude Desktop. Confirm Todoist connector is installed and authenticated.**

If not: Settings → Connectors → add Todoist → complete OAuth.

- [ ] **Step 2: Verify the skills appear**

Type `/td-morning` in Claude Desktop. Expected: Claude recognizes the slash command (autocompletes or describes it).

If not recognized: Claude Desktop on macOS reads from `~/.claude/skills/`. Verify the symlinks from Task 6 resolve correctly on this filesystem. (On macOS the path is identical to Claude Code, so this should just work — but verify.)

- [ ] **Step 3: Run `/td-morning` in Claude Desktop**

Expected behavior:
1. Reads shared files (same Read tool exists in Desktop).
2. Detection: no `td` binary visible to Desktop's Bash → falls through to MCP scan.
3. MCP tools may be deferred; expect a `tool_search` call.
4. Once tools are visible, runs operation discovery from `_shared/environment.md`.
5. Performs the same flow as in CLI mode but with more tool calls (slower).

Expected runtime: 10-30s in Desktop vs <5s in CLI.

If the auth probe fails: the auth bootstrap flow runs. Complete OAuth, re-run.

- [ ] **Step 4: Run `/td-shutdown` in Claude Desktop**

Expected: identical flow to CLI mode. Verify journal file is written:

```bash
# From a terminal (not in Claude Desktop):
cat ~/.local/share/todoist-skills/journal/$(date -u +%Y-%m-%d).md
```

If Claude Desktop lacks filesystem write capability through MCP: the fallback in Task 5 Step 4 triggers ("Here's the line, paste it manually"). User pastes the line into the file via a terminal or editor. This is acceptable for MVP — note it as a Phase 1 limitation.

- [ ] **Step 5: Report findings back to the bundle**

Open `~/dev/skills/todoist-skills/SPEC.md` and add a final section:

```markdown
## MVP smoke-test results (2026-05-XX)

- Claude Code: ✓ /td-morning runs in <5s, /td-shutdown writes journal correctly.
- Claude Desktop: <Pass/Fail per ritual, with notes on observed latency and any fallbacks triggered>.
```

Commit the addition.

---

### Task 10: MCP capability inventory — populate `td-gtd` section 8

This task requires an authenticated MCP session to inspect available tools and their schemas. It can run from either Claude Code (after authenticating the Todoist MCP) or Claude Desktop.

- [ ] **Step 1: Ensure authed MCP session**

From within Claude Desktop (preferred, where the MCP is usually authed) or Claude Code (auth via the bootstrap tools if needed).

Verify by calling `tool_search` for `todoist` and confirming non-auth tools are visible.

- [ ] **Step 2: Inventory tools**

Run `tool_search` with query `todoist` and `max_results: 30`. Capture the full result.

For each tool, note:
- Exact tool name
- One-line summary of what it does
- Key parameters (which are required, what types)

- [ ] **Step 3: Map operations to tool names**

For each operation in the `td-gtd` capability table (section 8), identify the matching tool. Note the degradation if the tool is missing or weaker than the CLI equivalent.

Example expected output:

| Operation | CLI mode (`td`) | MCP mode (tool name) | Degradation in MCP |
|---|---|---|---|
| List tasks by filter | `td filter view "<name>" --json` | `mcp__claude_ai_Todoist__find-tasks-by-filter` | None |
| Add task | `td task add` / `td task quickadd "<text>"` | `mcp__claude_ai_Todoist__add-task` | No `quickadd` natural-language parse; must specify project/labels explicitly |
| Move task between sections | `td task move "id:<id>" --section "id:<id>"` | `mcp__claude_ai_Todoist__update-task` (set `section_id`) | One call vs one call; no degradation |
| Bulk reschedule | One Bash loop | Loop of update-task calls | N tool calls vs 1 — slow |

- [ ] **Step 4: Update `td-gtd/SKILL.md` section 8**

Replace the placeholder capability table with the real, populated table. Commit:

```bash
cd ~/dev/skills/todoist-skills
git add skills/td-gtd/SKILL.md
git commit -m "Populate td-gtd capability table from real MCP inventory"
```

- [ ] **Step 5: Update PLAN.md to mark this task complete and tag the release**

```bash
cd ~/dev/skills/todoist-skills
git tag -a v0.1.0-mvp -m "MVP: td-gtd + /td-morning + /td-shutdown shipped"
git tag
```

---

## Self-review

(Run by the plan author at write time. Findings inline.)

**Spec coverage:**
- Bundle structure (`SPEC.md` "Bundle structure") → Task 1.
- `td-gtd` content (SPEC.md "Skill: `td-gtd`") → Task 3.
- `/td-morning` content (SPEC.md "Skill: /td-morning") → Task 4.
- `/td-shutdown` content (SPEC.md "Skill: /td-shutdown") → Task 5.
- Environment detection (SPEC.md "Cross-cutting") → Task 2.
- Read-mechanism for rules (SPEC.md "How the rules doc is actually loaded") → Tasks 4 & 5 prelude steps.
- Journal in XDG path (SPEC.md "Persistence") → Tasks 5 & 7.
- Day-type field (SPEC.md "Persistence") → Task 5 Step 3.
- Read-modify-write concurrency (SPEC.md "Persistence: concurrency rule") → Task 5 Step 4 + Task 8 Step 4.
- Auth bootstrap (SPEC.md "Capability discovery") → Task 2 `environment.md` content; verified in Task 9.
- Symlink install (SPEC.md "Bundle structure") → Task 6.
- MCP capability table fill-in (SPEC.md "Capability discovery", "Open questions deferred") → Task 10.
- Phase 1 vs Phase 2 split (SPEC.md "Purpose") → captured in `td-gtd` section 4 (growth contract) + README status.

No gaps spotted.

**Placeholder scan:**
- "Filled in by Task 10" inside `td-gtd` section 8 — this is **deliberate** and the table has actual placeholder content showing the structure. The replacement is a defined task. Acceptable.
- No "TBD", "add appropriate", "similar to Task N", or "implement later" in any step. Each code block has real content.

**Type consistency:**
- Project IDs match `/Users/chris/Downloads/todoist-migration/04-execution-log.md` throughout.
- Filter names (with emojis) match `td-gtd` section 2 throughout.
- Journal field names (`type:`, `own:`, `client:`, `note:`) consistent across `td-gtd` section 9, `td-shutdown` Step 3-4, and Task 8 verification.
- "Read-modify-write" terminology consistent (SPEC, `td-gtd`, `td-shutdown`).

Plan is internally consistent. Ready for execution.
