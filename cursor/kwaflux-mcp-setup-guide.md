# KwaFlux MCP 配置说明（Cursor）

如何在任意一台新电脑上，把 KwaFlux MCP（`kwaflux-mcp`）接入 Cursor。

工具清单见同目录 `kwaflux-mcp-tools-reference.md`；实测结果与缺陷清单见同目录 `kwaflux-mcp-blackbox-test-report.md`；可复用的自动化流程见同目录 `kwaflux-mcp-setup/SKILL.md`。

> 本目录是从 Codex 版（`..\codex\`）和 DeepSeek Harness 版（`..\deepseek-harness\`）改写而来：步骤、验证与踩坑完全对应，只是把配置落点换成 Cursor 的 `mcp.json`。

---

## 一、原理：配置写在哪里

Cursor 的 MCP 服务来自 `mcp.json` 里的 `mcpServers.<名字>` 对象：

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

| 平台 | 全局配置 | 项目配置 |
|---|---|---|
| Windows 默认 | `C:\Users\<你>\.cursor\mcp.json` | `<工作区>\.cursor\mcp.json` |
| macOS / Linux 默认 | `~/.cursor/mcp.json` | `<工作区>/.cursor/mcp.json` |

要点：

- **全局**条目对这台机器上每个项目可见；**项目**条目只对当前工作区可见。KwaFlux 是本机应用，默认写全局。
- 对象名 `kwaflux` 是 `mcp.json` 里的键，不是 Agent 调用时的命名空间。写在用户级 `~\.cursor\mcp.json` 时，Agent 看到的命名空间是 `user-kwaflux`，工具短名仍是 `kwaflux_get_app_status`。本机当前会话就是这个名字。项目级 `.cursor/mcp.json` 的命名空间以 `GetDynamicTools` 实际列出的为准，不要猜。DSH / Hermes 的 `mcp__kwaflux__*` 和 Codex 的 `mcp__kwaflux::*` **不要**拿到 Cursor 里用。
- `kwaflux-mcp` 自带 `node_modules`（就在安装目录里），所以**不需要 npm install**，只要 `node` 在 PATH 上。
- 不要写 `cwd`：服务用 `import.meta.url` 定位自己的模块。
- 这是 **stdio 本地服务**。Cloud Agent 跑在远端，拉不起这台机器上的 `node` + KwaFlux，配了也不会通。
- 不想手写 JSON 时用 CLI，两者等价：

  ```powershell
  cursor --add-mcp '{"name":"kwaflux","command":"node","args":["C:\\Program Files\\Kwaflux\\Tools\\kwaflux-mcp\\kwaflux_mcp.mjs"]}'
  ```

  只要项目级配置时加上 `--mcp-workspace`。装了 Cursor Agent CLI 之后还可以：

  ```powershell
  agent mcp list
  agent mcp list-tools kwaflux
  agent mcp enable kwaflux
  agent mcp disable kwaflux
  ```

---

## 二、前置条件

| # | 条件 | 怎么确认 |
|---|---|---|
| 1 | 已安装 KwaFlux 应用 | 存在 `<KwaFlux 安装目录>\Tools\kwaflux-mcp\kwaflux_mcp.mjs` |
| 2 | MCP 入口脚本齐全 | 同目录应有 `agent_control.mjs`、`next_actions.mjs`、`node_modules\@modelcontextprotocol\`、`generated\enhance_catalog.mjs` |
| 3 | Node 可用 | `node --version`（本机为 `v24.18.1`）。脚本要求 Node 18+，并从同目录 `node_modules` 加载 `@modelcontextprotocol/sdk` 和 `zod`，不需要再 `npm install` |
| 4 | 知道在改全局还是项目 `mcp.json` | 见步骤 3 |

> **路径陷阱**：`kwaflux\_mcp.mjs` 这个路径**不存在**——真实入口是 `kwaflux-mcp\kwaflux_mcp.mjs`（没有子目录，文件名没有下划线前缀）。写错的话表现为启动失败或工具不出现。
>
> **JSON 陷阱**：Windows 路径在 JSON 里必须写成 `C:\\Program Files\\...`（两个反斜杠），或改用正斜杠 `C:/Program Files/...`。只写一个 `\` 会让整个 `mcp.json` 解析失败，**该文件里所有 MCP 一起消失**。

---

## 三、步骤

### 步骤 1 — 定位入口脚本

```powershell
Get-ChildItem "C:\Program Files\Kwaflux\Tools\kwaflux-mcp" -Force |
  Select-Object Mode, Length, Name
