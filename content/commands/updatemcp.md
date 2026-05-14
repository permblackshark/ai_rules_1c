---
description: Update installed 1C MCP servers from a local distribution according to its bundled instructions
---

# /updatemcp — update MCP servers from a distribution

This command updates already installed 1C MCP servers using a **local unpacked distribution** (usually newer than the original installation source). All update steps (new image versions, volume migrations, environment variable changes, container restarts, reindexing) are described **inside the distribution**. Do not invent or fetch external instructions; execute exactly what the distribution says.

Use `/installmcp` for first-time installation. Use `/checkmcp` to check current state.

## Steps

### 1. Ask for the distribution path

Ask the user for one parameter:

> Provide the path to the unpacked 1C MCP server distribution directory for update (for example, `D:\dist\mcp-1c`).

If the user did not provide a path, stop and ask for it. Do not guess, do not reuse a "previous" path from memory, and do not suggest `git pull`; the distribution must already exist locally and be unpacked.

### 2. Check the directory

After receiving the path:

1. Make sure the directory exists and is readable.
2. List its contents (PowerShell):

   ```powershell
   $dist = '<DIST_PATH>'
   if (-not (Test-Path -LiteralPath $dist)) { throw "Directory not found: $dist" }
   Get-ChildItem -LiteralPath $dist -Force | Select-Object Mode, Name, Length | Format-Table -AutoSize
   ```

3. Find the update instruction file. Candidates in priority order:
   - `UPDATE.md`, `update.md`
   - `UPGRADE.md`, `upgrade.md`
   - `CHANGELOG.md` if it describes the update procedure
   - `README.md` update section
   - `docs/UPDATE.md`, `docs/UPGRADE.md`
   - any top-level `.md` / `.txt` file whose name contains `update` / `upgrade` / `migration`

   If nothing is found, stop and tell the user: "No update instruction file (`UPDATE.md` / `UPGRADE.md` / `CHANGELOG.md`) was found in `<DIST_PATH>`. Please clarify the path or file name."

### 3. Read the instruction fully

Read the found file fully. If it links to other local distribution files (`./scripts/...`, `./migrations/...`, `./compose/docker-compose.yml`, `./env/...`), open those files too before running anything.

The source of truth is **only files inside `<DIST_PATH>`**. Do not replace them with `/checkmcp`, external documentation, or memory of a previous installation.

### 4. Capture pre-update state

Before changing anything, record the current state so there is something to compare against and roll back from:

```powershell
docker version --format '{{.Server.Version}}'
docker ps --all --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
docker images --format 'table {{.Repository}}:{{.Tag}}\t{{.ID}}\t{{.CreatedSince}}\t{{.Size}}'
```

Compare the output with the update instruction (container names, images, ports). If a container referenced by the instruction is absent from the system, that server was not installed; in this case **/updatemcp will not install it** (use `/installmcp` for installation), and will mark it as skipped.

If the instruction includes "back up volumes / indexes before update", do that **before** updating, not after.

### 5. Plan update

Before running commands:

1. Briefly summarize for the user (3-7 lines) what exactly will be done according to the distribution instruction: which images will be updated, which containers will restart, whether reindexing is needed and how long it may take, and which environment variables / volumes / license keys require input or changes.
2. Explicitly list risky steps: volume deletion, DB migration, stopping containers with active indexing, mandatory license key update.
3. List all missing data once and **ask the user for it in one message**, not one item at a time.
4. Wait for explicit user confirmation before starting the update.

### 6. Execute update

Execute steps exactly in the order described by the distribution instruction. Rules:

- Run commands from the project root unless the instruction explicitly says otherwise.
- If the instruction offers `docker compose pull && docker compose up -d` / a ready `update.ps1` inside the distribution, use that exact path instead of composing a command from the `/checkmcp` table or memory.
- Do not run `docker pull`, `docker compose up`, `docker rm`, or `docker volume rm` without user confirmation.
- Show each heavy command to the user before running it.
- After each step, briefly report what was updated (image → new digest), what restarted (container name → `Up X seconds` status), and which volumes were touched.

If the instruction requires reindexing (`1C-docs-mcp`, `1c-code-metadata-mcp`, `1c-graph-metadata-mcp`, `1c-ssl-mcp`), warn the user that reindexing may take tens of minutes or hours, and show the monitoring command (`docker logs -f <name>`).

### 7. Reconcile the active tool MCP config

After containers restart:

1. If the distribution instruction changed ports, service names, or MCP config schema, update the active config (`.cursor/mcp.json` / `.mcp.json` / `.opencode/opencode.json` / `.codex/config.toml`). If 1c-rules is installed, this can be done through `/update` (it re-renders the config by adapter), but only if the changes are compatible with `content/mcp-servers.json`. Otherwise edit the config manually according to the distribution instruction.
2. Ask the user to restart the client (Cursor / Claude Code / Codex / OpenCode / Kilo Code) so it reinitializes the MCP session.

### 8. Final check

After the client restart, run `/checkmcp`. All updated servers should get **TOOLS_OK** (or **HTTP_OK** if reindexing is still running). If anything remains **TOOLS_MISSING** / **HTTP_DOWN**, return to Steps 3-6 and compare the executed steps with the distribution instruction.

## Rollback

If the update broke the working state:

1. Find the rollback / `rollback` / `downgrade` section in the distribution instruction and follow it.
2. If no such section exists, use the typical recipe:
   - stop and remove new containers (`docker stop <name>` → `docker rm <name>`);
   - start containers with the old image tag from the `docker images` snapshot captured in Step 4;
   - restore volumes from backup if one was made.
3. Tell the user that rollback is complete and run `/checkmcp` again.

## Final report

Short user summary:

- distribution path used;
- instruction file that was read;
- servers actually updated (container name, port, previous → current image/digest);
- servers skipped and why (not installed, no `LICENSE_KEY`, no dump, etc.);
- next steps if reindexing is still running.

## Limits

- The command **does not replace** the distribution instruction and does not add its own steps.
- The command **does not download** the distribution; the user downloads and unpacks it.
- The command **does not install** servers that are not already in the system; use `/installmcp` for that.
- The command **does not run** `docker pull` / `docker compose up` / `docker volume rm` without user confirmation.
