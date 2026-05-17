# todoist-skills Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a 3-skill bundle (`td-gtd`, `/td-morning`, `/td-shutdown`) that supports a daily GTD ritual in both Claude Code (`td` CLI) and Claude Desktop (Todoist MCP connector), with a journal-based instrumentation layer that Phase 2's weekly review will consume.

**Architecture:** Single git repo at `~/dev/skills/todoist-skills/`. Three skill folders + one `_shared/` folder under `skills/`. Each individual skill is symlinked into `~/.agents/skills/<name>` which is in turn symlinked into `~/.claude/skills/<name>`. The ritual skills `Read` the rules doc (`td-gtd`) and detection snippet (`_shared/environment.md`) at invocation time. Journal lives at `~/.local/share/todoist-skills/journal/` — outside the bundle.

**Tech Stack:** Markdown only. No code beyond what skills instruct Claude to run via Bash / MCP tools. Source-of-truth: `~/dev/skills/todoist-skills/SPEC.md` (commit `594b737` or later).

---

## Revision history

- **v1** — Initial plan (commit `ecffe40`).
- **v2** (this) — Folds in 20 review findings: data-loss-safe journal recipe; mkdir prereqs; README/Task-6 install parity; `td` syntax verified against `todoist-cli` skill; client-load heuristic simplified; recursive-invocation problem in smoke test fixed; slash-arg fast-path removed (natural-language only); local-time dates; section-by-ID; project-ID preflight; auth-decline escape; honest test phasing; TESTING.md split from SPEC.md.

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
├── TESTING.md                                 (created in Task 9)
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
- `_shared/` is referenced by every ritual; keeping it under `skills/` keeps related files together.
- Each skill folder contains only its `SKILL.md` — no scripts. Skill content is markdown-only; Claude executes via Bash and MCP tools.
- Symlinks chain through `~/.agents/skills/` to match the established convention (`parsew-skills`, `todoist-cli`). Editing in the canonical bundle propagates everywhere.
- Journal is **outside** the bundle so the bundle can be installed read-only / shared across machines without polluting it with per-machine state.
- `TESTING.md` (not `SPEC.md`) holds smoke-test results. The spec is design-frozen; testing log is operational.

---

## Task Ordering Rationale

1. **Skeleton** before content (a place for files to live)
2. **Detection snippet** before the rituals that reference it
3. **`td-gtd`** before the rituals that Read it
4. **Rituals** in any order (independent of each other)
5. **Symlinks + journal dir + ID preflight** before testing
6. **Recipe-level fixture tests** (executor-runnable) before end-to-end
7. **End-to-end smoke** in Claude Code AND Claude Desktop (user-run only — agent cannot invoke its own slash commands)
8. **MCP capability inventory** (deferred until a real authed MCP session exists)
9. **Release tag** after user sign-off

---

### Task 1: Bundle skeleton (.gitignore, README, dir structure)

**Files:**
- Create: `~/dev/skills/todoist-skills/.gitignore`
- Create: `~/dev/skills/todoist-skills/README.md`
- Create: `~/dev/skills/todoist-skills/skills/_shared/.gitkeep`
- Create: `~/dev/skills/todoist-skills/skills/td-gtd/.gitkeep`
- Create: `~/dev/skills/todoist-skills/skills/td-morning/.gitkeep`
- Create: `~/dev/skills/todoist-skills/skills/td-shutdown/.gitkeep`

- [ ] **Step 1: Create the directory tree with `.gitkeep` so empty dirs are tracked**

```bash
mkdir -p ~/dev/skills/todoist-skills/skills/{_shared,td-gtd,td-morning,td-shutdown}
touch ~/dev/skills/todoist-skills/skills/{_shared,td-gtd,td-morning,td-shutdown}/.gitkeep
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

- [ ] **Step 3: Write `README.md` — install snippet MUST match Task 6 commands verbatim**

```markdown
# todoist-skills

A bundle of Claude skills wrapping a GTD + Kanban Todoist setup. Works in both Claude Code (`td` CLI) and Claude Desktop (Todoist MCP connector).

## Skills

- **`td-gtd`** — rules + system map + filter/label semantics. Read by the rituals; not a slash command.
- **`/td-morning`** — start-of-day ritual (~2-3 min). Locks in the deep_work block, surfaces client commitments.
- **`/td-shutdown`** — end-of-day ritual (~2-3 min). Cleans the board, logs own/client time split to the journal.

## Install

```bash
# 1. Clone or symlink the bundle to ~/dev/skills/todoist-skills/

# 2. Ensure the destination dirs exist
mkdir -p ~/.agents/skills ~/.claude/skills

# 3. Symlink each skill from the bundle into ~/.agents/skills/
for s in td-gtd td-morning td-shutdown; do
  ln -s ~/dev/skills/todoist-skills/skills/$s ~/.agents/skills/$s
done

# 4. Symlink each from ~/.agents/skills/ into ~/.claude/skills/ (relative paths)
cd ~/.claude/skills
for s in td-gtd td-morning td-shutdown; do
  ln -s ../../.agents/skills/$s $s
done
cd -

# 5. Create user-state journal directory
mkdir -p ~/.local/share/todoist-skills/journal
```

## Status

