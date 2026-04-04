# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Claude Code plugin that teaches AI agents how to use `diskcopilot-cli` for macOS disk scanning, analysis, and cleanup. There is no compiled code -- the entire plugin is markdown instruction files and JSON metadata.

## Repository Structure

- `.claude-plugin/plugin.json` -- plugin manifest (name, version, metadata)
- `skills/diskcopilot/SKILL.md` -- the core skill: complete instructions, SQLite schema, SQL examples, CLI reference. This is the "brain" of the plugin and what agents actually read.
- `commands/*.md` -- three slash commands (`scan`, `cleanup`, `report`) that orchestrate the skill for specific workflows
- `evals/evals.json` -- evaluation prompts to test skill effectiveness (3 scenarios)
- `disk-analysis-workspace/` -- gitignored eval artifacts (before/after comparisons)

## Architecture

The plugin follows an instruction-driven pattern:

1. **SKILL.md** contains all knowledge: CLI installation, SQLite schema docs, example SQL queries, predefined commands, safety rules, and example flows
2. **Commands** are thin orchestrators that reference the skill and add step-by-step workflows
3. The plugin auto-installs `diskcopilot-cli` binary to `~/.local/bin` on first use (no sudo)

Commands delegate to the skill via "Use the diskcopilot skill for instructions" -- they don't duplicate the skill content.

## Running Evaluations

Evals test whether the skill improves agent responses to disk space questions:

```bash
# Evals are defined in evals/evals.json
# Artifacts go to disk-analysis-workspace/ (gitignored)
```

There is no build step, linter, or test runner -- this is a documentation-only project.

## Key Conventions

- **Sizes**: always bytes in SQL/JSON, displayed with decimal units (1 GB = 1,000,000,000 bytes, matching macOS/Finder)
- **Deletion**: always `--trash` (recoverable), never `--permanent` unless user explicitly requests it
- **Scanning**: a home directory scan covers all subdirectory queries -- no need to re-scan child paths
- **Two modes**: "quick query" (SQL in conversation) vs "interactive cleanup" (write insights file, launch web UI with `diskcopilot-cli serve`)

## Extending the Plugin

- **New command**: add a `.md` file to `commands/` with YAML frontmatter (`description` field) and step-by-step instructions that reference the skill
- **New capability**: update `skills/diskcopilot/SKILL.md` with SQL examples, CLI flags, or safety guidelines
- **New eval**: add an entry to `evals/evals.json` with `id`, `prompt`, and `expected_output`
