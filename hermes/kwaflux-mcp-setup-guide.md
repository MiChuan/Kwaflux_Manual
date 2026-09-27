# KwaFlux MCP 配置说明（Hermes）

如何在任意一台新电脑上，把 KwaFlux MCP（`kwaflux-mcp`）接入 Hermes Agent。

工具清单见同目录 `kwaflux-mcp-tools-reference.md`；实测结果与缺陷清单见同目录 `kwaflux-mcp-blackbox-test-report.md`；可复用的自动化流程见同目录 `kwaflux-mcp-setup/SKILL.md`。

> 本目录是从 Codex 版（`..\codex\`）改写而来：步骤、验证与踩坑对应，配置落点从 Codex 的 `config.toml` 换成 Hermes 的 `config.yaml`。Hermes 比 Codex 多一个前提：必须先装上 `mcp` 这个 Python extra，否则条目写对了工具也不会出现。

---

## 一、原理：配置写在哪里

Hermes 的 MCP 服务来自 `HERMES_HOME/config.yaml` 里的 `mcp_servers.<名字>`：

```yaml
mcp_servers:
  kwaflux:
    command: node
    args:
      - 'C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs'
```

| 平台 | 数据目录（`HERMES_HOME` 未设置时） | 配置文件 |
|---|---|---|
| Windows 原生 | `%LOCALAPPDATA%\hermes` | `%LOCALAPPDATA%\hermes\config.yaml` |
| macOS / Linux | `~/.hermes` | `~/.hermes/config.yaml` |
| 任意 | 环境变量 `HERMES_HOME` 优先 | `%HERMES_HOME%\config.yaml` |

要点：

- 这里注册的 MCP 服务属于当前 Hermes 数据目录。Windows 上数据目录**不是** `%USERPROFILE%\.hermes`：那个路径如果存在，通常只是源码 checkout（本机是 `%USERPROFILE%\.hermes\hermes-agent`）。配置写错目录，工具不会出现。
- 键名 `kwaflux` 决定工具前缀。Hermes 注册名是 `mcp__kwaflux__<工具短名>`，例如 `mcp__kwaflux__kwaflux_get_app_status`。不要把 Codex 的 `mcp__kwaflux::` 或 Cursor 的命名空间 `kwaflux` 拿来用。
- `kwaflux-mcp` 自带 `node_modules`（就在安装目录里），所以**不需要 npm install**，只要 `node` 在 PATH 上。
- Hermes 进程自己还要有 Python 包 `mcp`（extra 名也叫 `mcp`）。缺它时 `hermes mcp test` 报 `requires the 'mcp' Python SDK`。
- 不要写 `cwd`：服务用 `import.meta.url` 定位自己的模块。
- 不写 `tools.include` / `tools.exclude` 表示 28 个工具全部启用。`hermes mcp list` 的 Tools 列会显示 `all`。
- 交互式 CLI 与手写 YAML 等价，但 `hermes mcp add` 连上之后会弹出工具勾选界面，非交互 shell 里会卡住。代理或脚本请手写 YAML。

  ```powershell
  hermes mcp list
  hermes mcp test kwaflux
  hermes mcp add kwaflux --command node --args "C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs"
  hermes mcp remove kwaflux
  ```

  `add` / `remove` 都会再要一次确认或勾选。`--args` 后面的内容整段传给 MCP 进程，不要在它后面再加 Hermes 自己的旗标。

---

## 二、前置条件

| # | 条件 | 怎么确认 |
|---|---|---|
| 1 | 已安装 KwaFlux 应用 | 存在 `<KwaFlux 安装目录>\Tools\kwaflux-mcp\kwaflux_mcp.mjs` |
| 2 | MCP 入口脚本齐全 | 同目录应有 `agent_control.mjs`、`next_actions.mjs`、`node_modules\@modelcontextprotocol\`、`generated\` |
| 3 | Node 可用 | `node --version`（本机为 `v24.18.1`）。脚本要求 Node 18+ |
| 4 | `hermes` 在 PATH 上 | `Get-Command hermes`。Windows 安装器把 `%LOCALAPPDATA%\hermes\bin` 加进用户 PATH；已打开的终端要新开一个才看得到 |
| 5 | Hermes 已装 `mcp` Python extra | `hermes mcp test` 不再报 SDK 缺失。见步骤 4 |
| 6 | 知道在改哪个 `HERMES_HOME` | 见步骤 3 |

> **路径陷阱**：`kwaflux\_mcp.mjs` 这个路径**不存在**——真实入口是 `kwaflux-mcp\kwaflux_mcp.mjs`。写错的话表现为启动失败或工具不出现。

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

`command: node` 依赖 PATH。`node` 不在 PATH 上时，把 `command` 写成绝对路径，YAML 里用单引号，反斜杠不要转义：

```yaml
command: 'C:\Program Files\nodejs\node.exe'
```

### 步骤 3 — 确认改的是哪个 HERMES_HOME

```powershell
"HERMES_HOME = $env:HERMES_HOME"
"user env   = $([Environment]::GetEnvironmentVariable('HERMES_HOME','User'))"
"config     = $(if ($env:HERMES_HOME) { $env:HERMES_HOME } else { "$env:LOCALAPPDATA\hermes" })\config.yaml"
```

Windows 默认数据根是 `%LOCALAPPDATA%\hermes`，不是 `%USERPROFILE%\.hermes`。设置了用户级 `HERMES_HOME` 时以它为准。本机两者都是 `C:\Users\Admin1\AppData\Local\hermes`。

改完用户环境变量后，已经打开的 PowerShell **不会**自动更新。新开一个窗口，或在当前窗口执行：

```powershell
$env:HERMES_HOME = "$env:LOCALAPPDATA\hermes"
```

### 步骤 4 — 装上 MCP 的 Python 组件

标准安装脚本通常已经带上这个 extra。缺的时候 `hermes mcp test kwaflux` 会失败，原文是：

```
MCP server 'kwaflux' requires the 'mcp' Python SDK, but it is not installed.
```

在 **Hermes 源码目录**里用它自己的虚拟环境装（本机源码在用户目录，不在 `LOCALAPPDATA`）：

```powershell
$repo = "$env:USERPROFILE\.hermes\hermes-agent"
$py = Join-Path $repo ".venv\Scripts\python.exe"
Push-Location $repo
& $py -c "import pm; pm.sync_venv(['mcp'], explicit=True)"
Pop-Location
```

官方 Windows 安装器把源码放在 `%LOCALAPPDATA%\hermes\hermes-agent`。哪边有 `pyproject.toml` 和 `.venv`（或 `venv`），就在哪边执行。这是一次性的；装完不必再跑。

### 步骤 5 — 幂等检查

```powershell
hermes mcp list
```

已经有一行 `kwaflux` 就不要再追加第二个同名键。YAML 里重复的 `kwaflux:` 会让后写的覆盖先写的，或者让加载失败。要改路径，编辑现有那一段。

`hermes mcp add` 发现同名服务器时会问是否覆盖，非交互环境下不要用它做这件事。

### 步骤 6 — 备份并写入

先备份：

```powershell
$cfg = if ($env:HERMES_HOME) { "$env:HERMES_HOME\config.yaml" } else { "$env:LOCALAPPDATA\hermes\config.yaml" }
Copy-Item $cfg "$cfg.bak" -Force
```

在现有 `config.yaml` 末尾追加（文件里已有 `model:` 等键时，只加 `mcp_servers` 这一段，不要另起一份文件）：

```yaml
mcp_servers:
  kwaflux:
    command: node
    args:
      - 'C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs'
