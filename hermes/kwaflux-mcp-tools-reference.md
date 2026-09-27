# KwaFlux MCP 工具手册（Hermes）

KwaFlux MCP（服务名 `kwaflux-mcp`）暴露的全部 28 个工具的功能表与调用说明。

> 黑盒主体：kwaflux-mcp `0.5.0` / KwaFlux 应用 `1.0.4` / Node `v24.18.1` / Windows，2026-09-24。
> Hermes 侧复验：同一服务 `0.5.0`，应用 `1.0.5`，Hermes `vgit.28e6496`，2026-09-27。`hermes mcp test kwaflux` 枚举到 28 个工具。
> 配置方法见同目录 `kwaflux-mcp-setup-guide.md`；实测结果与缺陷清单见同目录 `kwaflux-mcp-blackbox-test-report.md`。
>
> **本文档已按黑盒测试结果修正。** 标「实测」的条目来自真实调用，而非源码推断。

---

## 一、这是什么

| 项 | 值 |
|---|---|
| 服务名 | `kwaflux-mcp` v0.5.0 |
| 传输 | stdio |
| 工具数 | 28（Hermes `hermes mcp test kwaflux` 枚举到 28 个） |
| 本质 | **桥接器，不是执行器**。它自己不转码、不下载，只把请求转发给正在运行的 KwaFlux 应用 |
| 入口脚本 | `<KwaFlux 安装目录>\Tools\kwaflux-mcp\kwaflux_mcp.mjs` |

调用链路：

```
模型 → Hermes → kwaflux_mcp.mjs (Node, stdio)
     → AgentControl NDJSON → 命名管道 / unix socket → KwaFlux 宿主 → Worker
```

**端点是动态发现的**：脚本读取控制文件拿到管道/套接字地址。

| 平台 | 控制文件目录 |
|---|---|
| Windows | `%LOCALAPPDATA%\KwaFlux\agent\control.json` |
| macOS | `~/Library/Application Support/KwaFlux/agent/control.json` |

宿主正常退出会删掉 `control.json`，被强杀则会残留 `control-<pid>.json`；脚本会回退到按 mtime 取最新、且该 pid 仍然存活的那一个。**应用没运行时所有工具返回 `app_not_running: missing ...control.json`。**

### 工具名的写法

Hermes 把 MCP 工具注册成 `mcp__<服务器>__<工具短名>`，形如：

```
mcp__kwaflux__kwaflux_get_app_status
└─ mcp__ ─┘└服务器┘ └──── 工具短名 ────┘
```

服务器名就是 `config.yaml` 里 `mcp_servers` 下的键。本手册表格一律用**短名**，前面加上 `mcp__kwaflux__` 即为 Hermes 里的注册名。当前 28 个短名加上此前缀后都短于 64 字符；更长的名字会被截断并带上稳定哈希后缀。

服务端自己的提示文本（`instructions`）会成为该命名空间的说明：*Operate a running KwaFlux app via kwaflux_* tools. After enqueue_convert call kwaflux_wait_task. Diagnose before repair. Probe before download; never invent compliance_ack.*

### 超时与重试（可调）

| 行为 | 默认值 | 覆盖用的环境变量 |
|---|---|---|
| 连接管道超时 | 1500 ms | `KWAFLUX_MCP_CONNECT_TIMEOUT_MS` |
| 单次 RPC 调用超时 | 15000 ms | `KWAFLUX_MCP_CALL_TIMEOUT_MS` |
| 连接失败重试 | 3 次，间隔 300 ms | 硬编码 |

连接超时故意设得很短——本地管道要么立刻连上，要么端点就不在，长时间等待没有意义。要放宽就按配置说明附录 A 写 `mcp_servers.kwaflux.env`。

---

## 二、七条通用规则