```

默认安装路径是 `C:\Program Files\Kwaflux`，**但安装时可以改**。以实际存在的路径为准，后面写进 `args`。

### 步骤 2 — 确认 Node

```powershell
(Get-Command node).Source
node --version
```

`"command": "node"` 依赖 PATH；想锁定版本或 `node` 不在 PATH 上时，直接写绝对路径：

```json
"command": "C:\\Program Files\\nodejs\\node.exe"
```

### 步骤 3 — 确认改的是哪一份 mcp.json

```powershell
"global  = $env:USERPROFILE\.cursor\mcp.json"
"exists  = $(Test-Path "$env:USERPROFILE\.cursor\mcp.json")"
"project = $(Join-Path (Get-Location) '.cursor\mcp.json')"
```

> **常见坑**：Cursor 同时认全局和项目两份配置。只写项目文件时，换一个工作区就会"看不到工具"。本机当前**还没有** `%USERPROFILE%\.cursor\mcp.json`——第一次配置就是在创建这个文件。

### 步骤 4 — 幂等检查

```powershell
$mcp = "$env:USERPROFILE\.cursor\mcp.json"
if (Test-Path $mcp) { Get-Content $mcp -Raw } else { "no global mcp.json yet" }
```

已经有 `mcpServers.kwaflux` 就别再塞第二个同名键。`cursor --add-mcp` 覆盖同名条目没问题；手写两个 `kwaflux` 键是未定义行为。

### 步骤 5 — 备份并写入

CLI 方式（推荐；写入用户配置，不用自己拼 JSON）：

```powershell
cursor --add-mcp '{"name":"kwaflux","command":"node","args":["C:\\Program Files\\Kwaflux\\Tools\\kwaflux-mcp\\kwaflux_mcp.mjs"]}'
```

手改方式，先备份（文件不存在就跳过）：

```powershell
$mcp = "$env:USERPROFILE\.cursor\mcp.json"
if (Test-Path $mcp) { Copy-Item $mcp "$mcp.bak" -Force }
```

然后写成（或合并进已有 `mcpServers`）：

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

要点：

- JSON **不允许注释、不允许尾逗号**。写坏了影响的是这份文件里全部 MCP，不只 kwaflux。
- 不要写 `"type": "stdio"`。本机正在使用的用户级 `mcp.json` 只有 `command` 和 `args`，没有 `type`。
- `command` / `args` / `env` 支持 `${env:NAME}`、`${userHome}`、`${workspaceFolder}`、`${pathSeparator}`。本服务用绝对安装路径即可，不必插值。
- 只改 `kwaflux` 这一段，别动文件其它服务器。

### 步骤 6 — 生效

改完 `mcp.json` 后：

1. 打开侧栏 **Customize → MCP**，确认 `kwaflux` 出现且为开启。
2. 若仍是 disconnected，点一次开关，或重启 Cursor。
3. 排障看 **Output 面板 → MCP Logs**。

官方说明：自定义本地服务器更新文件后通常需要重启 Cursor。

### 步骤 7 — 验证

**验证 A：配置文件正确**

```powershell
Get-Content "$env:USERPROFILE\.cursor\mcp.json" -Raw | ConvertFrom-Json |
  Select-Object -ExpandProperty mcpServers |
  Select-Object -ExpandProperty kwaflux
