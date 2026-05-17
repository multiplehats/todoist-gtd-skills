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
