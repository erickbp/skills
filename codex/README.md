# Codex Skills

Skills in this folder are packaged for [Codex](https://developers.openai.com/codex/skills).

## Available Skills

| Skill | Description |
| --- | --- |
| [task-splitter](task-splitter/) | Break large PRDs, RFCs, epics, roadmap items, migrations, and implementation plans into atomic task cards |

## Install

From the repository root, copy the skill folder into `~/.codex/skills/`:

```bash
cp -r codex/task-splitter ~/.codex/skills/
```

Restart Codex after copying a new skill.

## Use

Invoke the skill explicitly with `$task-splitter`, or rely on normal trigger phrases such as:

- "Break this PRD into task cards"
- "Turn this spec into executable tasks"
- "What are the implementation steps for this plan?"
