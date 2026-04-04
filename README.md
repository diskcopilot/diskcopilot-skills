# diskcopilot

Claude Code plugin for disk scanning, analysis, and cleanup on macOS.

Ask Claude things like:
- "What's taking up space on my Mac?"
- "How big are my node_modules?"
- "Help me free up 10GB"

Or use slash commands:
- `/diskcopilot:scan` — scan a directory
- `/diskcopilot:cleanup` — open interactive cleanup dashboard in browser
- `/diskcopilot:report` — generate a disk usage report

## Install

```bash
# 1. Install diskcopilot-cli
git clone https://github.com/bluedusk/diskcopilot-cli && cd diskcopilot-cli && make install && cd ..

# 2. Install the Claude Code plugin
claude plugin add bluedusk/diskcopilot-skills
```

That's it. Start a Claude Code session and ask about your disk space.

## What it does

The plugin teaches Claude how to use [diskcopilot-cli](https://github.com/bluedusk/diskcopilot-cli) — a fast macOS disk scanner (~12s for a home directory). It scans filesystem metadata into a local SQLite cache, then Claude queries it with SQL to answer any disk space question.

**Skill** (auto-triggers when you talk about disk space):
- Scans directories and caches metadata
- Writes SQL queries against the cache to answer questions
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
