# CatCode Claude Plugins

Claude Code plugins for [CatCode](https://github.com/CatCodeGames) Unity libraries. Each library has its own skill with the API and usage rules. Claude loads a skill only when a task touches that library.

## Installation

```bash
claude plugin marketplace add CatCodeGames/claude-plugins
claude plugin install unity-libs@catcode
```

Update after changes:

```bash
claude plugin marketplace update catcode
```

## Skills

| Skill | Library |
|---|---|
| `catcode-audiosystem` | [AudioSystem](https://github.com/CatCodeGames/AudioSystem) |

## Adding a skill

1. Add `plugins/unity-libs/skills/<skill-name>/SKILL.md`.
2. Add a row to the table above.
3. Bump `version` in `plugins/unity-libs/.claude-plugin/plugin.json`.
