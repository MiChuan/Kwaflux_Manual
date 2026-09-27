---
name: kwaflux-mcp-setup
description: >-
  Wire, verify, and troubleshoot the KwaFlux MCP server (kwaflux-mcp) into
  Cursor, either with `cursor --add-mcp` or by editing mcpServers.kwaflux in
  ~/.cursor/mcp.json (global) or .cursor/mcp.json (project). Use when setting
  up KwaFlux MCP on a new machine, when the kwaflux tools are missing or fail
  to connect, or when that entry must be added, changed, or removed.
---

# KwaFlux MCP Setup (Cursor)

Wire the `kwaflux-mcp` stdio server into Cursor so the `kwaflux_*` tools show up. A user-level `~/.cursor/mcp.json` entry named `kwaflux` is exposed to the agent as namespace `user-kwaflux`.

**Never claim success without Step 6.** The failure mode here is silent: a broken entry produces no error, the tools simply never appear.

This skill is for the **local Cursor app / local agent**. Cloud Agents cannot spawn a local `node` child that talks to a local KwaFlux instance.

## What you are editing

Cursor reads MCP servers from `mcp.json`:

| Scope | File | Who sees it |
|---|---|---|
| Global (default) | `~/.cursor/mcp.json` → Windows `%USERPROFILE%\.cursor\mcp.json` | Every project on this machine |
| Project | `<workspace>/.cursor/mcp.json` | This workspace only |

Two equivalent ways in; pick the one the user asked for:

```powershell
cursor --add-mcp '{"name":"kwaflux","command":"node","args":["C:\\Program Files\\Kwaflux\\Tools\\kwaflux-mcp\\kwaflux_mcp.mjs"]}'
```

or hand-edit the JSON. Never write a `cwd`: the server resolves its own modules through `import.meta.url`.

`kwaflux-mcp` ships its own `node_modules` inside the install directory, so nothing has to be installed for it to run.

Prefer **global** unless the user asked for project-only. KwaFlux is a local app, not a repo tool.

## Checklist

```
- [ ] 1. Confirm the KwaFlux MCP entry script exists
- [ ] 2. Confirm Node is resolvable
- [ ] 3. Confirm which mcp.json is in play
- [ ] 4. Check whether the entry is already present (idempotency)
- [ ] 5. Back up, then write the entry
- [ ] 6. Verify: registration + live tool call
```

## Step 1 — Locate the entry script

Default install root is `C:\Program Files\Kwaflux` on Windows, but the user may have installed elsewhere. Use the path that actually exists:

```powershell
Get-ChildItem "C:\Program Files\Kwaflux\Tools\kwaflux-mcp" -Force |
  Select-Object Mode, Length, Name
```

