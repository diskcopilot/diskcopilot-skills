---
name: diskcopilot
description: |
  How to use diskcopilot-cli to scan, analyze, and clean up disk space on macOS. Use this skill whenever the user mentions disk space, storage usage, large files, duplicate files, cleanup, what's taking up space, freeing up storage, old files, dev artifacts (node_modules, target, .build), or wants to understand their disk usage patterns. Also use when the user asks you to help organize, audit, or reduce the size of a directory. Even if the user doesn't mention "diskcopilot" by name — if they're talking about disk space or storage, this is the skill to use.
---

# Disk Analysis with diskcopilot-cli

`diskcopilot-cli` is a fast macOS disk scanner that caches filesystem metadata in SQLite for instant queries. It scans a home directory (~1.4M files) in ~12 seconds using macOS `getattrlistbulk(2)`.

The tool only reads filesystem metadata (names, sizes, timestamps) — it never reads file contents (except for duplicate detection, which computes local-only blake3 hashes). It has no network access.

## First: ensure diskcopilot-cli is installed

Before doing anything else, check if the CLI is available:

```bash
which diskcopilot-cli
```

If not found, install it automatically:

```bash
curl -fsSL https://github.com/bluedusk/diskcopilot-cli/releases/latest/download/diskcopilot-cli-$(uname -m)-apple-darwin.tar.gz | tar xz && mv diskcopilot-cli /usr/local/bin/
```

This downloads a single pre-built binary (~5 MB). No Rust or build tools needed. If `/usr/local/bin` requires sudo, use `sudo mv` instead.

## Quick start

1. Check if scanned: `diskcopilot-cli query info <path>`
2. Scan if needed: `diskcopilot-cli scan <path> --full`
3. Query with SQL: `diskcopilot-cli query sql "<SELECT ...>" <path>`

## Two response modes

**Quick query** — for specific questions ("how big are my node_modules?", "find large mp4 files"). Run a SQL query, present results in conversation. Fast.

**Interactive cleanup** — for browsing and deleting ("help me clean up", "free up 10GB"). Analyze with SQL, write insights to a file, launch the web UI:
```bash
diskcopilot-cli serve <path> --insights-file /tmp/insights.txt
```
User selects and trashes items in the browser. No AI round-trips for the interactive part.

## Scanning

```bash
diskcopilot-cli scan <path>           # files >= 1MB (fast, covers cleanup use cases)
diskcopilot-cli scan <path> --full    # all files (slower, needed for counts/small files)
diskcopilot-cli scan / --force --full # full disk (needs Full Disk Access in System Settings)
```

A home directory scan covers all subdirectory queries — no need to scan each folder separately. Check `query info` first to avoid unnecessary re-scans.

## Querying with SQL

The primary query tool for the agent. Write any read-only SQL against the cache:

```bash
diskcopilot-cli query sql "<SQL>" <path>
```

Output is always JSON. Only SELECT, WITH, and PRAGMA are allowed.

### Schema

```sql
dirs (
  id              INTEGER PRIMARY KEY,
  parent_id       INTEGER,     -- parent directory (NULL for root)
  name            TEXT,        -- directory name (root stores absolute path)
  file_count      INTEGER,     -- direct files in this dir
  total_file_count INTEGER,    -- recursive file count
  total_logical_size INTEGER,  -- recursive logical bytes
  total_disk_size  INTEGER,    -- recursive on-disk bytes
  created_at      INTEGER,     -- unix timestamp
  modified_at     INTEGER      -- unix timestamp
)

files (
  id              INTEGER PRIMARY KEY,
  dir_id          INTEGER,     -- FK to dirs.id
  name            TEXT,
  logical_size    INTEGER,
  disk_size       INTEGER,     -- on-disk bytes (what matters for cleanup)
  created_at      INTEGER,     -- unix timestamp
  modified_at     INTEGER,     -- unix timestamp
  extension       TEXT,        -- lowercase, e.g. 'mp4', 'pdf'
  inode           INTEGER,
  content_hash    TEXT          -- blake3 hash (only set after duplicate scan)
)

scan_meta (
  id, root_path, scanned_at, total_files, total_dirs, total_size, scan_duration_ms
)
```

The `dirs` table is a tree via `parent_id`. Each dir's `total_disk_size` is the sum of all files underneath it recursively. Sizes are in bytes (decimal: 1 GB = 1,000,000,000 bytes, matching macOS/Finder).

### Example queries