```

预期能看到 `command = node`、`args` 指向 `kwaflux_mcp.mjs`。若已安装 `agent` CLI：

```powershell
agent mcp list
agent mcp list-tools kwaflux
```

**验证 B：应用侧是否连上**

让模型调用 `kwaflux_get_app_status`（用户级配置的命名空间是 `user-kwaflux`），预期字段包括：

```json
{
  "running": true,
  "platform": "win",
  "app_version": "1.0.4",
  "host_version": "uilayer-host",
  "protocol_version": 1,
  "signed_in": true,
  "entitlement_allowed": true
}
```

再调用 `kwaflux_list_tasks`，能返回任务数组（哪怕是空的）就说明整条链路打通。

**验证 C：不走模型的裸协议检查**（怀疑工具没挂上时用）

```powershell
$msgs = @(
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"1"}}}',
  '{"jsonrpc":"2.0","method":"notifications/initialized"}',
  '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"kwaflux_get_app_status","arguments":{}}}'
) -join "`n"
$msgs | node "C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs"
```

返回 `"running": true` 即服务本身没问题。stdio 服务会继续等下一行 JSON-RPC，查完后停掉这个 `node` 进程。

> 在**沙箱内**跑这条可能得到 `app_not_running: EPERM: operation not permitted, open '...\KwaFlux\agent\control.json'`。这是沙箱禁止读取控制文件，不是配置错误——换个非沙箱 shell，或直接用验证 B。

---

## 四、故障排查

| 现象 | 原因 | 处理 |
|---|---|---|
| 模型看不到 `kwaflux` 工具 | 改的不是正在生效的那份 `mcp.json` | 按步骤 3 确认路径，再用验证 A 复查 |
| 同上 | Cursor 没重新加载配置 | Customize 里开关一次，或重启 Cursor |
| 所有 MCP 一起消失 | JSON 语法错误（未转义的 `\`、尾逗号、注释） | 用 `mcp.json.bak` 还原后只补一条 |
| 子进程起不来 / `spawn ENOENT` | `node` 不在 PATH | `command` 改成 node 绝对路径 |
| 工具返回 `app_not_running: missing ...control.json` | KwaFlux 应用没在运行 | 启动 KwaFlux |
| 工具返回 `connect_timeout ...` | KwaFlux 卡在启动或达到实例上限 | 重启 KwaFlux；必要时调大 `KWAFLUX_MCP_CONNECT_TIMEOUT_MS` |
| 调用卡住然后 `call_timeout after 15000ms` | 单次 RPC 超过 15 秒 | 调大 `KWAFLUX_MCP_CALL_TIMEOUT_MS`（见附录 A） |
| 写操作（convert / repair / enhance / download）被拒 | 未登录或 entitlement 不足 | `get_app_status` 看 `signed_in` / `entitlement_allowed`，这是账号问题不是配置问题 |
| 增强（`enhance_*`）任务一律失败 `-10505 / vendor:bo` | KwaFlux 本体缺陷（见黑盒报告） | 与配置无关；改用其它链路或等应用修复 |
| 裸协议检查报 `EPERM ... control.json` | 沙箱禁止读控制文件 | 非沙箱执行，或用验证 B |
| Cloud Agent 里没有工具 / 调用失败 | 本地 stdio 到不了这台机器的 KwaFlux | 只能在本机 Cursor 用；不要改成 HTTP URL 硬凑 |

---

## 五、回滚

删掉 `mcp.json` 里的 `kwaflux` 键，或恢复备份：

```powershell
Copy-Item "$env:USERPROFILE\.cursor\mcp.json.bak" "$env:USERPROFILE\.cursor\mcp.json" -Force
```

然后在 Customize 里关掉该服务器（或重启 Cursor）。`mcp.json.bak` 不会被加载，留着无副作用。

---

## 六、跨平台 / 多机差异

换电脑时需要重新确认的只有三处：

| 项 | Windows | macOS / Linux |
|---|---|---|
| MCP 入口脚本 | `C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs` | `/Applications/KwaFlux.app/.../Tools/kwaflux-mcp/kwaflux_mcp.mjs`（以实际安装位置为准） |
| 控制端点目录 | `%LOCALAPPDATA%\KwaFlux\agent\` | `~/Library/Application Support/KwaFlux/agent/` |
| 全局配置文件 | `%USERPROFILE%\.cursor\mcp.json` | `~/.cursor/mcp.json` |

其余（对象名、`command`、参数形状）都相同。macOS/Linux 下 `args` 用正斜杠；Windows 用 `\\` 或正斜杠都可以。不要为了“补全”而加上 `type`。

---

## 附录 A — 可选字段与调优

| 字段 | 默认 | 含义 |
|---|---|---|
| `type` | 省略 | 不是必需字段。本机可用配置没有它 |
| `command` | — | 可执行文件，用 `node` 或绝对路径 |
| `args` | — | 参数数组，这里就是入口脚本路径 |
| `env` | — | 追加给子进程的环境变量 |
| `envFile` | — | 再读一个 env 文件（仅 stdio） |

给 kwaflux 放宽超时的写法：

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

> DSH 那版里还有 `failOnStartupError`、`reconnect.*`、`toolCallTimeoutMs`；Codex 有 `startup_timeout_sec`。这些都是**对端客户端专属**的，Cursor 没有对应键。连接失败在 Cursor 里的表现就是 Customize 里 disconnected，或工具不出现。看 **Output → MCP Logs**。

---

## 附录 B — 用 KwaFlux 自带 ffmpeg 补足 GIF 能力

KwaFlux 的导出容器**不含 GIF**，MCP 的 `enqueue_convert` 也无目标格式与裁剪参数。需要 GIF 时用应用自带的 ffmpeg：

```powershell
$ff  = "C:\Program Files\Kwaflux\ffmpeg.exe"
$src = "C:\path\source.mp4"
$out = "C:\path\source.gif"