Expect `kwaflux_mcp.mjs` plus `agent_control.mjs`, `next_actions.mjs`, `generated\`, and `node_modules\@modelcontextprotocol\`. There is **no** `kwaflux\_mcp.mjs` — that path shows up in some hand-written configs and always fails. If the directory is absent, ask the user where KwaFlux is installed; do not guess.

## Step 2 — Confirm Node

```powershell
(Get-Command node).Source
node --version
```

Node 18+ is required. The script imports `@modelcontextprotocol/sdk` and `zod` from the `node_modules` shipped beside the entry script, so do not run `npm install`. `"command": "node"` is fine while `node` is on PATH; otherwise use the absolute path, e.g. `C:\\Program Files\\nodejs\\node.exe`.

## Step 3 — Confirm which mcp.json is in play

```powershell
"global  = $env:USERPROFILE\.cursor\mcp.json"
"project = $(Get-Location)\.cursor\mcp.json"
Test-Path "$env:USERPROFILE\.cursor\mcp.json"
Test-Path ".\.cursor\mcp.json"
```

> **Common trap:** Cursor has **two** config layers (unlike Codex, which is global-only). Writing the project file leaves other workspaces without tools; writing the global file is what "configure it on this machine" usually means. If the user uses Cloud Agents, say plainly that this stdio server will not work there.

## Step 4 — Idempotency check

```powershell
$global = "$env:USERPROFILE\.cursor\mcp.json"
if (Test-Path $global) {
  Get-Content $global -Raw
} else {
  "no global mcp.json yet"
}
```

If `mcpServers.kwaflux` already exists, **do not add a second key**. JSON duplicate keys are undefined (last-or-first wins, or the file fails to parse). Update the existing object or stop.

## Step 5 — Back up and write

CLI path (writes the user-profile / global file; add `--mcp-workspace` only when the user asked for project scope):

```powershell
cursor --add-mcp '{"name":"kwaflux","command":"node","args":["C:\\Program Files\\Kwaflux\\Tools\\kwaflux-mcp\\kwaflux_mcp.mjs"]}'
```

Hand-edit path — copy first if the file exists, then write a valid JSON object:

```powershell
$mcp = "$env:USERPROFILE\.cursor\mcp.json"
if (Test-Path $mcp) { Copy-Item $mcp "$mcp.bak" -Force }
```

```json
{
  "mcpServers": {
    "kwaflux": {
      "command": "node",
      "args": ["C:\\Program Files\\Kwaflux\\Tools\\kwaflux-mcp\\kwaflux_mcp.mjs"]
    }
  }
}
```

- JSON strings need `\\` for Windows backslashes. A single `\` is invalid JSON and the whole file is ignored.
- Forward slashes also work: `"C:/Program Files/Kwaflux/Tools/kwaflux-mcp/kwaflux_mcp.mjs"`.
- Do not add `"type": "stdio"`. The working user `mcp.json` on this machine has only `command` and `args`.
- Keep the file valid JSON — a trailing comma or a comment will take down every MCP server in that file. That is what the `.bak` is for.
- Do not merge by concatenating two root objects.

## Step 6 — Verify

**A. The file parses and the key is there.**

```powershell
Get-Content "$env:USERPROFILE\.cursor\mcp.json" -Raw | ConvertFrom-Json |
  Select-Object -ExpandProperty mcpServers |
  Select-Object -ExpandProperty kwaflux
