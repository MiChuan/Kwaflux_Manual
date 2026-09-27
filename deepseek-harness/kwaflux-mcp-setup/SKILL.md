---
name: kwaflux-mcp-setup
description: >-
  Install, verify, and troubleshoot the KwaFlux MCP server (kwaflux-mcp) in a
  DeepSeek Harness profile by editing that profile's cordis.patch.yml. Use when
  configuring KwaFlux MCP on a new machine, when the mcp__kwaflux__* tools are
  missing or fail to connect, or when a kwaflux MCP entry must be added,
  changed, or removed.
---

# KwaFlux MCP Setup

Wire the `kwaflux-mcp` stdio server into a DSH profile so the `mcp__kwaflux__*` tools become available.

**Never claim success without running the verification in Step 6.** The whole point of this skill is a configuration whose failure mode is silent (tools simply do not appear).

## What you are editing

DSH composes each profile from layered patches. The only file to touch is the profile's user patch layer:

```
<DSH_HOME>/profiles/<profile>/cordis.patch.yml
```

Do **not** edit `cordis.yml` — it is the empty root and its own header says to edit the patch file instead.

`@deepseek-ai/dsh-mcp-client` ships inside the DSH installation, so inserting it needs **no `pnpm install`** even when the profile has no `node_modules`.

## Checklist

Copy and track:

```
- [ ] 1. Confirm the KwaFlux MCP entry script exists
- [ ] 2. Confirm Node is resolvable
- [ ] 3. Determine the active DSH profile
- [ ] 4. Check whether the entry is already present (idempotency)
- [ ] 5. Back up, then write the insert entry
- [ ] 6. Verify: child process + live tool call
```

## Step 1 — Locate the entry script

Default install root is `C:\Program Files\Kwaflux` on Windows, but the user may have chosen another location. Use the path that actually exists:

```powershell
Get-ChildItem "C:\Program Files\Kwaflux\Tools\kwaflux-mcp" -Force |
  Select-Object Mode, Length, Name
```

Expect `kwaflux_mcp.mjs` plus `agent_control.mjs`, `next_actions.mjs`, `generated/`, and `node_modules/@modelcontextprotocol/`. If the directory is absent, ask the user where KwaFlux is installed — do not guess.

Ignore a missing `SKILL.md` in that directory: the server falls back to built-in instruction text.

## Step 2 — Confirm Node

```powershell
(Get-Command node).Source
node --version
```

Node 18+ is required. The script imports `@modelcontextprotocol/sdk` and `zod` from the `node_modules` shipped beside the entry script, so do not run `npm install`. DSH scrubs child env names matching `/KEY|PASSWORD|SECRET|TOKEN/i` and all `DSH_*`, **but `PATH` survives**, so `command: 'node'` resolves normally.

If `node` is not on PATH, use an absolute path in `command` instead.

## Step 3 — Determine the active profile

Read the environment DSH exposes to you:

```powershell
"DSH_HOME        = $env:DSH_HOME"
"DSH_PROFILE     = $env:DSH_PROFILE"
"DSH_PROFILE_DIR = $env:DSH_PROFILE_DIR"
```

Cross-check against the running host if needed:

```powershell
Get-CimInstance Win32_Process -Filter "Name='DeepSeek Harness.exe'" |
  Where-Object { $_.CommandLine -like '*dsh-desktop-host*' } |
  ForEach-Object { $_.CommandLine }
```

The third positional argument of that command line is the profile directory.

> `desktop` and `web` are separate profiles. The desktop app uses `desktop`; `dsh web` uses `web`. Writing to the wrong one produces "configured but no tools appear". If the user uses both, ask whether to patch both.

## Step 4 — Idempotency check

Read the patch file first. If an entry with `serverName: kwaflux` already exists, **do not add a second one** — two entries sharing a server name make the later one fail to load. Repair the existing entry or stop.

## Step 5 — Back up and write

```powershell
$patch = "$env:DSH_PROFILE_DIR\cordis.patch.yml"
Copy-Item $patch "$patch.bak" -Force
```

The patch file is a **top-level YAML array**. Append this entry, matching the indentation exactly (`insert:` holds a nested list, so its children are indented one extra level):

```yaml
- insert:
    - id: mcp-kwaflux
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: kwaflux
        transport: stdio
        command: 'node'
        args: ['C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs']
```

