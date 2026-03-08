# Skills

Reusable skills for AI coding agents. This repository publishes the same core ideas in agent-specific packaging where needed.

## Repository Layout

- **[claude-code/](claude-code/)**: Skills packaged for [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- **[codex/](codex/)**: Skills packaged for [Codex](https://developers.openai.com/codex/skills)

Each skill lives in its own folder. The root README covers repo-wide structure; each platform folder has its own README with platform-specific install details.

## Available Skills

| Platform | Skill | Description |
| --- | --- | --- |
| Claude Code | [task-splitter](claude-code/task-splitter/) | Decompose large PRDs, RFCs, epics, migrations, and implementation plans into atomic task cards |
| Codex | [task-splitter](codex/task-splitter/) | Decompose large PRDs, RFCs, epics, migrations, and implementation plans into atomic task cards |

## Installation

### Claude Code

Copy a skill folder into your project's `.claude/skills/` directory or your user-level `~/.claude/skills/` directory.

```bash
cp -r claude-code/task-splitter /path/to/project/.claude/skills/
```

### Codex

Copy a skill folder into `~/.codex/skills/`, then restart Codex so it picks up the new skill.

```bash
cp -r codex/task-splitter ~/.codex/skills/
```