```

If `agent` is on PATH:

```powershell
agent mcp list
agent mcp list-tools kwaflux
```

**B. Cursor loaded the namespace.** After the edit, toggle the server in **Customize → MCP**, or restart Cursor. Then discover tools:

- Agent session: `GetDynamicTools` with `namespace: "user-kwaflux"` when the entry is in the user `mcp.json`. Expect ~28 tools (`kwaflux_get_app_status`, `kwaflux_list_tasks`, …). The config key stays `kwaflux`; the `user-` prefix is added by Cursor for that global file. If the tools were added in a project `mcp.json` instead, use the namespace `GetDynamicTools` actually lists — do not guess it.
- Chat: the tools appear under Available Tools.

Call `kwaflux_get_app_status`. A working link returns `running`, `signed_in`, `entitlement_allowed`, `app_version`, `host_version`, `protocol_version`. Then call `kwaflux_list_tasks` — an array (even empty) means the full path works.

In this agent, invoke via `CallDynamicTool` (`namespace: "user-kwaflux"`, `toolName: "kwaflux_get_app_status"`, plus `mcpDetails`). Do not invent a `mcp__kwaflux__*` or `mcp__kwaflux::*` prefix — those are DSH / Hermes and Codex spellings.

**C. Headless check without asking the model**, useful when tools appear to be missing. Feed the server an `initialize` plus one `tools/call` over stdio:

```powershell
$msgs = @(
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"1"}}}',
  '{"jsonrpc":"2.0","method":"notifications/initialized"}',
  '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"kwaflux_get_app_status","arguments":{}}}'
) -join "`n"
$msgs | node "C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs"
```

`"running": true` means the chain is good. `app_not_running: EPERM ... control.json` from inside a sandboxed shell means the sandbox blocked `%LOCALAPPDATA%\KwaFlux\agent\`, not that the config is wrong — rerun outside the sandbox or just use path B.

The stdio server stays open waiting for more JSON-RPC; stop the leftover `node` process after the probe.

If Customize still shows the server as disconnected, open **Output → MCP Logs**.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No `kwaflux` namespace | Wrote the wrong `mcp.json` (project vs global) | Re-check Step 3/6A |
| No tools | Cursor never reloaded the file | Toggle the server in Customize, or restart Cursor |
| File ignored / all MCP dead | Invalid JSON (`\` not escaped, trailing comma, comments) | Restore `mcp.json.bak`, re-apply one entry |
| Duplicate / missing tools | Two `kwaflux` keys, or project + global both defined differently | Keep exactly one intended entry |
| `spawn ENOENT` / no child process | `node` not on PATH | Use an absolute `command` |
| `app_not_running: missing ...control.json` | KwaFlux is not running | Start KwaFlux |
| `connect_timeout` | App busy, mid-start, or at instance cap | Restart KwaFlux; raise `KWAFLUX_MCP_CONNECT_TIMEOUT_MS` |
| Write tools rejected | Not signed in, or no entitlement | Report `signed_in` / `entitlement_allowed` from Step 6B — an account issue, not a config issue |
| Tools exist but calls fail | KwaFlux not running / AgentControl endpoint gone | Start or restart KwaFlux, re-run 6B |
| Works in IDE, fails in Cloud Agent | Local stdio cannot reach this machine's KwaFlux | Say so; do not try to "fix" it with a remote URL |

## Rollback

Delete the `kwaflux` key from the same `mcp.json` you edited, or restore the backup:

```powershell
Copy-Item "$env:USERPROFILE\.cursor\mcp.json.bak" "$env:USERPROFILE\.cursor\mcp.json" -Force
```

Then toggle the server off in Customize (or restart Cursor).

## Paths by platform

| | Windows | macOS / Linux |
|---|---|---|
| Entry script | `C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs` | under the KwaFlux app bundle — confirm on the machine |
| Global config | `%USERPROFILE%\.cursor\mcp.json` | `~/.cursor/mcp.json` |
| Project config | `<workspace>\.cursor\mcp.json` | `<workspace>/.cursor/mcp.json` |
| Control endpoint | `%LOCALAPPDATA%\KwaFlux\agent\` | `~/Library/Application Support/KwaFlux/agent/` |

Only the `args` path and the config location change between machines; `command` and the server name stay the same.

## Optional fields

| Field | Default | Meaning |
|---|---|---|
| `type` | omitted | Not required. The working entry on this machine has no `type` |
| `env` | — | Extra child environment, merged over the inherited one |
| `envFile` | — | Extra env file; stdio only |
| interpolation | — | `${env:NAME}`, `${userHome}`, `${workspaceFolder}`, `${pathSeparator}` in `command` / `args` / `env` |

DSH's `failOnStartupError` / `reconnect.*` / `toolCallTimeoutMs` and Codex's `startup_timeout_sec` are **not** Cursor keys.

To raise KwaFlux's own RPC budgets, pass them as env:

```json
{
  "mcpServers": {
    "kwaflux": {
      "command": "node",
      "args": ["C:\\Program Files\\Kwaflux\\Tools\\kwaflux-mcp\\kwaflux_mcp.mjs"],
      "env": {
        "KWAFLUX_MCP_CALL_TIMEOUT_MS": "30000",
        "KWAFLUX_MCP_CONNECT_TIMEOUT_MS": "3000"
      }
    }
  }
}
```

## After setup

Report the tool count (28 as of kwaflux-mcp 0.5.0) and hand the user the tool reference (`../kwaflux-mcp-tools-reference.md`).

Two things to say before the user leans on these tools: the **15 `kwaflux_enhance_*` tools currently fail on every input** (`error_code: -10505`, `error_domain: vendor:bo`, at `progress: 0`, no output file), and **KwaFlux cannot export GIF** — its containers are mp4/mkv/mov/avi/webm/mp3/flac/wav and `enqueue_convert` takes neither a target format nor a time range. For GIF, use the bundled `<KwaFlux>\ffmpeg.exe`; details in the setup guide and the blackbox report.

Never set `compliance_ack` yourself when calling `kwaflux_enqueue_download`; it is the user's confirmation that they hold the rights to the content.
