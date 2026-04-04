---
description: "Scan a directory and cache metadata for disk analysis"
---

Scan the specified directory (or current working directory if none given) using diskcopilot-cli. Use the diskcopilot skill for instructions.

1. Ensure diskcopilot-cli is installed (check `which diskcopilot-cli`, install from GitHub releases if missing — see diskcopilot skill)
2. Check if already scanned: `diskcopilot-cli query info <path>`
2. If not scanned or stale, run: `diskcopilot-cli scan <path> --full`
3. After scan completes, show the user the scan summary (files, dirs, size, duration)
4. Suggest next steps: "You can now ask me about disk usage, large files, or run `/diskcopilot:cleanup` for interactive cleanup."

If no path argument is provided, use the current working directory. If the user provides a path, use that.
