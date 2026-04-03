# diskcopilot-skills

Claude Code skills for [diskcopilot-cli](https://github.com/bluedusk/diskcopilot-cli) — disk scanning, analysis, and cleanup on macOS.

## What's this?

This is a [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code) that teaches Claude how to use `diskcopilot-cli` to analyze your disk space, find large files, detect duplicates, locate dev build caches, and help you clean up.

Once installed, just ask Claude things like:
- "What's taking up space on my Mac?"
- "Find large files in my Downloads"
- "How much space are my node_modules using?"
- "Help me free up 10GB of disk space"

Claude will scan your filesystem, query the cache, and present actionable cleanup recommendations.

## Prerequisites

Install [diskcopilot-cli](https://github.com/bluedusk/diskcopilot-cli):

```bash
git clone https://github.com/bluedusk/diskcopilot-cli
cd diskcopilot-cli
make install
```

## Install

### Claude Code CLI

```bash
claude plugin add bluedusk/diskcopilot-skills
```

### Manual

Add to your Claude Code settings (`~/.claude/settings.json`):

```json
{
  "extraKnownMarketplaces": {
    "diskcopilot-skills": {
      "source": {
        "source": "github",
        "repo": "bluedusk/diskcopilot-skills"
      },
      "autoUpdate": true
    }
  }
}
```

Then enable the plugin:

```json
{
  "enabledPlugins": {
    "diskcopilot-skills@diskcopilot-skills": true
  }
}
```

## Skills

### disk-analysis

Teaches Claude how to use `diskcopilot-cli` for:
- Scanning directories and building a metadata cache
- Querying for large files, old files, duplicates, dev artifacts
- Presenting directory size trees and cleanup summaries
- Safely deleting files (Trash or permanent)

## Privacy

`diskcopilot-cli` only reads filesystem metadata (names, sizes, timestamps). It has no network dependencies and never sends data anywhere. See the [diskcopilot-cli README](https://github.com/bluedusk/diskcopilot-cli#privacy--security) for details.

## License

MIT