Phase 1 (MVP): instrumentation only. See `SPEC.md` and `PLAN.md`.
Phase 2 (next): `/td-weekly-review` reads the journal, closes the growth loop.
```

Save to `~/dev/skills/todoist-skills/README.md`.

- [ ] **Step 4: Verify the structure**

Use `find` with explicit grouping (BSD `find` on macOS requires `\( ... \)` to apply `-type` correctly across alternation):

```bash
find ~/dev/skills/todoist-skills \( -type f -o -type d \) -not -path '*/.git/*' -not -path '*/.git' | sort
```

Expected (note absolute paths — `~` expands to `/Users/chris`):
```
/Users/chris/dev/skills/todoist-skills
/Users/chris/dev/skills/todoist-skills/.gitignore
/Users/chris/dev/skills/todoist-skills/PLAN.md
/Users/chris/dev/skills/todoist-skills/README.md
/Users/chris/dev/skills/todoist-skills/SPEC.md
/Users/chris/dev/skills/todoist-skills/skills
/Users/chris/dev/skills/todoist-skills/skills/_shared
/Users/chris/dev/skills/todoist-skills/skills/_shared/.gitkeep
/Users/chris/dev/skills/todoist-skills/skills/td-gtd
/Users/chris/dev/skills/todoist-skills/skills/td-gtd/.gitkeep
/Users/chris/dev/skills/todoist-skills/skills/td-morning
/Users/chris/dev/skills/todoist-skills/skills/td-morning/.gitkeep
/Users/chris/dev/skills/todoist-skills/skills/td-shutdown
/Users/chris/dev/skills/todoist-skills/skills/td-shutdown/.gitkeep
```

- [ ] **Step 5: Commit**

```bash
cd ~/dev/skills/todoist-skills
git add .gitignore README.md skills/
git commit -m "Scaffold bundle directory structure and README"
```

---

### Task 2: `_shared/environment.md` — detection snippet

**Files:**
- Create: `~/dev/skills/todoist-skills/skills/_shared/environment.md`

- [ ] **Step 1: Write the file with the full detection logic (including auth-decline escape)**

````markdown
# Environment detection

The skill calling this file MUST run these steps before any Todoist operation.

## Runtime detection

Determine the runtime by trying each in order:

1. **CLI mode** — Run `command -v td` via the Bash tool. If it returns a path, the `td` CLI is available. Use `td` for everything. Reference the `todoist-cli` skill (already installed at `~/.claude/skills/todoist-cli/SKILL.md`) for syntax.

2. **MCP mode (tools already visible)** — If no `td` CLI, scan currently-visible tool names. If any name matches `^Todoist:` OR contains `Todoist` after an `mcp__` prefix (e.g., `mcp__claude_ai_Todoist__find-tasks-by-date`), the MCP connector is loaded. Proceed to "Capability discovery."

3. **MCP mode (tools deferred)** — Many runtimes hide MCP tool schemas until `tool_search` is called. Run `tool_search` with query `todoist` and `max_results: 20`. If results return, fetch their schemas, then re-check tool visibility. If now visible → proceed to "Capability discovery."

4. **No runtime** — If none of the above succeed, tell the user verbatim:

   > I need either the `td` CLI (Claude Code) or the Todoist MCP connector (Claude Desktop → Connectors menu → add Todoist). Stopping here. Once you've installed one, re-invoke me.

   Stop the skill. Do not proceed.

## Capability discovery (MCP mode only)

Once MCP tools are visible:

1. **Auth probe.** Find a tool whose name matches `*user-info*` or `*user*` or `*me*`. Call it with no arguments. If it succeeds, you are authenticated — proceed. If it fails with an auth error or no such tool exists, fall through to "Auth bootstrap."

2. **Auth bootstrap.** Only the `authenticate` and `complete_authentication` tools are visible in the unauthed state. Tell the user:

   > Todoist MCP is installed but not authenticated. I can trigger the OAuth flow now — say "yes" to proceed, or "skip" to halt and authenticate later via the Connectors menu yourself.

   - If the user says "skip" / "no" / "later" / anything declining → halt with: "OK, halting. Authenticate via Connectors when ready and re-invoke me." Do not retry, do not loop.
   - If the user says "yes" / "go" / "proceed" → call the `authenticate` tool, wait for the user to confirm browser completion, call `complete_authentication`, then re-run `tool_search` for `todoist` and re-run the auth probe. If the second probe still fails → halt with a message telling the user the auth flow didn't complete and to retry manually.

3. **Operation → tool mapping.** Skills describe operations functionally ("list tasks for today"). Map each operation to whatever tool name best matches:

   | Operation | Match pattern |
   |---|---|
   | List tasks by filter | `*find*task*` or `*list*task*` (with a filter/query argument) |
   | List tasks by project | `*find*task*` or `*list*task*` (with a project argument) |
   | Add task | `*add*task*` or `*create*task*` |
   | Complete task | `*complete*task*` or `*close*task*` |
   | Update task | `*update*task*` |
   | Move task between projects/sections | `*update*task*` with `project_id` / `section_id` |
   | List projects | `*list*project*` or `*find*project*` |
   | Get project | `*get*project*` |
   | List completed tasks | `*completed*` or `*get*completed*` |

   Cache the mapping in working memory for this turn so each operation doesn't re-search.

## Capability table

The `td-gtd` SKILL.md contains the populated capability table (filled in by Task 10 after a real MCP inventory). When acting, prefer that table; this file only describes the discovery mechanism.

## Performance note

In MCP mode, bulk operations (e.g., walking 🎯 Today + 5 filters) cost 5-10 tool calls. Skills should:
- Batch where possible.
- Warn explicitly: "Running this in Desktop takes ~20s; for daily speed, prefer Claude Code."
- Accept natural-language requests for a faster flow ("quick version", "skip the triage") and trim steps accordingly.
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
git commit -m "Add _shared/environment.md detection snippet with auth-decline escape"
```

---

### Task 3: `td-gtd/SKILL.md` — rules doc

**Files:**
- Create: `~/dev/skills/todoist-skills/skills/td-gtd/SKILL.md`

System map data comes verbatim from `/Users/chris/Downloads/todoist-migration/04-execution-log.md`. Cross-reference while writing.

- [ ] **Step 1: Write the frontmatter + opening**

The `description` field follows the "Use when…" trigger pattern (matches existing skills like `todoist-cli`):

```markdown
---
name: td-gtd
description: Use when the user is working in Todoist, asks about the GTD system, requests a daily/weekly review, or invokes /td-morning or /td-shutdown. This file is the source of truth for project IDs, filter queries, label semantics, and growth-contract targets — Read by the ritual skills.
---

# td-gtd — rules reference

This file is **Read by the ritual skills** (`/td-morning`, `/td-shutdown`), not invoked as a slash command. It is the single source of truth for the user's Todoist structure, filters, labels, growth contract, and CLI-vs-MCP capability differences.

When a ritual skill starts, it reads this file once and applies the rules below for the rest of the turn.
```