1. **写操作全部异步。** 入队只返回 `task_id`，必须再调 `wait_task` 拿结果。单次最多等 300 秒；超时返回 `timed_out: true`，但任务仍在后台跑，可以接着等。
2. **输出路径不可自选。** 转换写在输入同目录；修复写 `<name>_repair.<ext>`；增强写在输入旁边。`enqueue_convert` 的 `output_path` 参数会被**强制忽略**（安全设计）。
3. **写操作有速率限制**（按时间窗）。`diagnose_media` 和 `probe_download` 这两个免登录预检也占用同一份额度。
4. **只需要登录 + entitlement 的是**：3 个 `enqueue_*` 和 15 个 `enhance_*`。全部 8 个管理工具、2 个预检、以及资源读取**都不需要**。
5. **只有只读工具会自动重连重试。** 写操作故意不做"断线后重试"——请求可能已经落地，重发会造成二次入队。
6. **路径必须是绝对路径。** 服务端返回时会主动脱敏，`list_tasks` 不含绝对路径。
7. **参数按模块强类型生成。** 枚举收敛成 enum、滑杆带上下界、`visibleWhen` 不成立的参数根本不会发出——所以"选了 portrait 却传 animeTier"不会报错，而是被静默裁掉。

---

## 三、工具表

### 3.1 状态与任务管理（8 个，只读/幂等，无需登录）

| 工具短名 | 参数 | 默认 | 作用 / 何时用 |
|---|---|---|---|
| `kwaflux_get_app_status` | 无 | — | 返回 `running` / `signed_in` / `entitlement_allowed` / `app_version` / `host_version` / `protocol_version`。应用 `1.0.5` 起另有 `agent_download_enabled`（默认 false，需在设置里打开「允许代理发起下载」）。**排查任何问题的第一步** |
| `kwaflux_list_tasks` | `limit` 1–200；`state` = `all`/`active`/`completed`/`failed` | — | 列出任务（已脱敏，无绝对路径） |
| `kwaflux_get_task` | `task_id` | — | 查单个任务。**实测：返回字段与 `list_tasks` 完全相同，不含 `result`**，且未知 id 时 `invalid_args` 的 `data` 为空、不给出 `reason` |
| `kwaflux_wait_task` | `task_id`；`timeout_s` 1–300 | `timeout_s`=60 | 轮询到终态、`paused` 或超时。返回 `{done, timed_out, waited_ms, task}`。**优先用它，不要手写轮询循环** |
| `kwaflux_pause_task` | `task_id` | — | 中止 Worker 但**保留 checkpoint**。仅 pending/running 可暂停；重复暂停视为无害成功 |
| `kwaflux_resume_task` | `task_id` | — | 只能用于 `paused`，从 checkpoint 续跑（占写额度） |
| `kwaflux_cancel_task` | `task_id` | — | pending/running/paused 可取消，也会停止崩溃任务的重试。终态任务返回 `invalid_args` |
| `kwaflux_retry_task` | `task_id` | — | 仅 failed/cancelled/crashed 可用。**复用同一 task_id**，有 checkpoint 就续跑；对 active/succeeded 返回 `task_not_retryable` |

### 3.2 预检（2 个，无需登录，但占用写额度）

| 工具短名 | 参数 | 默认 | 作用 / 何时用 |
|---|---|---|---|
| `kwaflux_diagnose_media` | `input_path`；`reference_path`?；`timeout_s` 1–300 | `timeout_s`=60 | 修复前**必做**。返回 `repairable` / `damageLevel` / `damageDescription` / `recommendedMode` / `referenceRequired` / `smartPossible` / `hasMoov` / `hasValidMdat` / `isFragmented` / `majorBrand` / `estimatedDurationSec` / `estimatedRepairTimeSec` 等，并附 `next_actions` 指明下一步模式。**这是实测中唯一能拿到 `task.result` 明细的入口** |
| `kwaflux_probe_download` | `url`；`timeout_s` 1–300 | `timeout_s`=120 | 下载前**必做**。返回标题 / 上传者 / 时长 / 各清晰度格式（id、ext、分辨率、fps、编解码器、大小）/ 字幕语言 / 是否 playlist、直播、DRM |

> `reference_path` 建议给**同机位同参数**的健康文件，诊断会更准。

