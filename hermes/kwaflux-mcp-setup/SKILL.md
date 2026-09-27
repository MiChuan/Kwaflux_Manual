---
name: kwaflux-mcp-setup
description: >-
  Wire, verify, and troubleshoot the KwaFlux MCP server (kwaflux-mcp) into
  Hermes Agent by editing mcp_servers.kwaflux in $HERMES_HOME/config.yaml
  (Windows default %LOCALAPPDATA%\hermes\config.yaml). Use when setting up
  KwaFlux MCP on a new machine, when the kwaflux tools are missing or fail to
  connect, or when that entry must be added, changed, or removed. Also use
  when `hermes mcp test` reports that the mcp Python SDK is not installed.
---

# KwaFlux MCP Setup (Hermes)

Wire the `kwaflux-mcp` stdio server into Hermes so the `kwaflux_*` tools show up as `mcp__kwaflux__<tool>`.

**Never claim success without Step 7.** The failure mode here is silent: a broken entry produces no error, the tools simply never appear. A second silent failure is specific to Hermes: the YAML can be correct and the Node script can answer, while Hermes still loads nothing because the `mcp` Python extra is missing.

## What you are editing

Hermes reads MCP servers from `mcp_servers` in `$HERMES_HOME/config.yaml`.

| Platform | Default `HERMES_HOME` | Config |
|---|---|---|
| Windows | `%LOCALAPPDATA%\hermes` | `%LOCALAPPDATA%\hermes\config.yaml` |
| macOS / Linux | `~/.hermes` | `~/.hermes/config.yaml` |

`HERMES_HOME`, when set, wins over the default. On Windows the data directory is **not** `%USERPROFILE%\.hermes`. That path, when it exists, is often a source checkout (`%USERPROFILE%\.hermes\hermes-agent` on the machine this skill was written against). Writing the YAML there does not register the server.

Prefer hand-editing the YAML. `hermes mcp add` connects and then opens an interactive tool checklist; it stalls in a non-interactive shell. `hermes mcp remove` also asks for confirmation.

```powershell
hermes mcp list
hermes mcp test kwaflux
```

Never write a `cwd`: the server resolves its own modules through `import.meta.url`.

`kwaflux-mcp` ships its own `node_modules` inside the install directory, so nothing has to be npm-installed for the Node process. Hermes itself still needs the `mcp` Python extra.

## Checklist

```
- [ ] 1. Confirm the KwaFlux MCP entry script exists
- [ ] 2. Confirm Node is resolvable
- [ ] 3. Confirm which HERMES_HOME is in play
- [ ] 4. Confirm the mcp Python extra is installed
- [ ] 5. Check whether the entry is already present (idempotency)
- [ ] 6. Back up, then write the entry
- [ ] 7. Verify: hermes mcp test + live app status
```

## Step 1 — Locate the entry script

Default install root is `C:\Program Files\Kwaflux` on Windows, but the user may have installed elsewhere. Use the path that actually exists:

```powershell
Get-ChildItem "C:\Program Files\Kwaflux\Tools\kwaflux-mcp" -Force |
  Select-Object Mode, Length, Name
```

