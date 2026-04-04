---
name: diskcopilot
description: |
  How to use diskcopilot-cli to scan, analyze, and clean up disk space on macOS. Use this skill whenever the user mentions disk space, storage usage, large files, duplicate files, cleanup, what's taking up space, freeing up storage, old files, dev artifacts (node_modules, target, .build), or wants to understand their disk usage patterns. Also use when the user asks you to help organize, audit, or reduce the size of a directory. Even if the user doesn't mention "diskcopilot" by name — if they're talking about disk space or storage, this is the skill to use.
---

# Disk Analysis with diskcopilot-cli

`diskcopilot-cli` is a fast macOS disk scanner that caches filesystem metadata in SQLite for instant queries. It scans a home directory (~1.4M files) in ~12 seconds using macOS `getattrlistbulk(2)`.

The tool only reads filesystem metadata (names, sizes, timestamps) — it never reads file contents (except for duplicate detection, which computes local-only blake3 hashes). It has no network access.

## Presentation rules

- **Never show raw SQL, JSON, or command output to the user.** Run queries behind the scenes, then present clean formatted results (tables, lists, summaries).
- **Number every row** in file/directory tables so the user can say "delete 1, 3, 5" instead of typing paths. Keep an internal mapping of index → full path so you can act on their selection immediately.
- Convert bytes to human-readable sizes (GB, MB). Convert Unix timestamps to dates.
- When the user says "my files", they mean personal files — Documents, Downloads, Desktop, Pictures, Movies, Music. Exclude system files, build artifacts, caches, logs, and hidden directories.
- Filter out noise: package-lock.json, .DS_Store, build outputs, etc. Only show files a human would recognize as theirs.
- When showing file paths, format them as clickable links using `file://` URLs: `[filename](file:///full/path/to/file)`. Most terminals render these as clickable links that open the file or folder directly.
- If the user asks to open a file or folder, use `open <path>` (opens in default app) or `open -R <path>` (reveals in Finder).
- **For "find my files" queries** (recently created, modified, specific names): use `find` on the live filesystem, not the SQLite cache. The default scan only stores files >= 1MB, so small personal files (photos, documents, configs) won't be in the database. Use diskcopilot for size analysis; use `find` for file discovery.

## First: ensure diskcopilot-cli is installed

Before doing anything else, check if the CLI is available:

```bash
which diskcopilot-cli
```

If not found, install it to `~/.local/bin` (no sudo needed):

```bash
mkdir -p ~/.local/bin && curl -fsSL https://github.com/bluedusk/diskcopilot-cli/releases/latest/download/diskcopilot-cli-$(uname -m)-apple-darwin.tar.gz | tar xz -C ~/.local/bin/
```

Then verify it's on PATH. If `~/.local/bin` is not on PATH, run it with the full path (`~/.local/bin/diskcopilot-cli`) or add it:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

This downloads a single pre-built binary (~5 MB). No Rust or build tools needed.

## Quick start

1. **Always check the cache first**: `diskcopilot-cli query info ~` — never skip this step. Look at the `scanned_at` timestamp.
2. **If scan exists and is < 1 hour old**: use it silently.
3. **If scan exists and is 1-24 hours old**: use it, mention when it was scanned ("using scan from 3 hours ago").
4. **If scan exists and is > 24 hours old**: rescan automatically — don't ask, just do it. 12 seconds is faster than a round-trip question.
5. **If no scan exists**: scan: `diskcopilot-cli scan ~`
6. Query with SQL: `diskcopilot-cli query sql "<SELECT ...>" ~`

**Always scan the home directory (`~`), not a subdirectory.** A single scan of `~` covers all subdirectory queries. Don't scan `~/playground` when the user asks about node_modules — scan `~` once and query the subtree you need. Scanning narrow paths wastes time because the user will inevitably ask about another folder next.

## Generating an insight report

When the user asks for a scan, report, or disk analysis, produce a full **Disk Health Report**. This is where the agent adds real value — not just showing raw data, but interpreting it and giving actionable advice.

### Step 1: Gather data

Run these queries (in parallel if possible):