### 3.3 入队写操作（3 个，需登录 + entitlement，有速率限制）

| 工具短名 | 参数 | 默认 | 输出位置 |
|---|---|---|---|
| `kwaflux_enqueue_convert` | `input_path`；`profile` = `balanced`/`quality`/`size`；`output_path`（**被忽略**） | — | 输入文件旁边，命名为 **`<原名>_kwaflux_<profile>.<ext>`**（实测） |
| `kwaflux_enqueue_repair` | `input_path`；`repair_mode` = `auto`/`quick`/`deep`/`smart`；`reference_path`? | `auto` | `<name>_repair.<ext>` |
| `kwaflux_enqueue_download` | `url`；`output_dir`（**必须已存在**）；`audio_only`；`max_height` 1–4320；`compliance_ack`（**必填布尔**） | 最佳质量 mp4 | 落进 `output_dir`，按媒体标题命名；`audio_only=true` 出 mp3 |

> **`compliance_ack` 必须由用户本人确认拥有下载权利后才置 `true`。** 服务端注释写明 never set it yourself，`next_actions` 里也固定给 `false` 并附提醒。

### 3.4 AI 增强（15 个，需登录 + entitlement，输出写在输入旁边）

全部额外接收 `input_path`（绝对路径）。**仅支持视频输入。**

| 工具短名 | 可选取值（**加粗**为默认） | 数值参数（默认值） |
|---|---|---|
| `kwaflux_enhance_quality_repair` | `repairType`: **omni**/portrait/anime/nightScene/scene<br>`portraitTier`: **fast**/general<br>`animeTier`: **fast**/general/high_quality<br>`sceneType`: **dehaze**/derain/colorize/filmGrain<br>`nightSceneTier`: **fast**/general | `strength` 0–1（0.6，仅 scene） |
| `kwaflux_enhance_color_enhance` | `strategy`: **sdr2hdr**/colorAdjust/effect<br>`scene`: **landscape**/nightscape/custom（仅 sdr2hdr） | `lightLevel` 2–5（3.8，仅 sdr2hdr） |
| `kwaflux_enhance_video_stabilization` | `stabilizationMode`: **automatic**/subject_lock | — |
| `kwaflux_enhance_video_deinterlacing` | `deinterlaceTier`: **general**/high_quality | — |
| `kwaflux_enhance_fx_smart_cropping` | `aspectTier`: **9x16**/16x9/1x1/3x4/4x3<br>`trackingMode`: **auto**/point/sam2 | — |
| `kwaflux_enhance_fx_cartoonization` | 无参数 | — |
| `kwaflux_enhance_fx_sketch_generation` | 无参数 | — |
| `kwaflux_enhance_fx_fixed_style_transfer` | `style`: **Hayao**/Shinkai/ShinkaiV2 | — |
| `kwaflux_enhance_fx_relighting` | — | `lightAzimuth` −180~180（90，右）<br>`lightElevation` −90~90（40，顶光推荐 30–50）<br>`ambient` 0.5–2（1.1，补偿顶/底光）<br>`gain` 0.5–2（1，越大阴影越重） |
| `kwaflux_enhance_beauty_overall_retouch` | `style`: Elegant/Hollywood1/**Cinema**/Silky/Gloss/Flick/Glamour/Cute/Warm<br>`gender`: **auto**/female/male | `strength` 0–100（60） |
| `kwaflux_enhance_beauty_makeup` | `effect`: **Lips**/Gloss/Matte/Blush/Eyeshadow/Eyeliner/WingedEyeliner/CatEyes/Eyes/Eyelashes/Eyebrows/Foundation/Contour/Bright/BrightGloss/BrightMatte/Dark/DarkGloss/DarkMatte/Makeup/Makeup2/NoMakeup<br>`lipColor`: **native**/classicRed/coral/rosewood/nudePink/berry/orangeRed | `strength` 0–100（60） |
| `kwaflux_enhance_beauty_facial_features` | `effect`: **LongLashes**/ThickBrows/ThinBrows/CupidBow/Dimples | `strength` 0–100（60） |
| `kwaflux_enhance_beauty_face_shape` | `effect`: **ThinFace**/BigEyes/BigLips/SmallNose/BigNose | `strength` 0–100（50） |
| `kwaflux_enhance_beauty_skin_quality` | `effect`: **Smooth**/NoBags/NoWrinkles/Ruddy/Tan/Rough/NoAcne/NoSpots/NoDarkCircles/NoShine/NoRedness | `strength` 0–100（65） |
| `kwaflux_enhance_beauty_hair_recolor` | `hairColor`: **magenta**/cherryRed/auburn/goldenBlonde/ashBlonde/platinum/copper/blueBlack/purple/teal | `strength` 0–100（80） |