Expect `kwaflux_mcp.mjs` plus `agent_control.mjs`, `next_actions.mjs`, `generated\`, and `node_modules\@modelcontextprotocol\`. There is **no** `kwaflux\_mcp.mjs`. If the directory is absent, ask the user where KwaFlux is installed; do not guess.

## Step 2 — Confirm Node

```powershell
(Get-Command node).Source
node --version
```

Node 18+ is required. `command: node` is fine while `node` is on PATH; otherwise set `command` to the absolute path, for example `C:\Program Files\nodejs\node.exe`, as a YAML single-quoted string.

## Step 3 — Confirm which HERMES_HOME is in play

```powershell
"HERMES_HOME = $env:HERMES_HOME"
"user env   = $([Environment]::GetEnvironmentVariable('HERMES_HOME','User'))"
"config     = $(if ($env:HERMES_HOME) { $env:HERMES_HOME } else { "$env:LOCALAPPDATA\hermes" })\config.yaml"
```

On macOS and Linux, replace the fallback with `$env:USERPROFILE\.hermes` (`~/.hermes`). An already-open terminal does not see a User-level `HERMES_HOME` that was set after it started.

## Step 4 — Confirm the mcp Python extra

```powershell
hermes mcp test kwaflux
```

If this says `requires the 'mcp' Python SDK`, install the extra from the Hermes checkout, using that checkout's virtualenv Python. Official Windows installs keep the checkout at `%LOCALAPPDATA%\hermes\hermes-agent`. A git checkout may live elsewhere; use whichever tree contains `pyproject.toml` and `.venv` or `venv`.

```powershell
$repo = "$env:USERPROFILE\.hermes\hermes-agent"   # or the installer's hermes-agent directory
$py = Join-Path $repo ".venv\Scripts\python.exe"
Push-Location $repo
& $py -c "import pm; pm.sync_venv(['mcp'], explicit=True)"
Pop-Location
```

Do this once. Do not claim the server is wired until `hermes mcp test` gets past the SDK error.

## Step 5 — Idempotency check

```powershell
hermes mcp list
```

If `kwaflux` is already listed, do not append a second `kwaflux:` key. Edit the existing block. A duplicate YAML key corrupts or shadows the server entry.

## Step 6 — Back up and write

```powershell
$cfg = if ($env:HERMES_HOME) { "$env:HERMES_HOME\config.yaml" } else { "$env:LOCALAPPDATA\hermes\config.yaml" }
Copy-Item $cfg "$cfg.bak" -Force
```

Append only this block. Leave `model`, `platform_toolsets`, and every other key alone. Single-quoted YAML strings keep Windows backslashes literal.

```yaml
mcp_servers:
  kwaflux:
    command: node
    args:
      - 'C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs'