```bash
diskcopilot-cli scan ~   # only if cache is missing or stale
diskcopilot-cli query sql "SELECT total_disk_size FROM dirs WHERE parent_id IS NULL" ~
diskcopilot-cli query sql "SELECT name, total_disk_size FROM dirs WHERE parent_id = (SELECT id FROM dirs WHERE parent_id IS NULL) ORDER BY total_disk_size DESC LIMIT 15" ~
diskcopilot-cli query summary ~ --json
diskcopilot-cli query dev-artifacts ~ --json
diskcopilot-cli query sql "SELECT name, disk_size, extension, modified_at FROM files ORDER BY disk_size DESC LIMIT 15" ~
diskcopilot-cli query sql "SELECT extension, COUNT(*) as count, SUM(disk_size) as total FROM files WHERE extension IS NOT NULL GROUP BY extension ORDER BY total DESC LIMIT 10" ~
```

Also query macOS system junk — see `references/macos-junk.md` for the full list of known junk categories, SQL queries, and cleanup commands:
```bash
diskcopilot-cli query sql "SELECT name, total_disk_size FROM dirs WHERE name IN ('Caches', 'Google', 'Firefox', 'Mozilla', 'Safari', 'Homebrew', 'pip', 'Yarn', 'CocoaPods', 'Logs', '.Trash', 'DerivedData', 'Archives', 'Backup', 'MobileSync', 'ShipIt', 'ms-playwright', 'puppeteer', 'prisma') AND total_disk_size > 10000000 ORDER BY total_disk_size DESC" ~
```

Also get disk free space:
```bash
df -h ~
```

### Step 2: Present the Disk Health Report

Use this format — markdown tables for data, plain text for context. This is an example of what the output should look like:

```markdown
# Disk Health Report

## Health Score: F (Critical)

**11 GB free out of 228 GB (95% full)** — your disk is critically low on space.

## Storage Overview

| | |
|---|---|
| **Total disk** | 228 GB |
| **Used** | 217 GB |
| **Free** | 11 GB (5%) |
| **Scanned** | 115.5 GB across 1.4M files |

### Top directories

| Size | % | Directory |
|------|---|-----------|
| 43.6 GB | 37.8% | Library (app data, caches, support files) |
| 29.5 GB | 25.5% | playground (dev projects) |
| 9.1 GB | 7.8% | .npm (package cache) |
| ... | | |

## Cleanup Opportunities

### 1. Safe to delete (45.9 GB) — dev artifacts, regenerated by build commands

| Size | Type | Instances | Regenerate with |
|------|------|-----------|-----------------|
| 21.9 GB | target/ (Rust) | 11 | `cargo build` |
| 14.7 GB | node_modules/ | 5,983 | `npm install` |
| ... | | | |

### 2. macOS system junk (3.7 GB)

| Size | Item | Action |
|------|------|--------|
| 1.9 GB | Chrome cache | Clear in Chrome settings |
| 812 MB | VSCode update cache | Safe to delete |
| ... | | |

### 3. Review before deleting (4.8 GB) — old or forgotten files

| Size | File | Age | Notes |
|------|------|-----|-------|
| 1.4 GB | VID_20230818_122637.mp4 | Aug 2023 | Video untouched for 2+ years |
| ... | | | |

### 4. App-managed data (15.5 GB) — don't delete manually

| Size | Item | Notes |
|------|------|-------|
| 8.3 GB | claudevm.bundle | Needed for Claude Code |
| ... | | |

## Recommended Actions

| # | Action | Saves |
|---|--------|-------|
| 1 | Delete Rust target/ dirs | 21.9 GB |
| 2 | `npm cache clean --force` | 9.1 GB |
| 3 | Delete inactive node_modules | 14.7 GB |
| ... | | |

**Potential total savings: 50+ GB**
```

Health score grades:
- A (>40% free): Healthy, plenty of room
- B (20-40% free): Good shape, minor cleanup possible
- C (10-20% free): Getting tight, cleanup recommended
- D (5-10% free): Low space, cleanup needed soon
- F (<5% free): Critical, immediate action needed

### Step 3: Offer to act

