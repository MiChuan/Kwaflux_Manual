---
name: kwaflux-mcp-setup
description: >-
  Wire, verify, and troubleshoot the KwaFlux MCP server (kwaflux-mcp) into Codex,
  either with `codex mcp add` or by editing the [mcp_servers.kwaflux] table in
  $CODEX_HOME/config.toml. Use when setting up KwaFlux MCP on a new machine, when
  the kwaflux tools are missing or fail to connect, or when that entry must be
  added, changed, or removed.
---

# KwaFlux MCP Setup (Codex)

Wire the `kwaflux-mcp` stdio server into Codex so the `kwaflux_*` tools show up under the `mcp__kwaflux` namespace.

**Never claim success without Step 6.** The failure mode here is silent: a broken entry produces no error, the tools simply never appear.

## What you are editing

Codex reads MCP servers from `[mcp_servers.<name>]` tables in `$CODEX_HOME/config.toml` (default `~/.codex/config.toml`). These are **global** — every project and every task sees them.

Two equivalent ways in; pick the one the user asked for:

```powershell
codex mcp add kwaflux -- node "C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs"
codex mcp remove kwaflux      # rollback
```

or hand-edit the table. Never write a `cwd`: the server resolves its own modules through `import.meta.url`.

`kwaflux-mcp` ships its own `node_modules` inside the install directory, so nothing has to be installed for it to run.

## Checklist

```
- [ ] 1. Confirm the KwaFlux MCP entry script exists
- [ ] 2. Confirm Node is resolvable
- [ ] 3. Confirm which CODEX_HOME is in play
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

Node 18+ is required. The script imports `@modelcontextprotocol/sdk` and `zod` from the `node_modules` shipped beside the entry script, so do not run `npm install`. `command = "node"` is fine while `node` is on PATH; otherwise use the absolute path, e.g. `C:\Program Files\nodejs\node.exe`.

## Step 3 — Confirm which CODEX_HOME is in play

```powershell
"CODEX_HOME = $env:CODEX_HOME"
"config     = $(if ($env:CODEX_HOME) { $env:CODEX_HOME } else { "$env:USERPROFILE\.codex" })\config.toml"
```

> **Common trap:** in a sandboxed or service-style shell, `$HOME`/`$USERPROFILE` may be missing and `codex mcp ...` dies with `Could not find home directory` / `failed to resolve CODEX_HOME`. Set it explicitly for that command:
>
> ```powershell
> $env:CODEX_HOME = 'C:\Users\<you>\.codex'; codex mcp add kwaflux -- node "<entry path>"
> ```
>
> Without it you may edit (or inspect) the wrong config and then "see no tools".

## Step 4 — Idempotency check

```powershell
$env:CODEX_HOME = 'C:\Users\<you>\.codex'; codex mcp get kwaflux
```

If it already prints a config, **do not add a second one**. `codex mcp add` on an existing name replaces it (good); hand-editing a second `[mcp_servers.kwaflux]` table produces a TOML duplicate-table error that can take the whole config down.

## Step 5 — Back up and write

CLI path (no backup needed, the tool rewrites the table):

```powershell
$env:CODEX_HOME = 'C:\Users\<you>\.codex'; codex mcp add kwaflux -- node "C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs"
```

Hand-edit path — copy first, then append:

```powershell
Copy-Item "$env:CODEX_HOME\config.toml" "$env:CODEX_HOME\config.toml.bak" -Force
```

```toml
[mcp_servers.kwaflux]
command = "node"
args = ['C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs']
```

- TOML single-quoted strings are literal, so a Windows path needs no `\\`.
- `startup_timeout_sec` is optional; this server starts in well under a second, so `20` is generous and the default is fine.
- Keep the file valid TOML — a broken `config.toml` affects far more than this one server. That is what the `.bak` is for.

## Step 6 — Verify

**A. Codex registered it.**

```powershell
$env:CODEX_HOME = 'C:\Users\<you>\.codex'; codex mcp get kwaflux
codex mcp list
```

**B. The server answers and the app is reachable.** Ask Codex to call `kwaflux_get_app_status`. A working link returns `running`, `signed_in`, `entitlement_allowed`, `app_version`, `host_version`, `protocol_version`. Then call `kwaflux_list_tasks` — an array (even empty) means the full path works.

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

Codex picks the new server up on its own in observed use, but if a task still shows no `mcp__kwaflux` tools, restart the Codex app.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No `mcp__kwaflux` tools | Server not registered in the active `CODEX_HOME` | Re-check Step 3/6A |
| No tools | App never reloaded the config | Restart Codex |
| `failed to resolve CODEX_HOME` / `Could not find home directory` | `HOME`/`USERPROFILE` unset in this shell | Set `$env:CODEX_HOME` explicitly |
| Config parse error on startup | Duplicate `[mcp_servers.kwaflux]` table, or invalid TOML | Restore `config.toml.bak`, re-apply one entry |
| `spawn ENOENT` / no child process | `node` not on PATH | Use an absolute `command` |
| `app_not_running: missing ...control.json` | KwaFlux is not running | Start KwaFlux |
| `connect_timeout` | App busy, mid-start, or at instance cap | Restart KwaFlux; raise `KWAFLUX_MCP_CONNECT_TIMEOUT_MS` |
| Write tools rejected | Not signed in, or no entitlement | Report `signed_in` / `entitlement_allowed` from Step 6B — an account issue, not a config issue |
| `mcp__kwaflux` tools exist but calls fail | KwaFlux not running / AgentControl endpoint gone | Start or restart KwaFlux, re-run 6B |

## Rollback

```powershell
codex mcp remove kwaflux
```

or restore the backup:

```powershell
Copy-Item "$env:CODEX_HOME\config.toml.bak" "$env:CODEX_HOME\config.toml" -Force
```

## Paths by platform

| | Windows | macOS / Linux |
|---|---|---|
| Entry script | `C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs` | under the KwaFlux app bundle — confirm on the machine |
| Codex config | `%USERPROFILE%\.codex\config.toml` | `~/.codex/config.toml` |
| Control endpoint | `%LOCALAPPDATA%\KwaFlux\agent\` | `~/Library/Application Support/KwaFlux/agent/` |

Only the `args` path and the config location change between machines; `command` and the table name stay the same.

## Optional fields

| Field | Default | Meaning |
|---|---|---|
| `startup_timeout_sec` | Codex default | Give up on the first handshake after this long |
| `env` (or `--env KEY=VALUE` on `codex mcp add`) | — | Extra child environment, merged over the inherited one |

To raise KwaFlux's own RPC budgets, pass them as env — as TOML:

```toml
[mcp_servers.kwaflux.env]
KWAFLUX_MCP_CALL_TIMEOUT_MS = "30000"
KWAFLUX_MCP_CONNECT_TIMEOUT_MS = "3000"
```

## After setup

Report the tool count (28 as of kwaflux-mcp 0.5.0) and hand the user the tool reference (`..\kwaflux-mcp-tools-reference.md`).

Two things to say before the user leans on these tools: the **15 `kwaflux_enhance_*` tools currently fail on every input** (`error_code: -10505`, `error_domain: vendor:bo`, at `progress: 0`, no output file), and **KwaFlux cannot export GIF** — its containers are mp4/mkv/mov/avi/webm/mp3/flac/wav and `enqueue_convert` takes neither a target format nor a time range. For GIF, use the bundled `<KwaFlux>\ffmpeg.exe`; details in the setup guide and the blackbox report.

Never set `compliance_ack` yourself when calling `kwaflux_enqueue_download`; it is the user's confirmation that they hold the rights to the content.
