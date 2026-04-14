# deploy.py

A lightweight, portable Python deployment script for pushing production-ready
files to one or more destinations, including UNC network shares.

Copy the `deploy/` folder into any project, edit `deploy.config.json`, and run.

---

## Requirements

- Python 3.10+
- Git available on PATH
- No third-party packages — stdlib only

---

## Quick Start (new project)

### Option A — Initialize from scratch

Run from your project root:

```bat
python deploy/deploy.py --init
```

This creates a `deploy/` folder with a skeleton `deploy.config.json` and a
`DEPLOY_SETUP.md` guide. Edit the config, then run a dry-run.

### Option B — Copy an existing deploy folder

1. Copy this `deploy/` folder into your project root.
2. Edit `deploy.config.json` — set `project_id`, `root`, `destinations`, and `files`.
3. Run:

```bat
deploy\deploy.bat --dry-run
```

4. When the dry-run looks right, drop the `--dry-run` flag.

---

## Usage

```bat
deploy\deploy.bat                                    # default config (auto-resolve)
deploy\deploy.bat --dry-run                          # preview only
deploy\deploy.bat --verbose                          # show every file operation
deploy\deploy.bat --config deploy\prod.config.json   # alternate config
deploy\deploy.bat --init                             # scaffold deploy folder
```

On Git Bash or Linux: `./deploy/deploy.sh` (same flags apply).

`--dry-run` can also be set in the config (`"dry_run": true`).
The CLI flag always wins if both are set.

---

## Config — deploy.config.json

### Full JSON schema

```json
{
  "project_id": "my-project-slug",
  "description": "Short description of what this project does",
  "root": "..",
  "deploy_from": "dist/MyApp",
  "destinations": [
    "\\\\10.115.200.250\\Dropbox\\Utilities\\MyProject"
  ],
  "dry_run": false,
  "files": [
    "*.exe",
    "_internal/",
    "readme.md"
  ]
}
```

### Field reference

| Field | Type | Required | Description |
|---|---|---|---|
| `project_id` | `string` | **Yes** | Stable identity slug for this project (e.g., `"olin-custom-utilities"`). Written into `deploy_log.json` so the client updater can match projects by ID even if folder names change. |
| `description` | `string` | No | Short description shown to users in the Utilities Manager dashboard. Recommend keeping under 80 characters. |
| `root` | `string` | No | Project root, relative to config file location. Defaults to the config file's parent directory. |
| `deploy_from` | `string` | No | Subfolder within `root` to use as the base for file resolution and relative paths at the destination. Leave empty `""` or omit to use `root`. Example: `"dist/Olin Custom Utilities"` means files resolve from that subfolder and land at the destination root — the `dist/Olin Custom Utilities` prefix is stripped. |
| `destinations` | `string[]` | Yes* | List of target paths. UNC paths (`\\server\share\...`) work natively on Windows. |
| `destination` | `string` | Yes* | Shorthand — single target path. Use `destinations` for multiple. |
| `dry_run` | `boolean` | No | If `true`, preview only. CLI `--dry-run` flag overrides this. Default: `false`. |
| `files` | `string[]` | Yes | Whitelist of files/patterns to deploy. Paths are relative to `deploy_from` (or `root` if `deploy_from` is not set). |

*One of `destination` or `destinations` is required. If both are present, `destinations` is used.

### File patterns

| Pattern | Behaviour |
|---|---|
| `readme.md` | Single explicit file |
| `*.vsix` | All `.vsix` files in deploy_from |
| `src/**/*.py` | All `.py` files under `src/`, recursively |
| `config/*.json` | All `.json` files directly in `config/` |
| `output/` | Entire folder, recursively (trailing slash optional) |
| `_internal/` | Entire folder, recursively |

All paths are relative to `deploy_from`.

### Example configs

**VS Code extension project** (deploy built package + docs):
```json
{
  "project_id": "deltav-language-support",
  "root": "..",
  "destinations": ["\\\\10.115.200.250\\Dropbox\\Utilities\\Advanced Utilities - Drummond\\DeltaV Language Support"],
  "files": ["*.vsix", "readme.md", "docs/SETUP_GUIDE.md"]
}
```

