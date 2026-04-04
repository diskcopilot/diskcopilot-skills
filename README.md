# diskcopilot

AI agent plugin for disk scanning, analysis, and cleanup. Requires macOS.

Teaches AI agents how to use [diskcopilot-cli](https://github.com/bluedusk/diskcopilot-cli) — a fast disk scanner (~12s for a home directory) that caches filesystem metadata in SQLite, letting the agent write SQL to answer any disk space question.

Ask your agent things like:
- "What's taking up space on my Mac?"
- "How big are my node_modules?"
- "Help me free up 10GB"

## Install

```bash
claude plugin add bluedusk/diskcopilot-skills
```

That's it. The plugin will automatically install the `diskcopilot-cli` binary on first use.

## Other agents

For Cursor, Copilot, Codex, or other AI agents, copy the skill instructions from [skills/diskcopilot/SKILL.md](skills/diskcopilot/SKILL.md) into your agent's configuration file (`.cursorrules`, `copilot-instructions.md`, `AGENTS.md`, etc.). The skill includes auto-install instructions for the CLI binary.

## Commands

| Command | What it does |
|---------|-------------|
| `/diskcopilot:scan` | Scan a directory (or cwd) and cache metadata |
| `/diskcopilot:cleanup` | Analyze + open browser dashboard to select and trash items |
| `/diskcopilot:report` | Analyze + show report in conversation |

## Privacy

`diskcopilot-cli` only reads filesystem metadata (names, sizes, timestamps). It has no network dependencies and never sends data anywhere. The interactive cleanup UI runs on localhost with a per-session auth token. See [diskcopilot-cli](https://github.com/bluedusk/diskcopilot-cli#privacy--security) for details.

## License

MIT
