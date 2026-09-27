# KwaFlux MCP 配置说明（Codex）

如何在任意一台新电脑上，把 KwaFlux MCP（`kwaflux-mcp`）接入 Codex。

工具清单见同目录 `kwaflux-mcp-tools-reference.md`；实测结果与缺陷清单见同目录 `kwaflux-mcp-blackbox-test-report.md`；可复用的自动化流程见同目录 `kwaflux-mcp-setup/SKILL.md`。

> 本目录是从 DeepSeek Harness 版（`..\deepseek-harness\`）改写而来：步骤、验证与踩坑完全对应，只是把配置落点从 DSH 的 `cordis.patch.yml` 换成 Codex 的 `config.toml`。

---

## 一、原理：配置写在哪里

Codex 的 MCP 服务来自 `$CODEX_HOME/config.toml` 里的 `[mcp_servers.<名字>]` 表：

```toml
[mcp_servers.kwaflux]
command = "node"
args = ['C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs']
```

| 平台 | 配置文件 |
|---|---|
| Windows 默认 | `C:\Users\<你>\.codex\config.toml` |
| macOS / Linux 默认 | `~/.codex/config.toml` |
| 任意 | `%CODEX_HOME%\config.toml`（环境变量优先） |

要点：

- 这里注册的 MCP 服务是**全局**的——所有项目、所有任务都能看到，不存在"每个项目一份"的配置层。
- 表名 `kwaflux` 就是工具命名空间：Codex 里工具全名形如 `mcp__kwaflux::kwaflux_get_app_status`（本机实测显示格式）。改名即改前缀；DSH 的 `serverName` 字段在 Codex 里 **不存在**。
- `kwaflux-mcp` 自带 `node_modules`（就在安装目录里），所以**不需要 npm install**，只要 `node` 在 PATH 上。
- 不要写 `cwd`：服务用 `import.meta.url` 定位自己的模块。
- 不想手写 TOML 时用 CLI，两者等价：

  ```powershell
  codex mcp add kwaflux -- node "C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs"
  codex mcp get kwaflux
  codex mcp list
  codex mcp remove kwaflux
  ```

---

## 二、前置条件

| # | 条件 | 怎么确认 |
|---|---|---|
| 1 | 已安装 KwaFlux 应用 | 存在 `<KwaFlux 安装目录>\Tools\kwaflux-mcp\kwaflux_mcp.mjs` |
| 2 | MCP 入口脚本齐全 | 同目录应有 `agent_control.mjs`、`next_actions.mjs`、`node_modules\@modelcontextprotocol\`、`generated\enhance_catalog.mjs` |
| 3 | Node 可用 | `node --version`（本机为 `v24.18.1`）。脚本要求 Node 18+，并从同目录 `node_modules` 加载 `@modelcontextprotocol/sdk` 和 `zod`，不需要再 `npm install` |
| 4 | 知道在改哪个 `CODEX_HOME` | 见步骤 3 |

> **路径陷阱**：`kwaflux\_mcp.mjs` 这个路径**不存在**——真实入口是 `kwaflux-mcp\kwaflux_mcp.mjs`（没有子目录，文件名没有下划线前缀）。写错的话表现为启动失败或工具不出现。

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

`command = "node"` 依赖 PATH；想锁定版本或 `node` 不在 PATH 上时，直接写绝对路径：

```toml
command = 'C:\Program Files\nodejs\node.exe'
```

### 步骤 3 — 确认改的是哪个 CODEX_HOME

```powershell
"CODEX_HOME = $env:CODEX_HOME"
"config     = $(if ($env:CODEX_HOME) { $env:CODEX_HOME } else { "$env:USERPROFILE\.codex" })\config.toml"
```

> **常见坑（本机实测踩到）**：在沙箱/服务式 shell 里 `HOME`、`USERPROFILE` 可能为空，此时 `codex mcp ...` 会直接失败：
>
> ```
> Error: failed to resolve CODEX_HOME
> Caused by: Could not find home directory
> ```
>
> 解决办法是把 `CODEX_HOME` 显式传给这一条命令：
>
> ```powershell
> $env:CODEX_HOME = 'C:\Users\Admin1\.codex'; codex mcp add kwaflux -- node "<入口路径>"
> ```
>
> 不显式指定，你可能会改到别的配置文件，然后在真正运行 Codex 的那份配置里"看不到工具"。

### 步骤 4 — 幂等检查

```powershell
$env:CODEX_HOME = 'C:\Users\Admin1\.codex'; codex mcp get kwaflux
```

已经有输出就别加第二条：`codex mcp add` 覆盖同名条目没问题，但手写两个 `[mcp_servers.kwaflux]` 表是 TOML 重复键，会让整个配置文件解析失败。

### 步骤 5 — 备份并写入

CLI 方式（推荐，不用备份，工具会重写该表）：

```powershell
$env:CODEX_HOME = 'C:\Users\Admin1\.codex'
codex mcp add kwaflux -- node "C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs"
```

手改方式，先备份：

```powershell
Copy-Item "$env:CODEX_HOME\config.toml" "$env:CODEX_HOME\config.toml.bak" -Force
```

然后追加（`codex mcp add` 生成的就是这两行，可直接粘）：

```toml
[mcp_servers.kwaflux]
command = "node"
args = ['C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs']
```

要点：

- TOML 单引号内**反斜杠是字面量**，Windows 路径直接写，不用转义成 `\\`。
- `startup_timeout_sec = 20` 是可选的。这个服务冷启动远小于 1 秒，默认值就够，写上也不亏。
- 只改这一段，别动文件其它部分——`config.toml` 还承载模型、沙箱、项目信任等设置，写坏了影响面远超这一个 MCP。

### 步骤 6 — 生效

Codex 不需要为 MCP 条目重启：本机实测**写入配置后同一会话内**就出现了 `mcp__kwaflux` 命名空间（28 个工具）。

若新任务里仍看不到工具，重启 Codex 应用即可。

### 步骤 7 — 验证

**验证 A：注册信息正确**

```powershell
$env:CODEX_HOME = 'C:\Users\Admin1\.codex'; codex mcp get kwaflux
```

预期：

```
kwaflux
  enabled: true
  transport: stdio
  command: node
  args: C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs
  remove: codex mcp remove kwaflux