- [ ] **Step 2: Add the system map section (with project IDs AND section IDs)**

Section IDs are needed because section names like "Doing" exist in every kanban board, so name-based moves are ambiguous. Use ID refs for moves.

````markdown
## 1. System map

**Validation**: if any ID below returns 404, halt and tell the user "Project/section ID for `<name>` no longer resolves — please re-run discovery."

### Projects

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
| └─ Job Boards | `6gg3hrH2JQQx74QV` | list | Sub-parent |
|     ├─ JobBoardStarter | `6gg3XV8P3CM7RWQH` | board | |
|     └─ Fashion Workplace | `6gg3XVxhfgWC559g` | board | |
| 🔧 Admin | `6gg3XW67r44qWM45` | board | Business admin |
| 🏠 Personal | `6gg3XWCqFVr6rgXP` | board | Non-work tasks |
| ⏳ Waiting For | `6gg3XWGPrg27g2QP` | list | Anything blocked on someone else |
| 💭 Someday / Maybe | `6gg3XWMfpCwFjr7X` | list | Parked ideas |

### Sections

For LangRelay (area-based): `🔥 In Progress`, `Visual Editor`, `WP Plugin`, `Switcher & Translations`, `Other`.

For Mediastap: `WP Content Agent`, `Wisa Export Plugin`.

For every other board project: standard kanban — `Backlog`, `This Week`, `Doing`, `Review`, `Done`.

**Section ID lookup**: section IDs are not enumerated in this file because Task 7.5 of the implementation plan generates a cached map (`section-ids.json`) the first time it's needed. When a ritual needs to move a task to a section, it: (1) determines the task's project ID, (2) looks up the section ID for the named section within that project from the cached map (regenerating the cache if stale), (3) calls `td task move "id:<task>" --section "id:<section_id>"`. **Never move by section name alone** — names are not unique across projects.
````

- [ ] **Step 3: Add the filter map section**

````markdown
## 2. Filter map

| Filter | Query | Intent |
|---|---|---|
| 🎯 Today | `(today \| overdue) & !@waiting` | What I'm doing today |
| 🧠 Deep work | `@deep_work & (today \| overdue \| no date) & (p1 \| p2)` | Pick when you have a 90-min block |
| ⚡ Quick wins | `@quick & !@waiting` | Between-meetings windows |
| 🤖 Agent queue | `@agent & !@waiting` | Tasks delegatable to AI agents |
| ⏳ Waiting check | `@waiting` | Anything blocked on someone else |
| ❓ Unclarified | `no date & !#"💭 Someday / Maybe" & !@waiting` | Backlog hygiene (over-inclusive; see SPEC follow-up) |
| 📅 This week | `(7 days \| overdue) & !@waiting` | What's queued for the coming week |
````

- [ ] **Step 4: Add labels & semantics (always with `@` prefix to match query syntax)**

````markdown
## 3. Labels & semantics