Notes:

- `serverName` must match `[A-Za-z0-9_-]{1,32}` and be unique within the profile.
- In a YAML single-quoted scalar a backslash is a literal, so a Windows path needs no `\\` escaping.
- `id` is a local identifier; keep it unique.
- Keep the file valid YAML — a broken patch can stop the GUI from starting. That is what the `.bak` is for.

## Step 6 — Verify

**A. The server process was actually spawned.**

```powershell
Get-CimInstance Win32_Process -Filter "Name='node.exe'" |
  Where-Object { $_.CommandLine -like '*kwaflux*' } |
  Select-Object ProcessId, CreationDate, CommandLine
```

**B. The tools answer.** Call `mcp__kwaflux__kwaflux_get_app_status`. A working link returns `running`, `signed_in`, `entitlement_allowed`, and version fields.

**C. The task API answers.** Call `mcp__kwaflux__kwaflux_list_tasks`; an array (even empty) means the full path works.

DSH hot-reloads the patch file, so no restart is normally needed (observed: child process up ~7 s after the edit). If nothing appears, restart DSH.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No `mcp__kwaflux__*` tools | Wrong profile patched | Re-check Step 3; patch the right file |
| No tools | Invalid YAML structure | Restore `.bak`, re-apply matching the template |
| No tools | Host did not reload | Restart DSH |
| Two same-name entries | Duplicate insert | Keep exactly one |
| `app_not_running: missing ...control.json` | KwaFlux app is not running | Start KwaFlux |
| `spawn ENOENT` / no child process | `node` not on PATH | Use an absolute `command` |
| `connect_timeout` | App busy, mid-start, or at instance cap | Restart KwaFlux; raise `KWAFLUX_MCP_CONNECT_TIMEOUT_MS` |
| Write tools rejected | Not signed in, or no entitlement | Report `signed_in` / `entitlement_allowed` from Step 6B; this is an account issue, not a config issue |
| GUI will not start | Patch badly broken | Restore `cordis.patch.yml.bak` |

A failed MCP entry does **not** abort DSH startup by default (`failOnStartupError: false`), so misconfiguration usually shows up as missing tools rather than a crash.

## Rollback

```powershell
Copy-Item "$env:DSH_PROFILE_DIR\cordis.patch.yml.bak" `
          "$env:DSH_PROFILE_DIR\cordis.patch.yml" -Force
```

Or delete just the `- insert:` entry with its nested items. A leftover `.bak` is harmless — the profile loader only reads `cordis.yml` and `cordis.patch.yml`.

## Paths by platform

| | Windows | macOS / Linux |
|---|---|---|
| Entry script | `C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs` | under the KwaFlux app bundle — confirm on the machine |
| Control endpoint | `%LOCALAPPDATA%\KwaFlux\agent\` | `~/Library/Application Support/KwaFlux/agent/` |
| Patch file | `%USERPROFILE%\.dsh\profiles\<profile>\cordis.patch.yml` | `~/.dsh/profiles/<profile>/cordis.patch.yml` |

Only the `args` path and the patch location change between machines; `name`, `transport`, and `serverName` stay the same.

## Optional config fields

| Field | Default | Meaning |
|---|---|---|
| `failOnStartupError` | `false` | Reject plugin activation when the first connect or tool sync fails. Leave at default |
| `toolCallTimeoutMs` | `60000` | Per `tools/call` timeout |
| `reconnect.*` | enabled, 500 ms → 30 s, 10 attempts | Automatic reconnect backoff |
| `env` | — | Extra child env, merged over the scrubbed parent env |

To raise KwaFlux's own RPC budget, pass `env` (these names do not match DSH's scrub pattern):

```yaml
        env:
          KWAFLUX_MCP_CALL_TIMEOUT_MS: '30000'
          KWAFLUX_MCP_CONNECT_TIMEOUT_MS: '3000'
```

## After setup

Report the tool count (28 as of kwaflux-mcp 0.5.0) and hand the user the tool reference. When asked what the tools can and cannot do, note the verified gap: **KwaFlux cannot export GIF** (its containers are mp4/mkv/mov/avi/webm/mp3/flac/wav) and `enqueue_convert` takes neither a target format nor a time range — use the bundled `<KwaFlux>\ffmpeg.exe` for that instead.