```

**验证 B：应用侧是否连上**

让模型调用 `kwaflux_get_app_status`，预期：

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

返回 `"running": true` 即服务本身没问题。

> 在**沙箱内**跑这条可能得到 `app_not_running: EPERM: operation not permitted, open '...\KwaFlux\agent\control.json'`。这是沙箱禁止读取控制文件，不是配置错误——换个非沙箱 shell，或直接用验证 B。

---

## 四、故障排查

| 现象 | 原因 | 处理 |
|---|---|---|
| 模型看不到 `mcp__kwaflux` 工具 | 改的不是正在生效的 `CODEX_HOME` | 按步骤 3 确认路径，再用验证 A 复查 |
| 同上 | Codex 没重新加载配置 | 重启 Codex 应用 |
| `failed to resolve CODEX_HOME` / `Could not find home directory` | 当前 shell 没有 `HOME`/`USERPROFILE` | 显式 `$env:CODEX_HOME = '...'` |
| 配置整体解析失败 | 重复的 `[mcp_servers.kwaflux]` 表，或 TOML 语法错误 | 用 `config.toml.bak` 还原后只补一条 |
| 子进程起不来 / `spawn ENOENT` | `node` 不在 PATH | `command` 改成 node 绝对路径 |
| 工具返回 `app_not_running: missing ...control.json` | KwaFlux 应用没在运行 | 启动 KwaFlux |
| 工具返回 `connect_timeout ...` | KwaFlux 卡在启动或达到实例上限 | 重启 KwaFlux；必要时调大 `KWAFLUX_MCP_CONNECT_TIMEOUT_MS` |
| 调用卡住然后 `call_timeout after 15000ms` | 单次 RPC 超过 15 秒 | 调大 `KWAFLUX_MCP_CALL_TIMEOUT_MS`（见附录 A） |
| 写操作（convert / repair / enhance / download）被拒 | 未登录或 entitlement 不足 | `get_app_status` 看 `signed_in` / `entitlement_allowed`，这是账号问题不是配置问题 |
| 增强（`enhance_*`）任务一律失败 `-10505 / vendor:bo` | KwaFlux 本体缺陷（见黑盒报告） | 与配置无关；改用其它链路或等应用修复 |
| 裸协议检查报 `EPERM ... control.json` | 沙箱禁止读控制文件 | 非沙箱执行，或用验证 B |

---

## 五、回滚

```powershell
codex mcp remove kwaflux
```

或恢复备份：

```powershell
Copy-Item "$env:CODEX_HOME\config.toml.bak" "$env:CODEX_HOME\config.toml" -Force
```

`config.toml.bak` 放在 `.codex` 目录里不会被当作配置文件加载，留着无副作用。

---

## 六、跨平台 / 多机差异

换电脑时需要重新确认的只有三处：

| 项 | Windows | macOS / Linux |
|---|---|---|
| MCP 入口脚本 | `C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs` | `/Applications/KwaFlux.app/.../Tools/kwaflux-mcp/kwaflux_mcp.mjs`（以实际安装位置为准） |
| 控制端点目录 | `%LOCALAPPDATA%\KwaFlux\agent\` | `~/Library/Application Support/KwaFlux/agent/` |
| 配置文件 | `%USERPROFILE%\.codex\config.toml` | `~/.codex/config.toml` |

其余（表名、`command`、参数形状）都相同。macOS/Linux 下 `args` 里的路径用正斜杠更稳；TOML 单引号字符串本身可以含空格，不用额外加引号。

---

## 附录 A — 可选字段与调优

| 字段 | 默认 | 含义 |
|---|---|---|
| `command` | — | 可执行文件，用 `node` 或绝对路径 |
| `args` | — | 参数数组，这里就是入口脚本路径 |
| `env` | — | 追加给子进程的环境变量（`codex mcp add` 用 `--env KEY=VALUE`） |
| `startup_timeout_sec` | Codex 默认 | 首次握手超时上限；本服务几乎瞬起 |

给 kwaflux 放宽超时的写法：

```toml
[mcp_servers.kwaflux]
command = "node"
args = ['C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs']
startup_timeout_sec = 20