After presenting the report, ask what the user wants to clean up. Delete items directly via `diskcopilot-cli delete <path> --trash` when they confirm. No need to leave the conversation.

## Response modes

**Report** — for "scan my disk", "give me a report", "what's using my space". Run the full insight report flow above.

**Quick query** — for specific questions ("how big are my node_modules?", "find large mp4 files"). Check the cache, run a SQL query, present results. No full report needed.

**Cleanup** — for "help me clean up", "free up 10GB", "delete junk". Run the report, then delete items the user confirms — all within the conversation.

## Scanning

```bash
diskcopilot-cli scan ~                # default — files >= 1MB, fast, accurate dir sizes
diskcopilot-cli scan ~ --full         # all files — needed for file counts, extension queries, name search
diskcopilot-cli scan / --force --full # full disk (needs Full Disk Access in System Settings)
```

The default mode gives accurate directory sizes and captures all large files. Use `--full` only when the user's question requires individual small file records (e.g. "how many .rs files", "search for a config file", "find duplicates").

A home directory scan covers all subdirectory queries — never scan a subdirectory when `~` will do.

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

## Safelist (keep)

Users can mark files as important so they never show up in cleanup recommendations:

```bash
diskcopilot-cli keep <path>       # protect a file or folder
diskcopilot-cli unkeep <path>     # remove protection
diskcopilot-cli keep-list         # show all protected items
```

Protected files still appear in reports (the user needs to see their full disk usage), but `diskcopilot-cli delete` will refuse to delete them. When the user indicates a file is important — "keep this", "don't delete this", "that's not junk", "I need that", "skip this one" — run `diskcopilot-cli keep <path>` immediately without asking. The safelist persists at `~/.diskcopilot/safelist.txt`.

## Example flows

**"How big are my node_modules?"**
1. `query info ~` — check cache
2. If scan exists, ask user if they want to rescan. If no scan, scan now.
3. `query sql "SELECT d.total_disk_size, p.name as project FROM dirs d JOIN dirs p ON d.parent_id = p.id WHERE d.name = 'node_modules' ORDER BY d.total_disk_size DESC" ~`
4. Present as a table. Done.

**"What's taking up my disk space?"**
1. `query info ~` — check cache
2. If scan exists, ask user if they want to rescan. If no scan, scan now.
3. `query sql "SELECT name, total_disk_size FROM dirs WHERE parent_id = (SELECT id FROM dirs WHERE parent_id IS NULL) ORDER BY total_disk_size DESC LIMIT 15" ~`
4. Present top-level breakdown. Follow up with specific areas if asked.

**"Help me free up 10GB"**
1. `query info ~` — check cache, scan if needed
2. Run the full Disk Health Report flow
3. Present categorized cleanup opportunities with sizes
4. Ask what to delete, execute with `diskcopilot-cli delete <path> --trash`

## File insights

When results include specific files or directories, don't just list names and sizes — explain what they are and whether they're safe to delete. Use the file extension, directory name, and path to infer purpose. Examples:

- `~/Library/Caches/com.spotify.client` — Spotify's local cache. Safe to delete; the app rebuilds it.
- `~/.rustup/toolchains/` — Rust compiler toolchains. Safe to remove unused ones, but the active toolchain is needed for `cargo build`.
- `~/Documents/taxes-2024.pdf` — likely personal documents. Not safe to delete without asking.
- `node_modules/` — npm dependencies. Safe to delete; `npm install` regenerates them.
- `.DS_Store` — macOS Finder metadata. Safe to delete; recreated automatically.

If you're unsure what a file is, say so rather than guessing. For ambiguous cases, recommend the user check before deleting.

## Important notes

- Sizes in JSON/SQL are bytes. Convert with decimal units: 1 GB = 1,000,000,000 bytes.
- Timestamps are Unix seconds since epoch.
- A home scan covers all subdirectory queries.
- Cache is at `~/.diskcopilot/cache/` — one SQLite DB per scan root.
- Dev artifacts (node_modules, target) are safe to delete but need `npm install` / `cargo build` to regenerate. Mention this.
- `query duplicates` reads file content for hashing — warn the user it's slower.