```

- Do not set `enabled: false`.
- Do not set `tools.include` unless the user asked for a subset. Omitting it enables all 28 tools (`hermes mcp list` shows `all`).
- Do not set `cwd`.
- Do not put API keys for KwaFlux in this block. Sign-in lives in the KwaFlux app.

`config.yaml.bak` sits next to the real file and is not loaded.

## Step 7 — Verify

**A. Hermes registered it.**

```powershell
hermes mcp list
```

Expect a `kwaflux` row: stdio via `node`, Tools `all`, Status enabled.

**B. Hermes can speak MCP and the app is reachable.**

```powershell
hermes mcp test kwaflux
```

Expect `Connected` and `Tools discovered: 28`. Then call `kwaflux_get_app_status` (registered name `mcp__kwaflux__kwaflux_get_app_status`). A working link returns `running`, `signed_in`, `entitlement_allowed`, `app_version`, `host_version`, `protocol_version`. App 1.0.5 also returns `agent_download_enabled`. Then call `kwaflux_list_tasks` — a `tasks` array, even empty, means the read path works.

**C. Headless check without asking the model**, useful when Hermes still shows no tools. Feed the server an `initialize` plus one `tools/call` over stdio:

```powershell
$msgs = @(
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"1"}}}',
  '{"jsonrpc":"2.0","method":"notifications/initialized"}',
  '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"kwaflux_get_app_status","arguments":{}}}'
) -join "`n"
$msgs | node "C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs"
```

`"running": true` means the Node server and the app are fine, so a remaining Hermes failure is the config path, the Python extra, or a session that has not reloaded. `app_not_running: EPERM ... control.json` from inside a sandboxed shell means the sandbox blocked `%LOCALAPPDATA%\KwaFlux\agent\`, not that the YAML is wrong.

An already-open `hermes` chat does not pick up a new server. Tell the user to run `/reload-mcp` in that session, or to quit and start `hermes` again.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `requires the 'mcp' Python SDK` | `mcp` extra not installed in the Hermes venv | Step 4 |
| No `kwaflux` row in `hermes mcp list` | Edited the wrong `HERMES_HOME` | Re-check Step 3 |
| `hermes mcp test` passes but the open chat has no tools | Session started before the write | `/reload-mcp` or restart `hermes` |
| Config parse error on startup | Duplicate `kwaflux:` key, or a double-quoted Windows path | Restore `config.yaml.bak`, re-apply one single-quoted entry |
| `spawn ENOENT` / no child process | `node` not on PATH | Absolute `command` |
| `app_not_running: missing ...control.json` | KwaFlux is not running | Start KwaFlux |
| `connect_timeout` | App busy, mid-start, or at instance cap | Restart KwaFlux; raise `KWAFLUX_MCP_CONNECT_TIMEOUT_MS` |
| Write tools rejected | Not signed in, or no entitlement | Report `signed_in` / `entitlement_allowed` from Step 7B |
| Download rejected, `agent_download_enabled: false` | Agent downloads are off in the app | User enables Settings → Agent access → Allow agents to start downloads |
| `mcp__kwaflux__*` tools exist but calls fail | KwaFlux not running / AgentControl endpoint gone | Start or restart KwaFlux, re-run 7B |
| `hermes` is not a command | `%LOCALAPPDATA%\hermes\bin` is not on the User PATH, or this terminal is older than that change | Open a new terminal |

## Rollback

Restore the backup (non-interactive):

```powershell
$cfg = if ($env:HERMES_HOME) { "$env:HERMES_HOME\config.yaml" } else { "$env:LOCALAPPDATA\hermes\config.yaml" }
Copy-Item "$cfg.bak" $cfg -Force
```

Or, in a terminal where a person can confirm: `hermes mcp remove kwaflux`.

## Paths by platform

| | Windows | macOS / Linux |
|---|---|---|
| Entry script | `C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs` | under the KwaFlux app bundle — confirm on the machine |
| Hermes config | `%LOCALAPPDATA%\hermes\config.yaml` | `~/.hermes/config.yaml` |
| Control endpoint | `%LOCALAPPDATA%\KwaFlux\agent\` | `~/Library/Application Support/KwaFlux/agent/` |

Only the `args` path and the config location change between machines; `command` and the key name stay the same. If `HERMES_HOME` is set, the config path follows it.

## Optional fields

| Field | Default | Meaning |
|---|---|---|
| `enabled` | on | `false` skips the server entirely |
| `timeout` | Hermes default | Per-tool-call timeout, seconds |
| `connect_timeout` | Hermes default | Initial connection timeout, seconds |
| `env` | — | Extra child environment |
| `tools.include` / `tools.exclude` | all tools | Filter by short tool name |

To raise KwaFlux's own RPC budgets:

```yaml
mcp_servers:
  kwaflux:
    command: node
    args:
      - 'C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs'
    env:
      KWAFLUX_MCP_CALL_TIMEOUT_MS: "30000"
      KWAFLUX_MCP_CONNECT_TIMEOUT_MS: "3000"
```

Codex `startup_timeout_sec` and DSH `failOnStartupError` / `reconnect` have no equivalent keys here.

## After setup

Report the tool count (28 as of kwaflux-mcp 0.5.0) and hand the user the tool reference (`..\kwaflux-mcp-tools-reference.md`).

Two things to say before the user leans on these tools: the **15 `kwaflux_enhance_*` tools currently fail on every input** (`error_code: -10505`, `error_domain: vendor:bo`, at `progress: 0`, no output file), and **KwaFlux cannot export GIF** — its containers are mp4/mkv/mov/avi/webm/mp3/flac/wav and `enqueue_convert` takes neither a target format nor a time range. For GIF, use the bundled `<KwaFlux>\ffmpeg.exe`; details in the setup guide and the blackbox report.

Never set `compliance_ack` yourself when calling `kwaflux_enqueue_download`; it is the user's confirmation that they hold the rights to the content.

Registered names are `mcp__kwaflux__` plus the short name (`mcp__kwaflux__kwaflux_get_app_status`). Names longer than 64 characters are clamped with a hash suffix; none of the current 28 tools hit that limit.
