# todoist-skills

A bundle of Claude skills wrapping a GTD + Kanban Todoist setup. Works in both Claude Code (`td` CLI) and Claude Desktop (Todoist MCP connector).

## Skills

- **`td-gtd`** — rules + system map + filter/label semantics. Read by the rituals; not a slash command.
- **`/td-morning`** — start-of-day ritual (~2-3 min). Locks in the deep_work block, surfaces client commitments.
- **`/td-shutdown`** — end-of-day ritual (~2-3 min). Cleans the board, logs own/client time split to the journal.

## Install

````bash
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
````

## Status

Phase 1 (MVP): instrumentation only. See `SPEC.md` and `PLAN.md`.
Phase 2 (next): `/td-weekly-review` reads the journal, closes the growth loop.