> 🔴 **实测警告（2026-09-24）：当前 15 个增强工具全部不可用。** 抽测 `fx-smart-cropping` 与 `quality-repair` 两个不同族系的模块，均在 `progress: 0` 处失败，`error_code: -10505`、`error_domain: "vendor:bo"`，且不产生任何输出文件。同一时刻 `enqueue_convert` 三次全部成功、`diagnose_media` 正常返回，故可排除 entitlement、素材、客户端三类原因，故障限定在 `enqueue_enhance` 链路。详见 `kwaflux-mcp-blackbox-test-report.md`。
>
> 另：增强工具**只返回 `{task_id}`，不返回 `next_actions`**（与 `enqueue_convert` 不对称，实测）。

补充说明：

- `gender` 的 male/female 模型是**分开训练的**，`auto` 表示逐脸 AI 检测。
- `lightLevel` 越大越亮；`scene` 选 landscape/nightscape 会用预设亮度，选 custom 才需要你给 slider。
- `trackingMode`: `auto` 自动找人脸/人；`point` 在暂停帧上点选主体；`sam2` 用 AI 分割跟踪，最稳但最慢。
- `deinterlaceTier`: `general` 是近实时 GPU 场插值；`high_quality` 用 AI 场重建，去梳齿最好但更慢。

### 3.5 MCP 资源（Hermes 按服务器生成的实用工具，非 kwaflux 专属）

| 注册名 | 用法 |
|---|---|
| `mcp__kwaflux__list_resources` | 无参数，列出该服务器的可读资源 URI |
| `mcp__kwaflux__read_resource` | `uri`：要读的资源 |

Hermes 还会为同一服务器生成 `list_prompts` / `get_prompt`。kwaflux-mcp 0.5.0 的 `initialize` 只通告了 `tools`。

> **实测：资源列表为 `{"resources":[]}`。** 这两个实用工具对 kwaflux 没有可读内容。

工具描述和输入 schema 一旦加载便进入上下文——工具越多，固定开销越大。kwaflux 一家就 28 个工具。已打开的会话要执行 `/reload-mcp` 或重新运行 `hermes` 才会看到新服务器。

---

## 四、任务状态机

```
pending ──→ running ──┬──→ succeeded     （终态）
                      ├──→ failed        （终态，可 retry_task）
                      ├──→ cancelled     （终态，可 retry_task）
                      └──→ crashed       （终态，可 retry_task）
                 ↕
              paused   ← pause_task 中止 Worker，保留 checkpoint
                       → resume_task 从 checkpoint 续跑
```

要点：

- **终态 = `succeeded` / `failed` / `cancelled` / `crashed`。**
- **`paused` 不是终态，但 `wait_task` 会立刻返回**，不会继续烧等待预算——因为它不会自己复活。
- `retry_task` / `resume_task` **复用同一个 task_id**，不新建任务。

---

## 五、四条典型链路

```
① 格式转换
   enqueue_convert ──────────────────────────────→ wait_task

② 媒体修复
   diagnose_media ──→(next_actions 给出模式)──→ enqueue_repair ──→ wait_task

③ 媒体下载
   probe_download ──→(确认非 DRM/非直播)──→【用户确认版权】──→ enqueue_download ──→ wait_task

④ AI 增强（当前不可用，见黑盒报告）
   kwaflux_enhance_<模块> ──────────────────────→ wait_task
```

