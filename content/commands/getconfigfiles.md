---
description: Extract configuration objects from infobase to files for editing
---

# /getconfigfiles — extract configuration objects from an infobase

See the full rule in `content/rules/getconfigfiles.md`.

## Parameters

All paths and identifiers come from `.dev.env` placeholders. If a value is unknown, ask the user.

| Placeholder | Purpose |
|---|---|
| `{PLATFORM_PATH}` | 1C platform installation directory containing `bin\1cv8.exe` |
| `{INFOBASE_PATH}` | File infobase path or server connection string |
| `{IB_USER}` | Infobase user |
| `{IB_PASSWORD}` | Password, if set |
| `{EXPORT_PATH}` | Source export directory |
| `{EXTENSION_NAME}` | Extension name; omit for the main configuration |
| `{LOG_PATH}` | Designer log file |
| `{IBCMD_CONFIG}` | Path to standalone server `config.yml` for `ibcmd`, optional |

## Steps

1. Build the object list in `repoobjects.txt` (one fully qualified metadata object name per line). Collect the list through `metadatasearch` / `search_metadata`.

2. Choose the tool:
   - If `Test-Path '{PLATFORM_PATH}\bin\ibcmd.exe'` is true and `IBCMD_CONFIG` is filled, use **2a (`ibcmd`)**.
   - Otherwise use **2b (Designer)**. `ibcmd infobase config` does not apply to 1C cluster infobases; for server cluster infobases always use Designer.

**2a.** Partial export through `ibcmd` (objects are read from `repoobjects.txt` and passed as positional arguments):

```powershell
$objects = Get-Content repoobjects.txt | Where-Object { $_.Trim() -ne '' }
& '{PLATFORM_PATH}\bin\ibcmd.exe' infobase config export objects `
    --config='{IBCMD_CONFIG}' `
    --user='{IB_USER}' `
    --password='{IB_PASSWORD}' `
    --recursive `
    --out='{EXPORT_PATH}' `
    --extension={EXTENSION_NAME} `
    @objects *>&1 | Tee-Object -FilePath '{LOG_PATH}'
```

Remove empty optional keys (`--user`, `--password`, `--extension`). `--recursive` exports child objects (attributes, tabular sections, forms, templates).

**2b.** Partial export through Designer (fallback):

```powershell
& '{PLATFORM_PATH}\bin\1cv8.exe' DESIGNER `
    /F '{INFOBASE_PATH}' `
    /N '{IB_USER}' `
    /P '{IB_PASSWORD}' `
    /DisableStartupMessages `
    /DumpConfigToFiles {EXPORT_PATH} `
    -listFile repoobjects.txt `
    -Extension {EXTENSION_NAME} `
    /Out {LOG_PATH}
```

Objects are exported fully and strictly into `{EXPORT_PATH}`; no extra subdirectories are created. When exporting the main configuration, remove `-Extension {EXTENSION_NAME}`.

3. Check `{LOG_PATH}` for errors.