Labels are context (HOW the work happens), not category (WHAT it's about). Category is the project.

In Todoist queries the labels are referenced with the `@` prefix (`@deep_work`); in the label-name field itself, the `@` is implicit. Both forms appear in the system — use `@name` everywhere consistently in skills and docs.

| Label | When to apply |
|---|---|
| `@deep_work` | Needs a 90+ min uninterrupted block. Honest application — not "important." |
| `@quick` | Fits in 5-15 min. Doesn't need deep focus. |
| `@errand` | Requires leaving the desk |
| `@calls` | Phone call required |
| `@email` | Outbound email or response |
| `@waiting` | Blocked on someone else (also lands on ⏳ Waiting For project) |
| `@two_minutes` | GTD 2-min rule — do immediately if encountered |
| `@agent` | Delegatable to an AI agent without supervision |
| `@review` | You're a reviewer/approver, not the doer |
````

- [ ] **Step 5: Add the growth contract**

````markdown
## 4. Growth contract

Source: `SPEC.md` "Growth contract."

- **Baseline (2026-05-17)**: ~€6500/mo client revenue, 0% own-product revenue, ~85% client time.
- **6-month target (2026-11-17)**: 70% client / 30% own work (time + revenue).
- **12-month target (2027-05-17)**: 50% client / 50% own work.
- **Primary growth bet**: LangRelay. When `/td-morning` proposes a deep_work block, default to LangRelay unless it has no actionable task.
- **Containment style**: soft. When a future `/td-capture` lands client work that pushes the user over a weekly cap, the skill notes it — never refuses.
- **Ratchet**: instrumented in Phase 1 (`/td-shutdown` writes the journal). Activated in Phase 2 when `/td-weekly-review` reads the journal, computes the rolling own/client ratio against the target, and names the corrective action.

**Phase 1 enforcement** (limited by design):
- `/td-morning` defaults the deep_work proposal to LangRelay.
- `/td-morning` flags client-heavy days (by task count, not duration estimate — see ritual SKILL for threshold).
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
- **Use local time** for "today" / "tomorrow" date strings: `date +%Y-%m-%d` (NOT `date -u`). The user is in CET/CEST.

## 7. When to ask vs. when to act

- **Clear routing** → act and confirm in the same step: "Moved Vendomat to Leat / This Week, due tomorrow. Undo?"
- **Ambiguous routing** → one question only.
- **Bulk** → batch into one question: "These 5 look like agent tasks — confirm all, or pick the exceptions?"

Never ask three questions in a row. If three would be needed, propose a default for all and let the user override.
````

- [ ] **Step 7: Add the capability table (placeholder + the verified `td` patterns we know work)**

Note about `td` syntax: list/filter operations belong on `td task list` (which supports `--project`, `--label`, `--priority`, and stored-filter queries) and `td filter view "<name>"` (for a saved filter by name). `td filter view` does **not** accept additional filter flags — chain at the `task list` level instead.

````markdown
## 8. Capability table (CLI vs MCP)

### Verified `td` CLI patterns

| Operation | Command |
|---|---|
| List tasks in a saved filter (by name) | `td filter view "<filter name>" --json` |
| List tasks in a project | `td task list --project "id:<project_id>" --json` |
| List tasks in a project's section | `td task list --project "id:<project_id>" --section "id:<section_id>" --json` |
| Add a task (rich flags) | `td task add "<title>" --project "id:<id>" --section "id:<id>" --labels "deep_work" --due "tomorrow" --priority p2` |
| Add a task (natural-language parsing) | `td task quickadd "<title> tomorrow p2 #Project @label"` |
| Complete a task | `td task complete "id:<task_id>"` |
| Update a task | `td task update "id:<task_id>" --due "tomorrow"` |
| Reschedule (positional date) | `td task reschedule "id:<task_id>" "tomorrow"` |
| Move a task | `td task move "id:<task_id>" --project "id:<project_id>" --section "id:<section_id>"` |
| List completed today | `td completed list --since "$(date +%Y-%m-%d)T00:00:00" --json` |
| List projects | `td project list --json` |
| Get a project | `td project view "id:<project_id>" --json` |

### MCP mode — TBD until Task 10 inventory

| Operation | MCP tool name | Degradation |
|---|---|---|
| List tasks by filter | _TBD by Task 10_ | _TBD_ |
| List tasks by project/section | _TBD_ | _TBD_ |
| Add task | _TBD_ | Likely no `quickadd` natural-language parse — specify project/labels explicitly |
| Complete task | _TBD_ | _TBD_ |
| Update task | _TBD_ | _TBD_ |
| Move task (project/section) | _TBD_ | _TBD_ |
| Bulk operations (e.g., reschedule 5 tasks) | Loop of single-task calls | N tool calls vs 1 — slow in Desktop |

**Task 10 of `PLAN.md`** populates the MCP column. Until then, in MCP mode the rituals fall back to runtime `tool_search` discovery on every operation.
````

- [ ] **Step 8: Add the journal schema section**

````markdown
## 9. Journal schema

Written by `/td-shutdown` to `~/.local/share/todoist-skills/journal/YYYY-MM-DD.md` (where `YYYY-MM-DD` is **local-time** date, not UTC).

The first line is canonical (machine-parseable). Anything below is free-text.

```
2026-05-17 | type:workday | own:2h | client:5h | note: shipped CJ-40
```

Fields:

- `type:` — `workday | partial | off`. Mandatory. `off` days are excluded from ratio math.
- `own:` — rough hours on own-work. Optional on `partial`, omitted on `off`.
- `client:` — rough hours on client work. Optional on `partial`, omitted on `off`.
- `note:` — optional free-text.

The canonical line must match this regex (anchored at start of file):

```
^[0-9]{4}-[0-9]{2}-[0-9]{2} \| type:(workday|partial|off)( \| (own|client):[^|]+)*( \| note: .*)?$
```

**Concurrency rule** (`/td-shutdown`): see Task 5 Step 4 of `PLAN.md` for the data-loss-safe read-modify-write recipe.
````

- [ ] **Step 9: Verify total length and commit**

```bash
wc -l ~/dev/skills/todoist-skills/skills/td-gtd/SKILL.md
cd ~/dev/skills/todoist-skills
git add skills/td-gtd/SKILL.md
git commit -m "Add td-gtd SKILL.md (rules, system map, growth contract, capability table)"
```

Expected: 150-260 lines.

---

### Task 4: `td-morning/SKILL.md` — start-of-day ritual

**Files:**
- Create: `~/dev/skills/todoist-skills/skills/td-morning/SKILL.md`

- [ ] **Step 1: Write the frontmatter + intro (using "Use when…" pattern)**

```markdown
---
name: td-morning
description: Use when the user invokes /td-morning or asks for a morning planning ritual. Pulls 🎯 Today + 📅 This week, triages stale carry-overs, locks in a LangRelay deep_work block, surfaces client commitments, suggests quick wins.
---

# /td-morning

A 2-3 minute ritual to lock in the day. Biases toward LangRelay for the deep_work slot.

**Default invocation:** `/td-morning`.

**Quick variant:** when the user invokes `/td-morning` and then says "quick", "fast", "skip the triage", or similar — OR includes any of those words in the same message — run the **Fast path** (see end of this file). Do NOT rely on slash-command arguments (`/td-morning fast`) — Claude Code parses that as a lookup for a skill literally named `td-morning fast` and won't match. The trigger is natural-language in the user's message, not a flag.
```

- [ ] **Step 2: Write the prelude**

````markdown
## Prelude (runs every time)

1. Resolve this skill's directory. The bundle root is two levels up from this file: `<this-file>/../../`. Resolve to `~/dev/skills/todoist-skills/`.

2. Use the Read tool to load `<bundle>/skills/td-gtd/SKILL.md`. Internalize the system map, filter map, label semantics, growth contract, and capability table.

3. Use the Read tool to load `<bundle>/skills/_shared/environment.md`. Run the runtime detection. If detection halts, stop here.

4. If MCP mode and unauthed → run the auth bootstrap from `_shared/environment.md`. If the user declines auth → halt.

5. Once detection completes, you have a runtime and an operation→tool mapping. Proceed.
````

- [ ] **Step 3: Write the default flow**

````markdown
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
````

- [ ] **Step 4: Write the fast-path variant**

````markdown
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
````

- [ ] **Step 5: Write the failure modes section**

````markdown
## Failure modes (handled in-skill)

- **Empty 🎯 Today, non-empty This Week** — offer multi-select from This Week (Step 3 handles this).
- **First-run: everything empty** — Step 2 catches this and offers the on-ramp.
- **Skipped yesterday, many overdue items** — batch into Step 3 as a single "These 8 are overdue: bulk-push to This Week, or pick which to keep on Today?"
- **MCP >10s** — show "running in MCP mode, hold on" before the user thinks it's hung.
- **Filter doesn't exist** (e.g., user renamed it) — halt with: "Filter `<name>` not found. Was it renamed? Check the system map in `td-gtd` section 2."
- **Project ID returns 404** — halt with: "Project `<name>` (id:<id>) doesn't resolve. Was it deleted? Update `td-gtd` section 1."
````

- [ ] **Step 6: Verify and commit**

```bash
wc -l ~/dev/skills/todoist-skills/skills/td-morning/SKILL.md
cd ~/dev/skills/todoist-skills
git add skills/td-morning/SKILL.md
git commit -m "Add /td-morning ritual SKILL.md"
```

Expected: ~120-170 lines.

---

### Task 5: `td-shutdown/SKILL.md` — end-of-day ritual

**Files:**
- Create: `~/dev/skills/todoist-skills/skills/td-shutdown/SKILL.md`

- [ ] **Step 1: Write the frontmatter + intro**

```markdown
---
name: td-shutdown
description: Use when the user invokes /td-shutdown or asks for an end-of-day ritual. Cleans 🎯 Today, captures own/client time split + day-type into the journal, surfaces tomorrow's anchor and waiting-for nudges.
---

# /td-shutdown

A 2-3 minute close. Two jobs: leave Todoist clean for tomorrow, and capture today's signal for Phase 2's weekly review.

**Invocation:** `/td-shutdown`.
```

- [ ] **Step 2: Write the prelude**

````markdown
## Prelude

1. Resolve this skill's directory. Bundle root is two levels up.
2. Read `<bundle>/skills/td-gtd/SKILL.md`. Internalize rules, growth contract, journal schema (section 9), and capability table (section 8).
3. Read `<bundle>/skills/_shared/environment.md`. Run detection. Halt if no runtime. If MCP-unauthed and user declines auth → halt.
4. Proceed.
````

- [ ] **Step 3: Write the flow**

````markdown
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
````

- [ ] **Step 4: Write failure modes**

````markdown
## Failure modes

- **Skipped 3 days** — first prompt: "Last entry was N days ago. Log the last N days now, or skip ahead?" If "log": loop day-type + split prompts for each missing day, using the appropriate past date in `$DATE`. If "skip": just do today.
- **Ambiguous split** ("a few hours own, mostly client") — accept as fuzzy: `own:fuzzy client:fuzzy | note: <quote>`. Phase 2 ignores fuzzy days for ratio math.
- **Journal write fails** (permission, disk full) — tell the user the path that failed; ask if they want the line printed for manual paste. Don't halt the rest.
- **Filesystem write unavailable (MCP)** — fallback in Step 4.
- **`td completed list` returns nothing today** — fine, no completed-task review needed.
- **Section ID lookup miss** (section name not found in cache) — regenerate the cache for that project: `td section list --project "id:<project_id>" --json`; retry the move.
````

- [ ] **Step 5: Verify and commit**

```bash
wc -l ~/dev/skills/todoist-skills/skills/td-shutdown/SKILL.md
cd ~/dev/skills/todoist-skills
git add skills/td-shutdown/SKILL.md
git commit -m "Add /td-shutdown ritual SKILL.md with safe journal recipe"
```

Expected: ~180-230 lines.

---

### Task 6: Install symlinks

**Files:**
- Create: `~/.agents/skills/td-gtd`, `~/.agents/skills/td-morning`, `~/.agents/skills/td-shutdown` (symlinks)
- Create: `~/.claude/skills/td-gtd`, `~/.claude/skills/td-morning`, `~/.claude/skills/td-shutdown` (symlinks)

- [ ] **Step 1: Ensure destination directories exist; check for conflicts**

```bash
mkdir -p ~/.agents/skills ~/.claude/skills
for s in td-gtd td-morning td-shutdown; do
  if [ -e "$HOME/.agents/skills/$s" ] || [ -L "$HOME/.agents/skills/$s" ]; then
    echo "CONFLICT in ~/.agents/skills/: $s already exists"
  fi
  if [ -e "$HOME/.claude/skills/$s" ] || [ -L "$HOME/.claude/skills/$s" ]; then
    echo "CONFLICT in ~/.claude/skills/: $s already exists"
  fi
done
```

Expected: no "CONFLICT" lines. If any → stop and resolve (rename or remove the existing entry).

- [ ] **Step 2: Create the three `~/.agents/skills/` symlinks**

```bash
ln -s ~/dev/skills/todoist-skills/skills/td-gtd ~/.agents/skills/td-gtd
ln -s ~/dev/skills/todoist-skills/skills/td-morning ~/.agents/skills/td-morning
ln -s ~/dev/skills/todoist-skills/skills/td-shutdown ~/.agents/skills/td-shutdown
ls -la ~/.agents/skills/td-gtd ~/.agents/skills/td-morning ~/.agents/skills/td-shutdown
```

Expected: each line shows `→ /Users/chris/dev/skills/todoist-skills/skills/<name>`.

- [ ] **Step 3: Create the three `~/.claude/skills/` symlinks with relative paths**

```bash
cd ~/.claude/skills
ln -s ../../.agents/skills/td-gtd td-gtd
ln -s ../../.agents/skills/td-morning td-morning
ln -s ../../.agents/skills/td-shutdown td-shutdown
ls -la td-gtd td-morning td-shutdown
cd -
```

Expected: each line shows `→ ../../.agents/skills/<name>`.

- [ ] **Step 4: Verify the full chain resolves on the filesystem**

```bash
readlink -f ~/.claude/skills/td-gtd
readlink -f ~/.claude/skills/td-morning
readlink -f ~/.claude/skills/td-shutdown
```

Expected: each resolves to `/Users/chris/dev/skills/todoist-skills/skills/<name>`.

- [ ] **Step 5: Verify SKILL.md is readable through the chain (static content check)**

```bash
head -5 ~/.claude/skills/td-gtd/SKILL.md
head -5 ~/.claude/skills/td-morning/SKILL.md
head -5 ~/.claude/skills/td-shutdown/SKILL.md
```

Expected: each shows frontmatter (`---`, `name:`, `description:`, `---`).

No commit — symlinks live outside the bundle.

---

### Task 7: Create the user-state journal directory

**Files:**
- Create: `~/.local/share/todoist-skills/journal/` (directory only)

- [ ] **Step 1: Create the directory and verify writability**

```bash
mkdir -p ~/.local/share/todoist-skills/journal
touch ~/.local/share/todoist-skills/journal/.test && rm ~/.local/share/todoist-skills/journal/.test && echo "writable"
```

Expected: `writable`.

No commit — user state lives outside the bundle.

---

### Task 7.5: Project ID preflight + section-ID cache generation

**Files:**
- Create: `~/dev/skills/todoist-skills/section-ids.json` (committed cache of project → section name → section ID)

This task runs once before any smoke test. It validates that all project IDs in `td-gtd` still resolve, then builds the section-ID cache that `/td-shutdown`'s move operations need.

- [ ] **Step 1: Validate every project ID resolves**

```bash
PROJECT_IDS="6gg3XRqxXPWPMwH7 6gg3XV44fmp4qp6j 6gg3gv5f4fj5M46g 6gg3XVMc99VMGxMV 6gg3XVq73jRXwxHR 6gg3XVQrrCgRV9Qv 6gg3hr9xM2F69qq7 6gg3XV25WQFjCWwQ 6gg3XV4C9xjPpjM2 6gg3XW32cC6MRwgG 6gg3XVGJjR3w6hJq 6gg3XV95224CRHJF 6gg3hrH2JQQx74QV 6gg3XV8P3CM7RWQH 6gg3XVxhfgWC559g 6gg3XW67r44qWM45 6gg3XWCqFVr6rgXP 6gg3XWGPrg27g2QP 6gg3XWMfpCwFjr7X"
for id in $PROJECT_IDS; do
  if ! td project view "id:$id" --json > /dev/null 2>&1; then
    echo "FAIL: $id does not resolve"
  fi
done
echo "preflight complete"
```

Expected: only `preflight complete`, no `FAIL:` lines. If any ID fails, halt and update `td-gtd` section 1 with the new ID before continuing.

- [ ] **Step 2: Build the section-ID cache**

```bash
cd ~/dev/skills/todoist-skills
python3 <<'PY'
import json, subprocess

PROJECTS = {
    "LangRelay": "6gg3XRqxXPWPMwH7",
    "Selektable": "6gg3XV44fmp4qp6j",
    "Leat": "6gg3XVMc99VMGxMV",
    "Hairgivers": "6gg3XVq73jRXwxHR",
    "Mediastap": "6gg3XVQrrCgRV9Qv",
    "Parsew": "6gg3XV25WQFjCWwQ",
    "Kopplio": "6gg3XV4C9xjPpjM2",
    "Landing Gallery": "6gg3XW32cC6MRwgG",
    "AutomagicWP": "6gg3XVGJjR3w6hJq",
    "EmitKit": "6gg3XV95224CRHJF",
    "JobBoardStarter": "6gg3XV8P3CM7RWQH",
    "Fashion Workplace": "6gg3XVxhfgWC559g",
    "🔧 Admin": "6gg3XW67r44qWM45",
    "🏠 Personal": "6gg3XWCqFVr6rgXP",
}
out = {}
for name, pid in PROJECTS.items():
    res = subprocess.run(
        ["td", "section", "list", "--project", f"id:{pid}", "--json"],
        capture_output=True, text=True, check=True
    )
    sections = json.loads(res.stdout)
    out[name] = {"project_id": pid, "sections": {s["name"]: s["id"] for s in sections}}

with open("section-ids.json", "w") as f:
    json.dump(out, f, indent=2, ensure_ascii=False)
print(f"wrote {len(out)} projects' sections to section-ids.json")
PY
```

Expected: a line like `wrote 14 projects' sections to section-ids.json`.

- [ ] **Step 3: Verify the cache and commit**

```bash
python3 -c "import json; d = json.load(open('section-ids.json')); print('\n'.join(f'{p}: {list(v[\"sections\"].keys())}' for p, v in d.items()))"
```

Expected: each project lists its kanban sections (Backlog, This Week, Doing, Review, Done) or for LangRelay/Mediastap the area-based / custom sections.

```bash
cd ~/dev/skills/todoist-skills
git add section-ids.json
git commit -m "Add section-ids.json cache (project → section name → ID)"
```

---

### Task 8: Static / fixture-based verification (executor-runnable)

This task replaces the prior "smoke test" approach. The executor (the agent running this plan) cannot invoke slash commands on itself — slash commands are user-initiated. So Task 8 covers everything the executor CAN verify without user interaction:

- [ ] **Step 1: Frontmatter syntax check**

For each of `td-gtd`, `td-morning`, `td-shutdown`, verify frontmatter is well-formed YAML:

```bash
for s in td-gtd td-morning td-shutdown; do
  python3 -c "
import yaml, re
with open('$HOME/.claude/skills/$s/SKILL.md') as f:
    content = f.read()
m = re.match(r'^---\n(.*?)\n---', content, re.DOTALL)
if not m:
    print('$s: NO FRONTMATTER')
else:
    fm = yaml.safe_load(m.group(1))
    print(f'$s: name={fm.get(\"name\")} desc-length={len(fm.get(\"description\",\"\"))}')
"
done
```

Expected: each line shows `<name>: name=<name> desc-length=<N>` with N ≥ 50 (description is descriptive enough to trigger skill matching).

- [ ] **Step 2: Symlink chain resolution check**

Already covered in Task 6 Step 4. Re-verify:

```bash
for s in td-gtd td-morning td-shutdown; do
  resolved=$(readlink -f ~/.claude/skills/$s/SKILL.md)
  expected="/Users/chris/dev/skills/todoist-skills/skills/$s/SKILL.md"
  if [ "$resolved" = "$expected" ]; then
    echo "$s: OK"
  else
    echo "$s: MISMATCH (resolved=$resolved expected=$expected)"
  fi
done
```

Expected: three `OK` lines.

- [ ] **Step 3: Journal recipe against fixtures**

Copy the recipe from `td-shutdown/SKILL.md` Step 4 into a standalone script `/tmp/test-journal.sh` and run it against five fixture inputs. The recipe should produce the expected output in each case.

```bash
mkdir -p /tmp/journal-test
cd /tmp/journal-test

# Inline the recipe as a function for testing
write_journal() {
  local FILE="$1"
  local NEW_LINE="$2"
  local TMP="$FILE.tmp.$$"
  if [ -f "$FILE" ]; then
    local FIRST_LINE
    FIRST_LINE="$(head -n 1 "$FILE")"
    if printf '%s' "$FIRST_LINE" | grep -qE '^[0-9]{4}-[0-9]{2}-[0-9]{2} \|'; then
      { printf '%s\n' "$NEW_LINE"; tail -n +2 "$FILE"; } > "$TMP"
    else
      { printf '%s\n' "$NEW_LINE"; cat "$FILE"; } > "$TMP"
    fi
    mv "$TMP" "$FILE"
  else
    printf '%s\n' "$NEW_LINE" > "$FILE"
  fi
}

NEW="2026-05-17 | type:workday | own:2h | client:5h"

# Fixture (a): file doesn't exist
rm -f a.md
write_journal "a.md" "$NEW"
echo "(a) new file → expect 1 line: $(wc -l < a.md)"

# Fixture (b): canonical-only file (single line)
printf '2026-05-17 | type:partial | own:1h\n' > b.md
write_journal "b.md" "$NEW"
echo "(b) canonical-only → expect 1 line with new content: $(head -1 b.md)"

# Fixture (c): canonical + free-text body
printf '2026-05-17 | type:partial | own:1h\nfree text below\nmore notes\n' > c.md
write_journal "c.md" "$NEW"
echo "(c) canonical + body → expect new canonical + 'free text below' + 'more notes':"
cat c.md

# Fixture (d): free-text-only (no canonical first line)
printf 'free text only\nsecond line\n' > d.md
write_journal "d.md" "$NEW"
echo "(d) free-text-only → expect new canonical PREPENDED, no data loss:"
cat d.md

# Fixture (e): file with no trailing newline
printf '2026-05-17 | type:partial | own:1h' > e.md   # no \n
write_journal "e.md" "$NEW"
echo "(e) no trailing newline → expect new canonical line, no smushing:"
cat e.md
```

Expected output:
- (a): 1 line.
- (b): the new `$NEW` line.
- (c): `$NEW` then `free text below` then `more notes`.
- (d): `$NEW` then `free text only` then `second line` — original content preserved.
- (e): `$NEW` (one line, properly terminated).

If any fixture fails, fix the recipe in `td-shutdown/SKILL.md` Step 4 before continuing.

- [ ] **Step 4: Verify `td` syntax patterns in `td-gtd` capability table**

For each verified-CLI command in `td-gtd` section 8, run a dry-run or `--help` to confirm the flags exist:

```bash
td task list --help | head -30
td task move --help | head -30
td filter view --help | head -20
td completed list --help | head -20
td task reschedule --help | head -20
```

Expected: each `--help` shows the flags used (`--project`, `--section`, `--filter`, `--since`, `--json`). If any flag listed in `td-gtd` section 8 doesn't appear → fix `td-gtd` before continuing.

- [ ] **Step 5: Commit any fixes from Steps 1-4 (if needed)**

If fixture (Step 3) or syntax (Step 4) caught a bug:

```bash
cd ~/dev/skills/todoist-skills
git add skills/
git commit -m "Fix issues caught by Task 8 fixture/syntax verification"
```

If no bugs, skip the commit.

---

### Task 9: End-to-end smoke test — USER-RUN

This task is for the user, not the executor. The plan cannot invoke its own slash commands. The user runs `/td-morning` and `/td-shutdown` in both runtimes and reports results.

- [ ] **Step 1: User runs `/td-morning` in Claude Code**

Expected:
1. Claude Code recognizes the skill.
2. Reads `td-gtd/SKILL.md` and `_shared/environment.md`.
3. Detects CLI mode.
4. Runs Step 1's 5 `td` calls.
5. Walks triage → deep_work → client commitments → quick wins → 3-line plan.

User reports: pass / fail (with any error message).

- [ ] **Step 2: User runs `/td-shutdown` in Claude Code**

Expected:
1. Reads shared files.
2. Walks 🎯 Today + completed-today.
3. Asks loose-ends, day-type, split.
4. Writes journal to `~/.local/share/todoist-skills/journal/YYYY-MM-DD.md` (local date).
5. 2-line close.

After: user verifies the journal file from a terminal:

```bash
ls ~/.local/share/todoist-skills/journal/
cat ~/.local/share/todoist-skills/journal/$(date +%Y-%m-%d).md
```

Expected: one file, first line matches the canonical regex.

- [ ] **Step 3: User re-runs `/td-shutdown` (same day) with different values**

Expected: journal file STILL has only one canonical first line (the new one). If two appear → the recipe isn't being followed; fix `td-shutdown` Step 4.

- [ ] **Step 4: User invokes `/td-morning` and says "quick version"**

Expected: fast-path triggers — runs in <5s in CLI mode, shows just 🎯 Today + one LangRelay proposal, exits after one confirm/reject.

- [ ] **Step 5: User opens Claude Desktop, ensures Todoist connector is installed and authed**

If not authed yet: Settings → Connectors → add Todoist → complete OAuth.

- [ ] **Step 6: User runs `/td-morning` in Claude Desktop**

Expected:
1. No `td` binary → falls through to MCP scan.
2. May call `tool_search` for `todoist` if deferred.
3. Auth probe succeeds (already authed in Step 5).
4. Same flow as CLI mode, just slower (~10-30s).

If auth probe fails → user follows the auth bootstrap flow.

- [ ] **Step 7: User runs `/td-shutdown` in Claude Desktop**

Expected: identical flow to CLI mode. Verify journal:

```bash
# From a terminal:
cat ~/.local/share/todoist-skills/journal/$(date +%Y-%m-%d).md
```

If Claude Desktop lacks filesystem write through MCP: the fallback in `td-shutdown` Step 4 triggers ("Here's the line, paste it manually"). User pastes the line into the file via terminal/editor. This is an acceptable Phase 1 limitation.

- [ ] **Step 8: User writes a TESTING.md log of results**

Create `~/dev/skills/todoist-skills/TESTING.md`:

```markdown
# todoist-skills MVP smoke test results

_Tested: <date> by Chris_

## Claude Code (CLI mode)
- /td-morning default: <pass/fail + notes>
- /td-morning "quick version": <pass/fail + notes>
- /td-shutdown: <pass/fail + notes>
- /td-shutdown re-run same day: <pass/fail — journal still 1 canonical line?>

## Claude Desktop (MCP mode)
- /td-morning: <pass/fail + observed latency>
- /td-shutdown: <pass/fail + did MCP filesystem write work or fallback triggered?>

## Known issues
<anything that didn't work, deferred to Phase 2 or to a follow-up>
```

Commit:

```bash
cd ~/dev/skills/todoist-skills
git add TESTING.md
git commit -m "Add smoke test results for MVP"
```

---

### Task 10: MCP capability inventory — populate `td-gtd` section 8

Requires an authed MCP session (Claude Desktop preferred, or Claude Code with the MCP authed).

- [ ] **Step 1: Ensure authed MCP session**

Verify by calling `tool_search` for `todoist` and confirming non-auth tools (e.g., `find-tasks-by-filter`, `add-task`) are visible.

- [ ] **Step 2: Inventory tools**

Run `tool_search` with query `todoist` and `max_results: 30`. Capture the full result. For each tool: exact name, one-line summary, required parameters.

- [ ] **Step 3: Map operations to tool names**

For each row in the `td-gtd` capability table (section 8 "MCP mode" column), identify the matching tool. Note any degradation.

Example expected output:

| Operation | MCP tool name | Degradation |
|---|---|---|
| List tasks by filter | `mcp__claude_ai_Todoist__find-tasks-by-filter` (or actual name) | None |
| Add task | `mcp__claude_ai_Todoist__add-task` | No natural-language parse — explicit project/labels required |
| Move task | `mcp__claude_ai_Todoist__update-task` (set `section_id`) | None |
| Bulk reschedule | N×update-task | N calls vs 1 Bash loop — slow |

- [ ] **Step 4: Update `td-gtd/SKILL.md` section 8**

Replace the placeholder MCP column. Commit:

```bash
cd ~/dev/skills/todoist-skills
git add skills/td-gtd/SKILL.md
git commit -m "Populate td-gtd capability table from real MCP inventory"
```

---

### Task 11: Release tag — after user sign-off

- [ ] **Step 1: Confirm user has signed off on Task 9 (TESTING.md exists and shows passing results)**

```bash
cat ~/dev/skills/todoist-skills/TESTING.md
```

If TESTING.md doesn't exist or shows failures → do not tag. Return to fix the failing tasks.

- [ ] **Step 2: Tag the release**

```bash
cd ~/dev/skills/todoist-skills
git tag -a v0.1.0-mvp -m "MVP: td-gtd + /td-morning + /td-shutdown shipped"
git tag
git log --oneline | head -10
```

---

## Self-review

(Run after the rewrite. Findings inline.)

**Spec coverage:**
- Bundle structure (SPEC) → Task 1.
- `td-gtd` content (SPEC) → Task 3.
- `/td-morning` content (SPEC) → Task 4.
- `/td-shutdown` content (SPEC) → Task 5.
- Environment detection (SPEC) → Task 2.
- Read-mechanism for rules (SPEC) → Tasks 4 & 5 prelude.
- Journal in XDG path (SPEC) → Tasks 5 & 7.
- Day-type field (SPEC) → Task 5 Step 3.
- Read-modify-write concurrency (SPEC) → Task 5 Step 4 + Task 8 Step 3 fixture test.
- Auth bootstrap with decline-escape (review finding #14) → Task 2 `environment.md`.
- Symlink install with mkdir prereq (review finding #2) → Task 6 Step 1.
- README/Task 6 install parity (review finding #3) → both now use the same `cd ~/.claude/skills` loop.
- Project ID validation (review finding #16) → Task 7.5 preflight.
- Section-ID cache (review finding #9) → Task 7.5 Step 2 + Task 5 Step 3 lookup.
- MCP capability table fill-in (SPEC) → Task 10.
- Phase 1 vs Phase 2 split (SPEC) → captured in `td-gtd` section 4 + README.
- TESTING.md split from SPEC.md (review finding #20) → Task 9 Step 8.
- Release tag after user sign-off (review finding #17) → Task 11.

**Placeholder scan:**
- "Filled in by Task 10" inside `td-gtd` section 8 — deliberate, has scaffold content, replacement is a defined task.
- No `TBD`, `add appropriate`, `similar to Task N`, or `implement later` in any code step.

**Bash recipe correctness** (review findings #1, #11, #12):
- Journal recipe (Task 5 Step 4) uses `printf '%s\n'`, regex check on first line, `mv` atomic rename, and `$$` tmp file. Verified against fixtures (a)-(e) in Task 8 Step 3.
- `find` command (Task 1 Step 4) uses `\( -type f -o -type d \)` for BSD-find compatibility.
- All date-string commands use `date +%Y-%m-%d` (local), not `date -u`.

**`td` syntax correctness** (review findings #8, #9):
- `td filter view` no longer chains additional filter flags (no `--filter` on top of `view`).
- Section moves use ID refs from the cached `section-ids.json`.
- Task 8 Step 4 validates each `td` command's flags against `--help` before smoke testing.

**Recursive-invocation problem fixed** (review findings #5, #6, #7):
- Task 8 is now executor-runnable static + fixture verification.
- Task 9 is explicitly user-run end-to-end.
- Slash-command-arg assumption (review finding #4) replaced with natural-language trigger inside the skill.

**Type / identifier consistency:**
- Project IDs match `04-execution-log.md` throughout.
- Filter names (with emojis) match `td-gtd` section 2 throughout.
- Journal field names (`type:`, `own:`, `client:`, `note:`) consistent across `td-gtd` section 9, `td-shutdown` Step 3-4, and Task 8 fixture tests.
- Labels written with `@` prefix consistently.
- "Read-modify-write" terminology consistent (SPEC, `td-gtd`, `td-shutdown`).

Plan is internally consistent. Ready for execution.
