# Skills

Reusable skills for AI coding agents. This repository publishes the same core ideas in agent-specific packaging where needed.

## Repository Layout

- **[claude-code/](claude-code/)**: Skills packaged for [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- **[codex/](codex/)**: Skills packaged for [Codex](https://developers.openai.com/codex/skills)

Each skill lives in its own folder. The root README covers repo-wide structure; each platform folder has its own README with platform-specific install details.

## Available Skills

| Platform | Skill | Description |
| --- | --- | --- |
| Claude Code | [iterative-builder](claude-code/iterative-builder/) | Plan one task at a time against the actual codebase, build/review/merge it, then plan the next |

## Installation

### Claude Code

Copy a skill folder into your project's `.claude/skills/` directory or your user-level `~/.claude/skills/` directory.

```bash
cp -r claude-code/iterative-builder /path/to/project/.claude/skills/
```
