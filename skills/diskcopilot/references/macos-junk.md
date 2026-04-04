# macOS Junk Categories

Known directories and files that accumulate on macOS and are safe or semi-safe to clean up.
Use this as a reference when building the Disk Health Report.

## Always safe to delete

| Category | Path / Query | Clean command | Notes |
|----------|-------------|---------------|-------|
| Homebrew cache | `~/Library/Caches/Homebrew` | `brew cleanup --prune=all` | Old downloaded bottles |
| pip cache | `~/Library/Caches/pip` | `pip cache purge` | Downloaded packages |
| npm cache | `~/.npm/_cacache` | `npm cache clean --force` | Package download cache |
| pnpm store | `~/Library/pnpm` | `pnpm store prune` | Unused package versions |
| Yarn cache | `~/Library/Caches/Yarn` | `yarn cache clean` | Package download cache |
| VSCode update cache | `~/Library/Caches/com.microsoft.VSCode.ShipIt` | Delete folder | Old update staging |
| Firefox update cache | `~/Library/Caches/Mozilla/updates` | Delete folder | Old update staging |
| Cursor update cache | `~/Library/Application Support/Caches/cursor-updater` | Delete folder | Old update staging |
| App update staging | Any `pending/` or `update*/` inside `Caches` | Delete folder | Old downloads that weren't applied |
| System logs (user) | `~/Library/Logs` | Delete old `.log` files | App crash/debug logs |
| Xcode DerivedData | `~/Library/Developer/Xcode/DerivedData` | Delete folder or Xcode > Settings > Locations | Build cache, rebuilt on next build |
| Xcode archives | `~/Library/Developer/Xcode/Archives` | Delete old archives | Old app builds |
| Xcode device support | `~/Library/Developer/Xcode/iOS DeviceSupport` | Delete old versions | Debug symbols for old iOS versions |
| CocoaPods cache | `~/Library/Caches/CocoaPods` | `pod cache clean --all` | Downloaded pod specs |
| Gradle cache | `~/.gradle/caches` | Delete folder | Rebuilt on next build |
| Maven cache | `~/.m2/repository` | Delete old artifacts | Re-downloaded on next build |
| Composer cache | `~/.composer/cache` | `composer clear-cache` | PHP package cache |
| Go module cache | `~/go/pkg/mod/cache` | `go clean -modcache` | Re-downloaded on next build |
| Rust registry cache | `~/.cargo/registry/cache` | `cargo cache --autoclean` (needs cargo-cache) | Old crate tarballs |
| Puppeteer browsers | `~/.cache/puppeteer` | Delete folder | Re-downloaded on next use |
| Playwright browsers | `~/Library/Caches/ms-playwright` | `npx playwright uninstall --all` | Re-downloaded on next use |
| Prisma engines | `~/.cache/prisma` | Delete folder | Re-downloaded on next use |

## Safe to delete (user review recommended)

| Category | Path / Query | Notes |
|----------|-------------|-------|
| Browser cache (Chrome) | `~/Library/Caches/Google/Chrome` | Clear from Chrome settings for selective control |
| Browser cache (Firefox) | `~/Library/Caches/Firefox` | Clear from Firefox settings |
| Browser cache (Safari) | `~/Library/Caches/com.apple.Safari` | Clear from Safari > Develop > Empty Caches |
| DMG installers in Downloads | `*.dmg` files in `~/Downloads` | Safe once the app is installed |
| Old zip/tar archives | `*.zip`, `*.tar`, `*.tar.gz` in Downloads | If already extracted |
| Trash | `~/.Trash` | Already deleted, just not emptied |
| Duplicate downloads | Files with `(1)`, `(2)` suffixes | Usually accidental re-downloads |

## Check before deleting

| Category | Path / Query | Notes |
|----------|-------------|-------|
| iOS device backups | `~/Library/Application Support/MobileSync/Backup` | Can be very large (10-50 GB per device). Only delete if backed up elsewhere |
| Docker disk image | `~/Library/Containers/com.docker.docker/Data` | Shrink with `docker system prune -a`, don't delete the raw file |
| Chrome AI model | `weights.bin` in Chrome OptGuide | 4+ GB, re-downloads if deleted. Disable in Chrome flags |
| Claude VM bundle | `claudevm.bundle` | ~8 GB, needed for Claude Code sandbox |
| Mail downloads | `~/Library/Mail` | Can grow large with attachments |
| Slack cache | `~/Library/Application Support/Slack` | Can accumulate GB of cached files |

## SQL queries for the report

```sql
-- Browser and app caches
SELECT name, total_disk_size FROM dirs
WHERE name IN ('Caches', 'Google', 'Firefox', 'Mozilla', 'Safari',
               'Homebrew', 'pip', 'Yarn', 'CocoaPods', 'Logs',
               '.Trash', 'DerivedData', 'Archives', 'Backup',
               'MobileSync', 'ShipIt', 'ms-playwright', 'puppeteer',
               'prisma')
AND total_disk_size > 10000000
ORDER BY total_disk_size DESC

-- DMG installers in Downloads
SELECT name, disk_size FROM files
WHERE extension = 'dmg'
ORDER BY disk_size DESC

-- Duplicate-looking files (name contains parenthesized number)
SELECT name, disk_size FROM files
WHERE name LIKE '% (1).%' OR name LIKE '% (2).%'
ORDER BY disk_size DESC LIMIT 20

-- Old large files (not modified in 6+ months)
SELECT name, disk_size, modified_at FROM files
WHERE modified_at < strftime('%s','now','-180 days')
ORDER BY disk_size DESC LIMIT 20
```