& $ff -y -i $src -an -vf "fps=30,scale=1280:720:flags=lanczos,format=rgb24,split[s0][s1];[s0]palettegen=max_colors=256[p];[s1][p]paletteuse=dither=bayer:bayer_scale=5" -loop 0 $out
```

- `format=rgb24` 必须加，否则 `palettegen` 会报 `input frame is not in sRGB, colors may be off`。
- 只看前 N 秒：在 `-i` **之前**加 `-t N`。
- `dither=bayer:bayer_scale=5` 比 `dither=sierra2_4a` 体积小得多、实测 SSIM 更高；若渐变区域出现色带就换成 `sierra2_4a:diff_mode=rectangle`（体积约大 40%）。
- 想要更小体积可降 `scale` 尺寸或 `fps`；GIF 是 256 色格式，4 秒 720p 通常就是几十 MB，无法靠参数压到平台上传限制内——真有体积要求应改用 WebP。

---

## 附录 C — 本机实测记录（Cursor）

| 项 | 值 |
|---|---|
| 配置日期 | 2026-09-24 做协议复验；2026-09-27 核对已写入的用户配置 |
| 全局配置 | `C:\Users\Admin1\.cursor\mcp.json` 已有 `mcpServers.kwaflux`：`command` 为 `node`，`args` 为入口脚本。没有 `type`。`env` 是空对象 |
| Agent 命名空间 | `user-kwaflux`（用户级 `mcp.json` 的键 `kwaflux` 会被加上 `user-` 前缀） |
| 入口脚本 | `C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs`（目录含 `agent_control.mjs`、`next_actions.mjs`、`generated\`、`node_modules\`） |
| Node | `C:\Program Files\nodejs\node.exe` v24.18.1 |
| CLI | Cursor `3.21.9` 支持 `cursor --add-mcp '<json>'`；本机 PATH 上没有独立的 `agent` 命令 |
| 协议校验 | 2026-09-24 裸 stdio：`kwaflux-mcp 0.5.0`，`tools/list` 28 个工具，`app_version: 1.0.4`。2026-09-27 同一服务对 Hermes 复验时应用为 `1.0.5` |
| 已知坑 | JSON 反斜杠必须转义；改完需 Customize 开关或重启；Cloud Agent 用不了这套 stdio；沙箱内读 `control.json` 会 `EPERM` |

---

## 附录 D — 把这份 skill 装到 Cursor

`kwaflux-mcp-setup/SKILL.md` 是一个 Cursor skill。

| 范围 | 路径 |
|---|---|
| 项目 | `<项目根>/.cursor/skills/kwaflux-mcp-setup/SKILL.md` |
| 用户 | 用户 Agent Store 里的 `skills/kwaflux-mcp-setup/SKILL.md`。当前会话列出的用户 store 是 `C:\Users\Admin1\AppData\Local\Cursor\AgentStores\cursor_agent_stores\u432533545\files` |

用户级 skill **不要**装到 `~/.cursor/skills/`，也不要装到 `~/.cursor/skills-cursor/`（后者是 Cursor 内置目录）。只有当前会话没有挂上用户 store 时，才退回 `~/.cursor/skills/`，并且那份只留在这台机器上。

只给这个仓库用时：

```powershell
$src = "D:\Project\GitHub\Kwaflux_Manual\cursor\kwaflux-mcp-setup"
$dest = "D:\Project\GitHub\Kwaflux_Manual\.cursor\skills\kwaflux-mcp-setup"
New-Item -ItemType Directory -Force (Split-Path $dest) | Out-Null
Copy-Item $src $dest -Recurse -Force
```

装到用户 store：

```powershell
$src = "D:\Project\GitHub\Kwaflux_Manual\cursor\kwaflux-mcp-setup"
$dest = "C:\Users\Admin1\AppData\Local\Cursor\AgentStores\cursor_agent_stores\u432533545\files\skills\kwaflux-mcp-setup"
New-Item -ItemType Directory -Force (Split-Path $dest) | Out-Null
Copy-Item $src $dest -Recurse -Force
```

规则要点：

- frontmatter 必须提供 `name`（kebab-case，且与文件夹名一致）和 `description`。`description` 决定模型什么时候自动调用。不要写 `disabled-environments`，当前 skill 规范没有这个字段。
- 本 skill **没有** `disable-model-invocation`，所以本机 Agent 可以自动调用。若希望只手动调用，加一行 `disable-model-invocation: true`。
- Cloud Agent 跑在远端，拉不起这台机器上的 `node` 和 KwaFlux。这是运行位置的限制，不是 frontmatter 能关掉的。
- 可选资源按需放：`references/`、`scripts/`、`assets/`。本 skill 目前不需要这些。
- 改 SKILL.md 正文不需要重启 Cursor，下次调用即读取当前文件。
