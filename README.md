# diskcopilot

AI agent plugin for disk scanning, analysis, and cleanup on macOS.

Ask your agent things like:
- "What's taking up space on my Mac?"
- "How big are my node_modules?"
- "Help me free up 10GB"

Or use slash commands:
- `/diskcopilot:scan` — scan a directory
- `/diskcopilot:cleanup` — open interactive cleanup dashboard in browser
- `/diskcopilot:report` — generate a disk usage report

## Install

```bash
claude plugin add bluedusk/diskcopilot-skills
```

That's it. The plugin will automatically install the `diskcopilot-cli` binary on first use.

### Other agents

For Cursor, Copilot, Codex, or other AI agents, copy the skill instructions from [skills/diskcopilot/SKILL.md](skills/diskcopilot/SKILL.md) into your agent's configuration file (`.cursorrules`, `copilot-instructions.md`, `AGENTS.md`, etc.). The skill includes auto-install instructions for the CLI binary.

## What it does

The plugin teaches AI agents how to use [diskcopilot-cli](https://github.com/bluedusk/diskcopilot-cli) — a fast macOS disk scanner (~12s for a home directory). It scans filesystem metadata into a local SQLite cache, then the agent queries it with SQL to answer any disk space question.

- Auto-installs the CLI binary on first use
- Writes SQL queries against the cache to answer any question
- Presents cleanup recommendations by category
- Launches an interactive web UI for browsing and trashing files

**Commands:**

| Command | What it does |
|---------|-------------|
| `/diskcopilot:scan` | Scan a directory (or cwd) |
| `/diskcopilot:cleanup` | Analyze + open browser dashboard to select and trash items |
| `/diskcopilot:report` | Analyze + show report in conversation |

## Privacy

`diskcopilot-cli` only reads filesystem metadata (names, sizes, timestamps). It has no network dependencies and never sends data anywhere. The interactive cleanup UI runs on localhost with a per-session auth token. See [diskcopilot-cli](https://github.com/bluedusk/diskcopilot-cli#privacy--security) for details.

## License

MIT