```bash
# How big are all node_modules?
diskcopilot-cli query sql "SELECT d.total_disk_size, p.name as project
  FROM dirs d JOIN dirs p ON d.parent_id = p.id
  WHERE d.name = 'node_modules'
  ORDER BY d.total_disk_size DESC" ~

# Top 10 largest files
diskcopilot-cli query sql "SELECT name, disk_size, extension
  FROM files ORDER BY disk_size DESC LIMIT 10" ~

# Total size by file extension
diskcopilot-cli query sql "SELECT extension, COUNT(*) as count, SUM(disk_size) as total
  FROM files WHERE extension IS NOT NULL
  GROUP BY extension ORDER BY total DESC LIMIT 20" ~

# Files modified this week
diskcopilot-cli query sql "SELECT name, disk_size
  FROM files WHERE modified_at > strftime('%s','now','-7 days')
  ORDER BY disk_size DESC LIMIT 20" ~

# Files older than 1 year
diskcopilot-cli query sql "SELECT name, disk_size, modified_at
  FROM files WHERE modified_at < strftime('%s','now','-365 days')
  ORDER BY disk_size DESC LIMIT 20" ~

# Find all .dmg installers
diskcopilot-cli query sql "SELECT name, disk_size FROM files
  WHERE extension = 'dmg' ORDER BY disk_size DESC" ~

# Directories named 'target' (Rust build caches)
diskcopilot-cli query sql "SELECT d.total_disk_size, p.name as project
  FROM dirs d JOIN dirs p ON d.parent_id = p.id
  WHERE d.name = 'target'
  ORDER BY d.total_disk_size DESC" ~

# Reconstruct full path for a directory (walk parent chain)
diskcopilot-cli query sql "WITH RECURSIVE path(id, name, parent_id, full) AS (
    SELECT id, name, parent_id, name FROM dirs WHERE id = <DIR_ID>
    UNION ALL
    SELECT p.id, p.name, p.parent_id, p.name || '/' || path.full
    FROM dirs p JOIN path ON p.id = path.parent_id
  ) SELECT full FROM path WHERE parent_id IS NULL" ~
```

You can write SQL for any question. If you need to reconstruct full paths for files, join `files.dir_id` to `dirs.id` and walk the `parent_id` chain with a recursive CTE, or use the predefined commands which do this automatically.

## Predefined query commands

Shortcuts for common queries. These handle path reconstruction and human-readable output. Use `--json` for structured output.

```bash
diskcopilot-cli query tree <path> --depth 2 --json       # directory size tree
diskcopilot-cli query large-files <path> --min-size 100M  # biggest files
diskcopilot-cli query dev-artifacts <path> --json         # node_modules, target, etc.
diskcopilot-cli query old <path> --days 365 --json        # old files
diskcopilot-cli query recent <path> --days 7 --json       # recent files
diskcopilot-cli query ext <path> --ext mp4 --json         # files by extension
diskcopilot-cli query search <path> --name docker --json  # search by filename
diskcopilot-cli query duplicates <path> --json            # duplicates (slow, reads content)
diskcopilot-cli query summary <path> --json               # all-in-one cleanup report
diskcopilot-cli query info <path> --json                  # scan metadata
```

Use predefined commands when they fit the question exactly. Use SQL when you need custom filtering, grouping, or joins.

## Deleting

```bash
diskcopilot-cli delete <path> --trash       # moves to macOS Trash (recoverable)
diskcopilot-cli delete <path> --permanent   # irreversible
```

Always prefer `--trash`. Always confirm with the user before deleting. System paths are blocked by a safety check.

## Interactive web UI

For cleanup sessions where the user wants to browse and select items to delete:

```bash
# Write your AI analysis to a file first
cat > /tmp/diskcopilot-insights.txt << 'EOF'
Your home directory uses 90 GB. Biggest cleanup opportunities:

**Dev build caches (15.7 GB)** — Rust target/ dirs. Safe to delete, rebuilt by cargo build.
**npm cache (7 GB)** — Safe to clear with npm cache clean --force.
**Old files (4.7 GB)** — Files in ~/Documents untouched for over a year.
EOF

# Launch the web UI
diskcopilot-cli serve <path> --insights-file /tmp/diskcopilot-insights.txt
```

The browser opens with:
- AI insights panel at the top (your analysis — this is the most valuable part)
- Directory overview with size bars
- Categorized items with checkboxes
- "Move to Trash" button with confirmation dialog

Security: per-session auth token, CORS locked to localhost, system paths blocked.

Tell the user: "I've opened a cleanup dashboard in your browser. You can browse the findings, select items, and trash them directly. Everything goes to macOS Trash so nothing is permanent."

## Example flows

**"How big are my node_modules?"**
1. `query info ~` — check cache
2. `query sql "SELECT d.total_disk_size, p.name as project FROM dirs d JOIN dirs p ON d.parent_id = p.id WHERE d.name = 'node_modules' ORDER BY d.total_disk_size DESC" ~`
3. Present as a table. Done.

**"What's taking up my disk space?"**
1. `query info ~` → scan if needed
2. `query sql "SELECT name, total_disk_size FROM dirs WHERE parent_id = (SELECT id FROM dirs WHERE parent_id IS NULL) ORDER BY total_disk_size DESC LIMIT 15" ~`
3. Present top-level breakdown. Follow up with specific areas if asked.

**"Help me free up 10GB"**
1. Scan if needed
2. Run multiple SQL queries (large files, dev artifacts, old files)
3. Write categorized analysis to insights file
4. `serve ~ --insights-file /tmp/diskcopilot-insights.txt`
5. User browses and trashes in browser

## Important notes

- Sizes in JSON/SQL are bytes. Convert with decimal units: 1 GB = 1,000,000,000 bytes.
- Timestamps are Unix seconds since epoch.
- A home scan covers all subdirectory queries.
- Cache is at `~/.diskcopilot/cache/` — one SQLite DB per scan root.
- Dev artifacts (node_modules, target) are safe to delete but need `npm install` / `cargo build` to regenerate. Mention this.
- `query duplicates` reads file content for hashing — warn the user it's slower.