失败分支：

- `wait_task` 回 `paused` → `resume_task`
- 回 `failed`/`crashed` → 先 `get_task` 读 `error_code` / `error_domain`，再决定 `retry_task` 还是改参数重新入队

### 原始调用形态

```json
{
  "tool": "kwaflux_enqueue_convert",
  "arguments": { "input_path": "D:\\media\\a.mov", "profile": "quality" }
}
```

```json
{
  "tool": "kwaflux_wait_task",
  "arguments": { "task_id": "task-1a0d340727d-1", "timeout_s": 120 }
}
```

---

## 六、能力边界（实测确认）

> 🔴 **15 个 AI 增强工具当前全部不可用。** 实测两个不同族系的模块均在 `progress: 0` 处失败，`error_code: -10505`、`error_domain: "vendor:bo"`，无产物。同一时刻转换与诊断工具工作正常，故与 entitlement、素材、客户端无关。详见 `kwaflux-mcp-blackbox-test-report.md`。

1. **不能输出 GIF。** `enqueue_convert` 只能选 `balanced`/`quality`/`size` 三档 profile，**既不能指定目标格式，也不能裁剪时间段**——它的入参只有 `input_path` 和 `profile`。而 KwaFlux 的导出容器只有 `mp4` / `mkv` / `mov` / `avi` / `webm` / `mp3` / `flac` / `wav`（见 `ExportOptionsConfig/formats/default.yaml`），整个 `ExportOptionsConfig/` 目录搜不到 `gif`。
   → 需要 GIF 时，改用 KwaFlux 自带的 `<安装目录>\ffmpeg.exe`（见 `kwaflux-mcp-setup-guide.md` 附录 B）。
2. **不能指定输出文件名和位置。** 全部写死在输入文件旁边。
3. **增强工具只处理视频**，不处理纯音频和图片。
4. **`compliance_ack` 不能被代填。**
5. **会话内路径被脱敏**，`list_tasks` 看不到绝对路径。
6. **`has_output` 不可信（实测）**：两个失败的增强任务都报 `has_output: true` 却没有任何产物；而 `has_output: false` 的 `formatDiagnose` 反而返回了完整 `result`。判断产物请直接查文件系统。
7. **`list_tasks` 的字段随任务新旧而变（实测）**：新任务带 `algorithm_label`，历史的 `compare`/`detect` 任务没有，按固定 schema 解析会踩空。
8. **错误码形状不一致（实测）**：四个任务控制工具在未知 id 时返回 `data.reason: "task_not_found"`，输入路径不存在时返回 `data.reason: "input_not_found"`，但 `get_task` 面对未知 id 时 `data` 是**空对象**，无从区分原因。

---

## 七、容易踩的坑

1. `retry_task` 对 active/succeeded 任务会被拒（`task_not_retryable`），别拿它当"重新跑一遍"。
2. `resume_task` 对非 paused 任务会被拒（`task_not_paused`）。
3. `cancel_task` 对终态任务返回 `invalid_args`。
4. `enqueue_download` 的 `output_dir` **必须已存在**，不会自动创建。
5. 写操作连续压测会撞速率限制——排查问题时尽量只用只读工具。
6. 应用没开时一切工具都是 `app_not_running`，先确认应用在跑再看别的。
7. `entitlement_allowed: false` 时，只读工具照常工作，但所有写操作都会被拒——这是账号授权问题，不是配置问题。
8. 沙箱内的 shell 读不到 `%LOCALAPPDATA%\KwaFlux\agent\control.json`（`EPERM`）；这会让裸协议检查误报 `app_not_running`。Hermes 自己拉起的子进程不走这层沙箱，不要把裸探测的 `EPERM` 当成配置写错。
9. Hermes 没装 `mcp` 这个 Python extra 时，`config.yaml` 里的条目不会变成可用工具。`hermes mcp test` 会直接报 SDK 未安装。见配置说明的前置条件。
