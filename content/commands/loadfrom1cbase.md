---
description: Dump the configuration from the infobase defined in .dev.env into the current repository files
---

# /loadfrom1cbase — dump from infobase to repository

Full configuration dump (`/DumpConfigToFiles`) from the infobase defined in `.dev.env` into the current repository directory.

For a partial object-by-object export, use `/getconfigfiles` (rule `getconfigfiles.md`, via `repoobjects.txt`).

## Step 0. Check `.dev.env` parameters

`.dev.env` is the single source of truth for connection parameters (created by the 1c-rules installer at the project root). If it is missing, ask the user to run `install.ps1 init` or manually copy `.dev.env.example` to `.dev.env`.

If the project still has legacy `infobasesettings.md`, migrate values to `.dev.env` (same key names, `KEY=value` format instead of a markdown list), preserving already-filled `.dev.env` keys, and delete the legacy file after successful migration. The ruleset has no other connection-settings location.

Used `.dev.env` keys:

| Key | Purpose |
|---|---|
| `PLATFORM_PATH` | Platform installation directory containing `bin\1cv8.exe` |
| `INFOBASE_KIND` | `file` or `server` |
| `INFOBASE_PATH` | File infobase path or server connection string |
| `IB_USER` | Infobase user; empty means no authentication |
| `IB_PASSWORD` | Password; empty means no password |
| `EXTENSION_NAME` | Extension name; empty means main configuration |
| `EXPORT_PATH` | Dump directory; empty means repository root |
| `LOG_PATH` | Designer log file |
| `IBCMD_CONFIG` | Path to standalone server `config.yml` for `ibcmd`, optional |

If critical fields are empty (`INFOBASE_PATH`, `PLATFORM_PATH`), ask the user and write the values to `.dev.env`; do not guess.

## Step 1. Choose tool: `ibcmd` or Designer

1. Check whether the utility exists: `Test-Path '{PLATFORM_PATH}\bin\ibcmd.exe'`.
2. Check whether `IBCMD_CONFIG` is filled in `.dev.env`.
3. If **both conditions are true**, use **Step 2a (`ibcmd`)**.
4. Otherwise use **Step 2b (Designer)**.

`ibcmd infobase config` does not apply to 1C cluster infobases; for server cluster infobases always use Designer.

## Step 2a. Export through `ibcmd` (preferred)

```powershell
& '{PLATFORM_PATH}\bin\ibcmd.exe' infobase config export `
    --config='{IBCMD_CONFIG}' `
    --user='{IB_USER}' `
    --password='{IB_PASSWORD}' `
    --extension={EXTENSION_NAME} `
    '{EXPORT_PATH}' *>&1 | Tee-Object -FilePath '{LOG_PATH}'
```

Remove empty optional keys (`--user`, `--password`, `--extension`). For repeated exports into the same directory with a valid `ConfigDumpInfo.xml`, add `--sync` to export only changed files.

`ibcmd` writes diagnostics to stdout/stderr; `Tee-Object` duplicates it into `{LOG_PATH}`. Continue to **Step 3**.

## Step 2b. Export through Designer (fallback)

Map `.dev.env` keys to Designer flags:

| Field | Flag |
|---|---|
| `INFOBASE_KIND=file` | `/F '{INFOBASE_PATH}'` |
| `INFOBASE_KIND=server` | `/S '{INFOBASE_PATH}'` |
| `IB_USER` when not empty | `/N '{IB_USER}'` |
| `IB_PASSWORD` when not empty | `/P '{IB_PASSWORD}'` |
| `EXTENSION_NAME` when not empty | `-Extension {EXTENSION_NAME}` |

```powershell
& '{PLATFORM_PATH}\bin\1cv8.exe' DESIGNER `
    /F '{INFOBASE_PATH}' `
    /N '{IB_USER}' `
    /P '{IB_PASSWORD}' `
    /DisableStartupMessages `
    /DumpConfigToFiles '{EXPORT_PATH}' `
    -Extension {EXTENSION_NAME} `
    /Out '{LOG_PATH}'
```

Remove empty optional keys (`/N`, `/P`, `-Extension`). When exporting the main configuration, remove `-Extension {EXTENSION_NAME}` entirely. For a server infobase, replace `/F` with `/S`.

The export goes **strictly into the specified directory**; no extra subdirectories are created.

## Step 3. Check result

1. Read `{LOG_PATH}`:
   - For Designer, success means `Конфигурация успешно сохранена` / `Configuration successfully saved`.
   - For `ibcmd`, success means no `error` / `ошибка` lines and no non-zero exit code reported.
2. If errors exist, show the relevant log fragment to the user and stop.
3. Briefly list which top-level object directories appeared or changed according to `git status`, without content diffs.
