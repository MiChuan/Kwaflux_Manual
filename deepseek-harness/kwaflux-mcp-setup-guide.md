# KwaFlux MCP 配置说明

如何在任意一台新电脑上，把 KwaFlux MCP（`kwaflux-mcp`）接入 DeepSeek Harness（DSH）。

工具清单见同目录 `kwaflux-mcp-tools-reference.md`；可复用的自动化流程见同目录 `kwaflux-mcp-setup/SKILL.md`。

---

## 一、原理：配置写在哪里

DSH 的插件树由**分层补丁**组成，每个 profile 的最终配置是：

```
package.json 里 dsh.profile.bundles 列出的每个 bundle 的 cordis.patch.yml
  → 你的 cordis.patch.yml        ← 只改这一层
  → 命令行 --patch 覆盖
```

所以**只改用户补丁层**，不要去动 `cordis.yml`（它是空的根，注释里明确写了 "Edit cordis.patch.yml, not this file"）。

| 平台 | 补丁文件路径 |
|---|---|
| 通用 | `<DSH_HOME>/profiles/<profile>/cordis.patch.yml` |
| Windows 默认 | `C:\Users\<你>\.dsh\profiles\<profile>\cordis.patch.yml` |
| macOS / Linux 默认 | `~/.dsh/profiles/<profile>/cordis.patch.yml` |

`<DSH_HOME>` 可用环境变量 `DSH_HOME` 覆盖。

**关键点：`@deepseek-ai/*` 系列插件由 DSH 安装包自带**（在 `resources/app.asar/dsh/node_modules/` 内），profile 目录下没有 `node_modules` 也能按名字解析。所以插入 MCP 客户端**不需要跑 pnpm install**。

---

## 二、前置条件

| # | 条件 | 怎么确认 |
|---|---|---|
| 1 | 已安装 KwaFlux 应用 | 存在 `<KwaFlux 安装目录>\Tools\kwaflux-mcp\kwaflux_mcp.mjs` |
| 2 | MCP 入口脚本齐全 | 同目录应有 `agent_control.mjs`、`next_actions.mjs`、`node_modules\@modelcontextprotocol\`、`generated\enhance_catalog.mjs` |
| 3 | Node 可用 | `node --version`（本机为 `v24.18.1`）。脚本要求 Node 18+，并从同目录 `node_modules` 加载 `@modelcontextprotocol/sdk` 和 `zod`，不需要再 `npm install` |
| 4 | 知道目标是哪个 profile | 见下一步 |

> **不要**依赖脚本目录里的 `SKILL.md`——它通常不存在，脚本会回落到内置说明文本。

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

`command` 建议先写 `node`（依赖 PATH）。DSH 对 stdio MCP 子进程会**擦除**环境变量中匹配 `/KEY|PASSWORD|SECRET|TOKEN/i` 的名字以及全部 `DSH_*`，**但 `PATH` 会保留**，所以 `node` 能被解析。

若 `node` 不在 PATH，或想锁定版本，就直接写绝对路径：

```yaml
command: 'C:\Program Files\nodejs\node.exe'
```

### 步骤 3 — 确定当前 profile

DSH 把活动 profile 暴露在环境变量里，这是最可靠的判断方式：

```powershell
"DSH_HOME        = $env:DSH_HOME"
"DSH_PROFILE     = $env:DSH_PROFILE"
"DSH_PROFILE_DIR = $env:DSH_PROFILE_DIR"
```

另一条交叉验证途径是看宿主进程的命令行（第二个位置参数就是 profile 目录）：

```powershell
Get-CimInstance Win32_Process -Filter "Name='DeepSeek Harness.exe'" |
  Where-Object { $_.CommandLine -like '*dsh-desktop-host*' } |
  ForEach-Object { $_.CommandLine }
```

> **常见坑**：`desktop` 和 `web` 是**两个独立 profile**。桌面应用用 `desktop`，`dsh web` 用 `web`。补丁只对写进去的那个 profile 生效——写错文件会表现为"配置明明写了但工具不出现"。

### 步骤 4 — 备份并写入补丁

先备份：

```powershell
$patch = "$env:DSH_PROFILE_DIR\cordis.patch.yml"
Copy-Item $patch "$patch.bak" -Force
```

补丁文件是**顶层 YAML 数组**。在数组末尾追加一个 `insert` 条目（注意 `insert` 下是**缩进的二级列表**）：

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

要点：

- `serverName` 是工具命名空间，必须是 `[A-Za-z0-9_-]{1,32}`，且**在同一作用域内唯一**——重复会让后一条加载失败。
- YAML 单引号内**反斜杠是字面量**，Windows 路径直接写即可，不用转义成 `\\`。
- `id` 只是这条配置的标识，可自定，保持唯一即可。

### 步骤 5 — 生效

DSH 会**热重载**补丁文件。实测：改完文件后约 7 秒，宿主就拉起了 MCP 子进程，无需重启。

若热重载没触发，重启 DSH 应用即可。

### 步骤 6 — 验证

**验证 A：MCP 子进程是否被拉起**

```powershell
Get-CimInstance Win32_Process -Filter "Name='node.exe'" |
  Where-Object { $_.CommandLine -like '*kwaflux*' } |
  Select-Object ProcessId, CreationDate, CommandLine