**Compiled tool with build output** (strip build directories):
```json
{
  "project_id": "olin-custom-utilities",
  "root": "..",
  "deploy_from": "dist/Olin Custom Utilities",
  "destinations": ["\\\\10.115.200.250\\Dropbox\\Utilities\\Advanced Utilities - Drummond\\Olin Custom Utilities"],
  "files": ["*.exe", "_internal/"]
}
```

**Multi-destination deploy** (dev + prod servers):
```json
{
  "project_id": "my-app",
  "root": "..",
  "destinations": [
    "\\\\10.115.1.50\\Dropbox\\MyApp",
    "\\\\10.115.1.55\\Dropbox\\MyApp"
  ],
  "files": ["dist/", "config/prod.json"]
}
```

---

## Deploy Log

A `deploy_log.json` is written to each destination after every run. It records:

- **`project_id`** — the stable identity key from your config
- **`description`** — project description from your config
- **`total_size_mb`** — total size of all deployed files in megabytes
- Timestamp and whether it was a dry run
- **Current HEAD commit** — short hash, long hash, tags, cleaned message
- **Up to 2 previous tagged commits** matching `v*`, `release-*`, or `deploy-*`
  patterns (Co-Authored-By trailers stripped automatically)
- **`files_copied`** — files that were new or changed
- **`files_skipped`** — files unchanged (byte-identical)
- **`files_removed`** — files that were in the previous deploy but no longer in source
- **`files_failed`** — files that failed to copy
- Rolling history of the last 20 deploy entries

### Example log snippet

```json
{
  "project_id": "deltav-language-support",
  "last_deploy": {
    "timestamp": "2026-04-09T14:32:01",
    "dry_run": false,
    "git": {
      "current_commit": {
        "short_hash": "a1b2c3d",
        "long_hash": "a1b2c3d4e5f6...",
        "tags": ["v1.2.0"],
        "message": "Add recursive folder deployment support"
      },
      "previous_tagged_commits": [
        {
          "short_hash": "9f8e7d6",
          "tags": ["v1.1.0"],
          "message": "Fix UNC path handling on Windows"
        }
      ]
    },
    "files_copied": ["deltav-expressions-0.7.3.vsix"],
    "files_skipped": ["readme.md"],
    "files_removed": ["deltav-expressions-0.7.2.vsix"],
    "files_failed": []
  }
}
```

---

## File Locking

The deploy script creates a `.update-lock` file in each destination folder
during file operations. This prevents conflicts with the client updater
(`Update-Utilities.ps1`) pulling files at the same time.

- Lock is acquired before copying, released after
- If a lock exists and is **< 5 minutes old**: waits and retries (up to 3 attempts)
- If a lock exists and is **>= 5 minutes old**: treats it as stale and removes it
- Locks are always cleaned up on exit, including crashes (`try/finally`)

---

## Output Verbosity

By default, the script shows **summary counts** per destination instead of
listing every file. This keeps output manageable for projects with hundreds
or thousands of files (e.g., compiled executables with `_internal/` folders).

Use `--verbose` to see every individual file operation:

```bat
deploy\deploy.bat --verbose
```

---

## Behaviour Notes

- **Unchanged files are skipped** — byte-level comparison via `filecmp.cmp(shallow=False)`.
  Skipped files appear in the deploy log under `files_skipped`.
- **Removed files are detected** — files that were in the previous deploy's
  `files_copied` but are no longer in the current source set are deleted from
  the destination and recorded in `files_removed`.
- **Destination folders are created automatically** if they don't exist.
- **Each destination is independent** — if one fails or is locked, the others still run.
- **The deploy log is NOT deployed** — it lives at the destination only.
  It won't be picked up by your file patterns unless you explicitly glob `*`.

---

## Exit Codes

| Code | Meaning |
|---|---|
| `0` | All files deployed successfully |
| `1` | One or more files failed (check log/console output) |
