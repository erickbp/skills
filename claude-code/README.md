# Claude Code Skills

Skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

## Available Skills

| Skill | Description |
|-------|-------------|
| [task-splitter](task-splitter/) | Decompose large PRDs, RFCs, epics, and implementation plans into atomic task cards for coding agents |

## Installation

Copy a skill folder into your project's `.claude/skills/` directory or your user-level `~/.claude/skills/` directory:

```bash
# Project-level (recommended for team-shared skills)
cp -r task-splitter /path/to/your/project/.claude/skills/

# User-level (available across all projects)
cp -r task-splitter ~/.claude/skills/
```