```

预期看到 `node ".\..\kwaflux_mcp.mjs"`，且 `CreationDate` 晚于你改文件的时间。

**验证 B：应用侧是否连上**

让模型调用 `mcp__kwaflux__kwaflux_get_app_status`，预期：

```json
{
  "running": true,
  "platform": "win",
  "app_version": "1.0.4",
  "signed_in": true,
  "entitlement_allowed": false
}
```

**验证 C：任务接口是否通**

调用 `mcp__kwaflux__kwaflux_list_tasks`，能返回任务数组（哪怕是空的）就说明整条链路打通。

---

## 四、故障排查

| 现象 | 原因 | 处理 |
|---|---|---|
| 模型看不到 `mcp__kwaflux__*` 工具 | 补丁写进了错误的 profile | 用步骤 3 确认 `DSH_PROFILE_DIR`，必要时两个 profile 都写 |
| 同上 | YAML 结构错（漏了 `insert` 下的缩进，或根不是数组） | 对照步骤 4 的片段；用 `$patch.bak` 还原后重写 |
| 同上 | 宿主没重载 | 重启 DSH |
| 日志报两个条目同名 | 同一个 profile 里插了两次 `serverName: kwaflux` | 只保留一条 |
| 工具返回 `app_not_running: missing ...control.json` | KwaFlux 应用没在运行 | 启动 KwaFlux |
| 子进程起不来 / `spawn ENOENT` | `node` 不在 PATH | `command` 改成 node 绝对路径 |
| 工具返回 `connect_timeout ...` | KwaFlux 卡在启动或达到实例上限 | 重启 KwaFlux；必要时调大 `KWAFLUX_MCP_CONNECT_TIMEOUT_MS` |
| 调用卡住然后 `call_timeout after 15000ms` | 单次 RPC 超过 15 秒 | 调大 `KWAFLUX_MCP_CALL_TIMEOUT_MS`（见附录 A） |
| 写操作（convert / repair / enhance / download）被拒 | 未登录或 entitlement 不足 | `get_app_status` 看 `signed_in` / `entitlement_allowed`，这是账号问题不是配置问题 |
| 整个 GUI 打不开 | 补丁语法严重损坏 | 用 `cordis.patch.yml.bak` 覆盖回去 |

> MCP 条目连接失败**默认不会阻止 DSH 启动**（`failOnStartupError` 默认 `false`），所以配置错误通常表现为"工具不出现"而不是"应用崩溃"。

---

## 五、回滚

```powershell
Copy-Item "$env:DSH_PROFILE_DIR\cordis.patch.yml.bak" `
          "$env:DSH_PROFILE_DIR\cordis.patch.yml" -Force
```

或者只删掉那个 `- insert:` 条目（连同它缩进的两行子项）。`.bak` 文件不会被 profile 加载器读取，留着无副作用。

---

## 六、跨平台 / 多机差异

换电脑时需要重新确认的只有三处：