```

要点：

- YAML **单引号**里反斜杠是字面量。Windows 路径用单引号，不要写成 `"C:\\Program Files\\..."`，也不要写成未加引号的双反斜杠。
- 只改这一段。`config.yaml` 里还有模型、工具集、浏览器后端，写坏了影响面远超这一个 MCP。
- 不要加 `enabled: false`。省略 `enabled` 即启用。
- 不要加 `tools.include`，除非你故意只开放其中几个工具。

给人用的交互式等价命令（会弹出勾选，全选才等于上面的 YAML）：

```powershell
hermes mcp add kwaflux --command node --args "C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs"
```

### 步骤 7 — 生效

`hermes mcp list` 和 `hermes mcp test` 立刻读当前配置。

已经打开的 `hermes` 聊天**不会**自动加载新服务器。在那个会话里输入 `/reload-mcp`，或者退出后重新运行 `hermes`。

### 步骤 8 — 验证

**验证 A：注册信息正确**

```powershell
hermes mcp list
```

预期有一行：`kwaflux`、传输是 `node` 加入口脚本、Tools 为 `all`、Status 为 enabled。

**验证 B：Hermes 能连上并枚举工具**

```powershell
hermes mcp test kwaflux
```

预期：`Connected`，`Tools discovered: 28`。这一步失败且提到 Python SDK，回到步骤 4。

**验证 C：应用侧是否连上**

让模型调用 `mcp__kwaflux__kwaflux_get_app_status`，或对入口脚本做裸协议检查：

```powershell
$msgs = @(
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"1"}}}',
  '{"jsonrpc":"2.0","method":"notifications/initialized"}',
  '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"kwaflux_get_app_status","arguments":{}}}'
) -join "`n"
$msgs | node "C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs"
```

工作链路返回 `running`、`signed_in`、`entitlement_allowed`、`app_version`、`host_version`、`protocol_version`。应用 `1.0.5` 起还有 `agent_download_enabled`。再调 `kwaflux_list_tasks`，返回 `tasks` 数组（哪怕是空的）就说明整条读路径打通。

> 在**沙箱内**跑裸协议检查可能得到 `app_not_running: EPERM: operation not permitted, open '...\KwaFlux\agent\control.json'`。这是沙箱禁止读取控制文件，不是配置错误——换个非沙箱 shell，或直接用 `hermes mcp test`。

本机 2026-09-27 的实测值见附录 C。

---

## 四、故障排查

| 现象 | 原因 | 处理 |
|---|---|---|
| `hermes mcp test` 报 `requires the 'mcp' Python SDK` | 源码环境没装 `mcp` extra | 按步骤 4 执行 `pm.sync_venv(['mcp'], explicit=True)` |
| `hermes mcp list` 没有 `kwaflux` | 改的不是正在生效的 `HERMES_HOME` | 按步骤 3 确认 `config.yaml` 路径 |
| 聊天里看不到 `mcp__kwaflux__*` | 会话是写入配置之前开的 | `/reload-mcp` 或重新运行 `hermes` |
| 配置整体解析失败 | 重复的 `kwaflux:` 键，或 YAML 把 Windows 路径写成了双引号 | 用 `config.yaml.bak` 还原后只补一条，路径用单引号 |
| 子进程起不来 / `spawn ENOENT` | `node` 不在 PATH | `command` 改成 node 绝对路径 |
| 工具返回 `app_not_running: missing ...control.json` | KwaFlux 应用没在运行 | 启动 KwaFlux |
| 工具返回 `connect_timeout ...` | KwaFlux 卡在启动或达到实例上限 | 重启 KwaFlux；必要时调大 `KWAFLUX_MCP_CONNECT_TIMEOUT_MS` |
| 调用卡住然后超时 | 单次 RPC 超过服务默认 15 秒 | 调大 `KWAFLUX_MCP_CALL_TIMEOUT_MS`（见附录 A） |
| 写操作（convert / repair / enhance / download）被拒 | 未登录或 entitlement 不足 | 看 `signed_in` / `entitlement_allowed`，这是账号问题不是配置问题 |
| 下载被拒且 `agent_download_enabled: false` | 应用没打开代理下载 | KwaFlux 设置里打开「允许代理发起下载」 |
| 增强（`enhance_*`）任务一律失败 `-10505 / vendor:bo` | KwaFlux 本体缺陷（见黑盒报告） | 与配置无关 |
| 裸协议检查报 `EPERM ... control.json` | 沙箱禁止读控制文件 | 非沙箱执行，或用 `hermes mcp test` |
| `hermes` 命令找不到 | 用户 PATH 没有 `%LOCALAPPDATA%\hermes\bin`，或当前窗口是改 PATH 之前开的 | 新开终端；确认 `hermes.cmd` 在该 `bin` 目录 |

---

## 五、回滚

交互式：

```powershell
hermes mcp remove kwaflux
```

或恢复备份（非交互、可重复）：

```powershell
$cfg = if ($env:HERMES_HOME) { "$env:HERMES_HOME\config.yaml" } else { "$env:LOCALAPPDATA\hermes\config.yaml" }
Copy-Item "$cfg.bak" $cfg -Force
```

`config.yaml.bak` 不会被 Hermes 当作配置加载。恢复后，已打开的会话仍要 `/reload-mcp`。

---

## 六、跨平台 / 多机差异

换电脑时需要重新确认的只有这几处：

| 项 | Windows | macOS / Linux |
|---|---|---|
| MCP 入口脚本 | `C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs` | 应用包内 `Tools/kwaflux-mcp/kwaflux_mcp.mjs`（以实际安装位置为准） |
| 控制端点目录 | `%LOCALAPPDATA%\KwaFlux\agent\` | `~/Library/Application Support/KwaFlux/agent/` |
| Hermes 数据目录 | `%LOCALAPPDATA%\hermes` | `~/.hermes` |
| 配置文件 | `%LOCALAPPDATA%\hermes\config.yaml` | `~/.hermes/config.yaml` |
| 源码 / 虚拟环境 | `%LOCALAPPDATA%\hermes\hermes-agent`，或自行 clone 的目录 | 安装器放置的 checkout |

键名、`command: node`、单元素 `args` 都相同。macOS/Linux 的路径用正斜杠。设了 `HERMES_HOME` 时，数据目录以它为准，上表的默认值不再适用。

---

## 附录 A — 可选字段与调优

| 字段 | 默认 | 含义 |
|---|---|---|
| `command` | — | 可执行文件，用 `node` 或绝对路径 |
| `args` | — | 参数列表，这里只有入口脚本路径 |
| `env` | — | 传给子进程的环境变量 |
| `enabled` | 启用 | `false` 时 Hermes 完全跳过该服务器 |
| `timeout` | Hermes 默认 | 单次工具调用超时（秒） |
| `connect_timeout` | Hermes 默认 | 首次连接超时（秒） |
| `tools.include` / `tools.exclude` | 全部工具 | 按短名过滤。写了 `include` 就只有列出的工具可用 |

给 kwaflux 放宽它自己的 RPC 预算（这是子进程环境变量，不是 Hermes 的 `timeout`）：

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

> Codex 的 `startup_timeout_sec`、DSH 的 `failOnStartupError` 和 `reconnect.*` 在 Hermes 里没有同名键。连接失败的表现是 `hermes mcp test` 报错，或聊天里根本没有 `mcp__kwaflux__*` 工具。

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

## 附录 C — 本机实测记录（Hermes）

| 项 | 值 |
|---|---|
| 配置日期 | 2026-09-27 |
| Hermes | `vgit.28e6496.dirty`（2026.9.24），Python 3.14.6 |
| 配置文件 | `C:\Users\Admin1\AppData\Local\hermes\config.yaml` 的 `mcp_servers.kwaflux` |
| `HERMES_HOME` | 用户环境变量 `C:\Users\Admin1\AppData\Local\hermes` |
| 源码目录 | `C:\Users\Admin1\.hermes\hermes-agent`（git checkout，不是安装器默认的 `%LOCALAPPDATA%\hermes\hermes-agent`） |
| 写入方式 | 手写 YAML。没有用 `hermes mcp add`，避免交互式勾选 |
| 入口脚本 | `C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs` |
| Node | `C:\Program Files\nodejs\node.exe` v24.18.1 |
| MCP extra | 首次测试缺少 SDK；`pm.sync_venv(['mcp'], explicit=True)` 之后 `hermes mcp test kwaflux` 为 Connected 1226 ms、28 个工具 |
| 协议校验 | 裸 stdio 握手返回 `kwaflux-mcp 0.5.0`；`kwaflux_get_app_status` → `running: true`、`signed_in: true`、`entitlement_allowed: true`、`agent_download_enabled: false`、`app_version: 1.0.5`；`kwaflux_list_tasks(limit=3)` 返回 `{"tasks":[]}` |
| 生效方式 | `hermes mcp test` 立即可见；已打开的聊天需要 `/reload-mcp` 或重启 `hermes` |

---

## 附录 D — 把这份 skill 装到 Hermes

`kwaflux-mcp-setup/SKILL.md` 是一个 Hermes skill。Hermes 从当前数据目录的 `skills` 扫描：

```
%HERMES_HOME%\skills\<skill-name>\SKILL.md
```

Windows 上未设置 `HERMES_HOME` 时，这就是 `%LOCALAPPDATA%\hermes\skills\`。

安装（用户级，跨项目可用）：

```powershell
$home = if ($env:HERMES_HOME) { $env:HERMES_HOME } else { "$env:LOCALAPPDATA\hermes" }
$dest = Join-Path $home "skills\kwaflux-mcp-setup"
New-Item -ItemType Directory -Force (Split-Path $dest) | Out-Null
Copy-Item "D:\Project\GitHub\Kwaflux_Manual\hermes\kwaflux-mcp-setup" $dest -Recurse -Force
```

结果应是：

```
<HERMES_HOME>\skills\kwaflux-mcp-setup\SKILL.md
```

规则要点：

- frontmatter 必须提供 `name`（kebab-case）和 `description`；`description` 决定模型什么时候自动调用这个 skill。
- 本 skill 保持默认可自动调用：在新电脑上直接说「帮我在新电脑上配置 kwaflux MCP」即可命中。
- 改 SKILL.md 正文后，新会话会读到当前文件。已经打开的 Hermes 会话需要重新加载 skills。