[mcp_servers.kwaflux.env]
KWAFLUX_MCP_CALL_TIMEOUT_MS = "30000"
KWAFLUX_MCP_CONNECT_TIMEOUT_MS = "3000"
```

> DSH 那版里还有 `failOnStartupError`、`reconnect.*`、`toolCallTimeoutMs` 这些字段，它们是 **dsh-mcp-client 专属**的，Codex 没有对应键。连接失败在 Codex 里的表现就是"工具不出现"。

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

## 附录 C — 本机实测记录（Codex）

| 项 | 值 |
|---|---|
| 配置日期 | 2026-09-24 |
| 配置文件 | `C:\Users\Admin1\.codex\config.toml` 的 `[mcp_servers.kwaflux]` |
| 写入方式 | `codex mcp add kwaflux -- node "..."` → `Added global MCP server 'kwaflux'.` |
| 入口脚本 | `C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs` |
| Node | `C:\Program Files\nodejs\node.exe` v24.18.1 |
| 生效方式 | 无需重启；同一会话内即出现 `mcp__kwaflux` 命名空间（28 个工具） |
| 注册校验 | `codex mcp get kwaflux` → `enabled: true`、`transport: stdio`、参数正确 |
| 协议校验 | 裸 stdio 握手返回 `kwaflux-mcp 0.5.0`；`kwaflux_get_app_status` → `running: true`、`signed_in: true`、`entitlement_allowed: true`；`kwaflux_list_tasks(limit=3)` 正常返回 |
| 已知坑 | 沙箱 shell 里 `codex mcp` 需要显式 `CODEX_HOME`；沙箱内读 `control.json` 会 `EPERM` |

---

## 附录 D — 把这份 skill 装到 Codex

`kwaflux-mcp-setup/SKILL.md` 是一个 Codex skill。Codex 从**用户级 skills 目录**扫描：

```
%CODEX_HOME%\skills\<skill-name>\SKILL.md
```

本机现有 skills 就放在 `C:\Users\Admin1\.codex\skills\`（系统自带的在 `.system\` 下）。

安装（用户级，跨项目可用）：

```powershell
$skills = if ($env:CODEX_HOME) { "$env:CODEX_HOME\skills" } else { "$env:USERPROFILE\.codex\skills" }
New-Item -ItemType Directory -Force $skills | Out-Null
Copy-Item "D:\Project\GitHub\Kwaflux_Manual\codex\kwaflux-mcp-setup" $skills -Recurse -Force
```

结果应是：

```
<CODEX_HOME>\skills\kwaflux-mcp-setup\SKILL.md
```

装完可以用 skill-creator 自带的校验脚本复查（路径随 Codex 版本变化，本机为 `C:\Users\Admin1\.codex\skills\.system\skill-creator\scripts\quick_validate.py`）。两个前置条件，缺一个就报错：

```powershell
$py = "$env:USERPROFILE\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe"
& $py -m pip install pyyaml        # ① 脚本要 import yaml，运行时默认没装
$env:PYTHONUTF8 = '1'              # ② 中文 Windows 下不设会 UnicodeDecodeError（脚本 read_text() 默认走 GBK）
& $py '<skill-creator 目录>\scripts\quick_validate.py' "$env:CODEX_HOME\skills\kwaflux-mcp-setup"
```

通过时输出 `Skill is valid!`。

规则要点：

- frontmatter 必须提供 `name`（kebab-case）和 `description`；`description` 决定模型**什么时候自动调用**这个 skill。
- 可选资源按需放：`references/`（按需加载的文档）、`scripts/`（可执行脚本）、`assets/`（产出用素材）、`agents/openai.yaml`（界面元数据与调用策略）。本 skill 目前不需要这些。
- 想让 skill **只手动调用**，在 `agents/openai.yaml` 里写 `policy: {allow_implicit_invocation: false}`，之后用 `$skill-name` 显式触发。本 skill 保持默认可自动调用：在新电脑上直接说"帮我在新电脑上配置 kwaflux MCP"即可命中。
- 改 SKILL.md 正文不需要重启 Codex，下次调用即读取当前文件。