| 项 | Windows | macOS / Linux |
|---|---|---|
| MCP 入口脚本 | `C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs` | `/Applications/KwaFlux.app/.../Tools/kwaflux-mcp/kwaflux_mcp.mjs`（以实际安装位置为准） |
| 控制端点目录 | `%LOCALAPPDATA%\KwaFlux\agent\` | `~/Library/Application Support/KwaFlux/agent/` |
| 补丁文件 | `%USERPROFILE%\.dsh\profiles\<profile>\cordis.patch.yml` | `~/.dsh/profiles/<profile>/cordis.patch.yml` |

其余（`name`、`transport`、`serverName`）都相同。macOS/Linux 下 `args` 里的路径用正斜杠并给含空格的部分加引号。

---

## 附录 A — 可选调优字段

`config` 下还支持（详见 DSH 内置的 `dsh-mcp-client` 文档）：

| 字段 | 默认 | 含义 |
|---|---|---|
| `failOnStartupError` | `false` | 置 `true` 时首次连接或工具同步失败会拒绝插件激活。**建议保持默认** |
| `toolCallTimeoutMs` | `60000` | 单次 `tools/call` 或资源请求超时 |
| `maxInstructionBytes` | `32768` | 服务器指令的 UTF-8 字节上限，超限拒绝连接 |
| `reconnect.enabled` | `true` | 断线后自动重连 |
| `reconnect.initialDelayMs` | `500` | 首次重连延迟，每次失败翻倍 |
| `reconnect.maxDelayMs` | `30000` | 退避上限；也是重置尝试预算的稳定时长 |
| `reconnect.maxAttempts` | `10` | 单次中断内连续失败尝试上限 |
| `env` | — | 追加给子进程的环境变量，会覆盖在被擦除过的父环境之上 |
| `cwd` | — | 子进程工作目录（本 MCP 用 `import.meta.url` 定位自身模块，无需设置） |

给 kwaflux 放宽超时的写法：

```yaml
- insert:
    - id: mcp-kwaflux
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: kwaflux
        transport: stdio
        command: 'node'
        args: ['C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs']
        env:
          KWAFLUX_MCP_CALL_TIMEOUT_MS: '30000'
          KWAFLUX_MCP_CONNECT_TIMEOUT_MS: '3000'
```

这两个变量名不含 `KEY`/`PASSWORD`/`SECRET`/`TOKEN`，不会被 DSH 的环境擦除规则命中。

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

## 附录 C — 本机实测记录

| 项 | 值 |
|---|---|
| 配置日期 | 2026-09-24 |
| profile | `desktop`（`DSH_PROFILE=desktop`） |
| 补丁文件 | `C:\Users\Admin1\.dsh\profiles\desktop\cordis.patch.yml` |
| 入口脚本 | `C:\Program Files\Kwaflux\Tools\kwaflux-mcp\kwaflux_mcp.mjs` |
| Node | `C:\Program Files\nodejs\node.exe` v24.18.1 |
| 生效方式 | 热重载，改文件后约 7 秒拉起子进程 |
| 验证结果 | `get_app_status` → `running: true`、`signed_in: true`、`entitlement_allowed: false`；`list_tasks` 正常返回 |

---

## 附录 D — 安装这份 skill

第 3 份文件 `kwaflux-mcp-setup/SKILL.md` 是一个 DSH skill。DSH 按以下顺序扫描 skill 根：

| Rank | 来源 | 路径 |
|---|---|---|
| 100 | 项目 | `<项目根>/.dsh/skills` |
| 200 | 项目 | `<项目根>/.agents/skills` |
| 300 | 自定义 | 由 `customSkillDirs` 配置 |
| 400 | 用户 | `<DSH_HOME>/skills` |
| 500 | 用户 | `<AGENTS_HOME>/skills`（默认 `~/.agents`） |

"项目根"指最近的、含 `.git` 的祖先目录；没有则用当前工作目录。

安装到用户级（跨项目可用）：

```powershell
$skills = if ($env:DSH_HOME) { "$env:DSH_HOME\skills" } else { "$env:USERPROFILE\.dsh\skills" }
New-Item -ItemType Directory -Force $skills | Out-Null
Copy-Item "D:\Project\GitHub\Kwaflux_Manual\deepseek-harness\kwaflux-mcp-setup" $skills -Recurse -Force
```

结果应是：

```
<DSH_HOME>\skills\kwaflux-mcp-setup\SKILL.md
```

规则要点：

- **发现深度只有一层**：只识别 `<root>/<name>/SKILL.md` 和 `<root>/<name>.md`；嵌套的 `**/SKILL.md` 不会被发现。
- frontmatter 必须提供 `name`（**kebab-case**）和 `description`；可选 `whenToUse`、`metadata`、`disable-model-invocation`、`user-invocable`。
- 两个调用开关的键名**必须精确写成带连字符的形式**。写成驼峰（`disableModelInvocation`）或给非布尔值，会导致**整个 skill 被排除**（不是忽略该字段），并记录警告。
- 本 skill 没有设置 `disable-model-invocation`，所以**模型可以自动调用**——在新电脑上直接说"帮我在新电脑上配置 kwaflux MCP"即可命中。若希望只手动调用，在 frontmatter 里加一行 `disable-model-invocation: true`。
- 目录是深度一层的 bundle，`references/`、`scripts/` 等子目录不影响发现。
- 修改正文不需要重启：每次加载都会重新读取当前文件。
