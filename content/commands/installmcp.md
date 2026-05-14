---
description: Install 1C MCP servers from a local distribution according to its bundled instructions
---

# /installmcp — install MCP servers from a distribution

This command performs the first-time installation of 1C MCP servers using a **local unpacked distribution** that the user downloaded and unpacked. All steps (images, volumes, environment variables, license keys, startup order) are described **inside the distribution**. Do not invent or fetch external instructions; execute exactly what the distribution says.

Use `/checkmcp` to check already installed servers. Use `/updatemcp` to update an already installed set.

## Steps

### 1. Ask for the distribution path

Ask the user for one parameter:

> Provide the path to the unpacked 1C MCP server distribution directory (for example, `D:\dist\mcp-1c`).

If the user did not provide a path, stop and ask for it. Do not guess, do not search "defaults", and do not suggest `git clone`; the distribution must already exist locally and be unpacked.

### 2. Check the directory

After receiving the path:

1. Make sure the directory exists and is readable.
2. List its contents (PowerShell):

   ```powershell
   $dist = '<DIST_PATH>'
   if (-not (Test-Path -LiteralPath $dist)) { throw "Directory not found: $dist" }
   Get-ChildItem -LiteralPath $dist -Force | Select-Object Mode, Name, Length | Format-Table -AutoSize
   ```

3. Find the installation instruction file. Candidates in priority order:
   - `INSTALL.md`, `install.md`
   - `README.md`, `readme.md`
   - `SETUP.md`, `setup.md`
   - `docs/INSTALL.md`, `docs/README.md`
   - any top-level `.md` / `.txt` file whose name contains `install` / `setup` / `quickstart` / `getting-started`

   If nothing is found, stop and tell the user: "No installation instruction file (`INSTALL.md` / `README.md` / `SETUP.md`) was found in `<DIST_PATH>`. Please clarify the path or file name."

### 3. Read the instruction fully

Read the found file fully, not just the first N lines. If it links to other local distribution files (`./scripts/...`, `./compose/docker-compose.yml`, `./env/...`, `./certs/...`, etc.), open those files too before running anything.

The source of truth at this step is **only files inside `<DIST_PATH>`**. Do not replace them with instructions from `/checkmcp`, `https://docs.onerpa.ru/mcp-servery-1c`, `https://vibecoding1c.ru/mcp_server`, `content/mcp-servers.json`, etc. Those sources may only be used to cross-check an already executed step, not to replace it.

### 4. Plan installation

Before running commands:

1. Briefly summarize for the user (3-7 lines) what exactly will be done according to the distribution instruction: which servers will start, which volumes/directories will be created, which environment variables are required, and which values must be entered manually (license keys, tokens, configuration dump path, platform `bin` path).
2. List all missing data once and **ask the user for it in one message**, not one item at a time.
3. If the instruction mentions additional environment requirements (Docker Desktop, Neo4j, open ports, disk space for indexes, internet access for `docker pull`), list them explicitly and confirm they are met before running.

### 5. Execute installation

Execute steps exactly in the order described by the distribution instruction. Rules:

- Run commands from the project root unless the instruction explicitly says otherwise.
- If the instruction offers `docker compose up -d` / `docker run ...` / a ready `install.ps1` inside the distribution, use that exact path instead of composing a command from the `/checkmcp` table.
- Do not run `docker run` or `docker compose up` without user confirmation; images may be several GB.
- Show each heavy command to the user before running it (one command per message or block).
- After each major step, briefly report what started and which container/name/port is involved before moving on.

If the instruction includes initial RAG indexing (`1C-docs-mcp`, `1c-code-metadata-mcp`, `1c-graph-metadata-mcp`, `1c-ssl-mcp`), warn the user that first launch may take tens of minutes or hours, and show the monitoring command (`docker logs -f <name>`).

### 6. Register servers in the active tool

After containers are up:

1. If the project has `.ai-rules.json`, the active tool MCP config (`.cursor/mcp.json` / `.mcp.json` / `.opencode/opencode.json` / `.codex/config.toml`) should already be rendered by the 1c-rules installer; re-render through `/update` if needed.
2. If 1c-rules was not installed, add servers to the active tool MCP config manually by the schema from `adapters/<tool>.yaml → mcp.schema` or by the distribution instruction (if it provides a ready JSON/TOML fragment, use it).
3. Ask the user to restart the client (Cursor / Claude Code / Codex / OpenCode / Kilo Code) so it reinitializes the MCP session.

### 7. Final check

After the client restart, run `/checkmcp`. All servers from the distribution should get **TOOLS_OK** (or at least **HTTP_OK** if indexing is still running). If anything remains **TOOLS_MISSING** / **HTTP_DOWN**, return to Steps 3-5 and compare the executed steps with the distribution instruction.

## Final report

Short user summary:

- distribution path used;
- instruction file that was read;
- servers actually started (container name, port, image);
- servers skipped and why (for example no `LICENSE_KEY`, no configuration dump, separate Neo4j setup required);
- next steps if indexing is still running.

## Limits

- The command **does not replace** the distribution instruction and does not add its own steps. If the instruction lacks something, ask the user instead of filling gaps from memory.
- The command **does not download** the distribution. The user is expected to download and unpack it before invoking the command.
- The command **does not run** `docker run` / `docker compose up` without explicit user confirmation.
- The graph server (`1c-graph-metadata-mcp`) usually requires separate Neo4j setup and indexing. If that is part of the instruction, execute it strictly by the instruction, not by generic recipes from `/checkmcp`.
