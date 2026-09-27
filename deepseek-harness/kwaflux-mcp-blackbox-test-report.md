# KwaFlux MCP 黑盒测试报告

| 项 | 值 |
|---|---|
| 被测对象 | kwaflux-mcp `0.5.0`（经 DSH MCP 客户端 `0.1.7-rc.1` 调用） |
| 宿主 | KwaFlux 应用 `1.0.4` / `uilayer-host` / protocol 1 |
| 测试素材 | `source.mp4`（HEVC Main / yuv420p 8bit / bt709 / 1280×720 / 30fps / 120 帧 / 4.05 s / 551,423 B） |
| 隔离目录 | `C:\Users\Admin1\Videos\Kwaflux\test\blackbox\`（素材副本 `input.mp4`，原文件未改动） |
| 执行日期 | 2026-09-24 |
| 用例总数 | 22（有效执行 22） |
| 结果 | **PASS 18 / 产品缺陷 2 / 文档偏差 2** |

---

## 一、方法论声明

**这不是盲测。** 测试者事先已读过 `kwaflux_mcp.mjs` 与 `agent_control.mjs` 源码，无法假装不知道实现。

为抵消这一偏差，采用**预注册期望值**：每个用例先写下预期结果（依据是源码契约、`dsh-mcp-client` 文档、以及已交付的《工具手册》），再执行，最后只判断"符合 / 不符合预期"。凡是不符合预期的都如实列为发现，不做事后合理化。

因此本报告的价值不在于"首次探索行为"，而在于**验证实现是否兑现了契约、以及已交付文档是否准确**——事实上它推翻了两处我此前写进《工具手册》的说法。

---

## 二、结果汇总

### 2.1 只读层

| # | 用例 | 预注册期望 | 实际结果 | 判定 |
|---|---|---|---|---|
| R1 | `get_app_status` | 返回 running / signed_in / entitlement | `running:true`、`signed_in:true`、**`entitlement_allowed:true`** | PASS |
| R2 | `list_tasks(limit=3)` | ≤3 条，字段合法 | 返回 2 条（库里仅 2 条），字段合法 | PASS |
| R3 | `get_task(有效 id)` | 返回任务对象 | 返回，但**不含 `result` 字段** | 偏差 |
| R4 | `get_task(不存在 id)` | 报错而非崩溃 | `-32005 invalid_args`，`data` 为空对象 | PASS |
| R5 | `wait_task(终态任务)` | 立即返回，不耗等待预算 | `done:true`、`timed_out:false`、`waited_ms:105` | PASS |

### 2.2 参数校验层（应在客户端被拦截）

| # | 用例 | 预注册期望 | 实际结果 | 判定 |
|---|---|---|---|---|
| S1 | `wait_task(timeout_s=0)` | 拒绝，提示下界 | `-32602 Too small: expected number to be >=1 at timeout_s` | PASS |
| S2 | `wait_task(timeout_s=999)` | 拒绝，提示上界 | `Too big: expected number to be <=300` | PASS |
| S3 | `list_tasks(limit=500)` | 拒绝，提示上界 | `Too big: expected number to be <=200` | PASS |
| S4 | `enqueue_convert(profile="ultra")` | 拒绝，列出合法枚举 | `Invalid option: expected one of "balanced"\|"quality"\|"size"` | PASS |
| S5 | `enqueue_convert(input_path="")` | 拒绝，提示下界 | `Too small: expected string to have >=1 characters` | PASS |

5/5 全部在**客户端 zod 层**被拦截，未触及 KwaFlux 应用——符合"不把非法请求下传"的设计意图。

### 2.3 错误路径

| # | 用例 | 预注册期望 | 实际结果 | 判定 |
|---|---|---|---|---|
| E1 | `cancel_task(不存在)` | 结构化错误 | `invalid_args` + `data.reason="task_not_found"` | PASS |
| E2 | `resume_task(不存在)` | 同上 | 同上 | PASS |
| E3 | `retry_task(不存在)` | 同上 | 同上 | PASS |
| E4 | `pause_task(不存在)` | 同上 | 同上 | PASS |
| E5 | `diagnose_media(不存在路径)` | 输入不存在错误 | `invalid_args` + `data.reason="input_not_found"` | PASS |
| E6 | `enqueue_convert(不存在路径)` | 同上 | `invalid_args` + `data.reason="input_not_found"` | PASS |

> 为避免改动用户真实的任务记录，E1–E4 一律使用伪造 id，而不是拿历史的 succeeded 任务去试"取消已成功任务"。副作用可控优先于覆盖更细的分支。

### 2.4 功能路径

| # | 用例 | 预注册期望 | 实际结果 | 判定 |
|---|---|---|---|---|
| D1 | `diagnose_media(健康文件)` | 返回可修复性判定 | `succeeded`／`damageLevel:"none"`／`repairable:false`／`damageDescription:"file is healthy"`／`hasMoov:true`／`hasValidMdat:true`／`majorBrand:"isom"`／`isFragmented:false`／`recommendedMode:"auto"` | PASS |
| W1 | `enqueue_convert(quality)` | 产出文件 | `task-…-5` succeeded，产出 `input_kwaflux_quality.mp4` 4,315,623 B | PASS |
| W2 | `enqueue_convert(balanced)` | 产出文件 | `task-…-6` succeeded，产出 2,219,973 B | PASS |
| W3 | `enqueue_convert(size)` | 产出文件 | `task-…-7` succeeded，产出 872,148 B | PASS |
| W4 | **`output_path` 指定诱饵路径** | 诱饵路径被忽略 | `DECOY-should-be-ignored.mp4` **未生成**，产物仍落在输入旁 | PASS |
| W5 | 产物可解码性 | 无损坏 | 三档产物 `ffmpeg -f null -` 全帧解码，`decode_exit=0`、0 条错误 | PASS |
| P1 | `enhance_fx_smart_cropping(1x1)` | 产出方形视频 | **任务失败** `error_code:-10505`／`error_domain:"vendor:bo"`／`progress:0`／`waited_ms:49` | **缺陷** |
| P2 | `enhance_quality_repair(omni)` | 产出修复视频 | **任务失败**，同错误码，`progress:0` | **缺陷** |
| P3 | `list_mcp_resources(kwaflux)` | 返回资源列表 | `{"resources":[]}`（0 条） | 偏差 |

---

## 三、发现的缺陷与偏差

### 🔴 缺陷 1：AI 增强子系统整体不可用

**现象**：两个不同族系的增强模块（`fx-smart-cropping`、`quality-repair`）均在 **progress 0** 处失败，耗时约 50 ms 与 4.4 s，`error_code: -10505`，`error_domain: "vendor:bo"`，且**未产生任何输出文件**。

**排除过程**：

| 假设 | 判别结果 |
|---|---|
| entitlement 不足 | **排除**。同一时刻 `get_app_status` 报 `entitlement_allowed:true`，且 `enqueue_convert` 三次全部成功——说明写闸门是开的 |
| 某模块特有问题（如无跟踪主体） | **排除**。`smart_cropping`（需跟踪主体）与 `quality_repair`（不需要）失败方式完全相同 |
| 输入素材问题 | **排除**。同一文件 `diagnose_media` 判定 "file is healthy"，`enqueue_convert` 三档全部成功 |
| 客户端/协议问题 | **排除**。任务已成功入队（返回了 task_id），失败发生在本体侧 |

**结论**：故障限定在 `enqueue_enhance` 这一条链路，与客户端和素材无关，属 KwaFlux 本体侧问题。错误码无公开对照表，应用日志中也无明文记录，**根因未能确认**（`vendor:bo` 表示来自业务编排层）。

**影响**：15 个 `kwaflux_enhance_*` 工具当前全部不可用。

### 🟠 缺陷 2：`has_output` 字段在失败任务上不可信

两个**失败**的增强任务都报告 `has_output: true`，但目录中不存在任何增强产物；反过来，`formatDiagnose` 任务 `has_output: false` 却返回了完整的 `result`。

该字段既不能用于判断"是否有产物落盘"，也不能用于判断"是否有结果可读"。调用方不应依赖它。

### 🟡 偏差 3：`get_task` 不返回 `result` 明细

实测对 `compare`、`convert`、`formatDiagnose` 等任务调用 `get_task`，返回字段与 `list_tasks` **完全一致**，均无 `result` 键。

因此《工具手册》中"`get_task` 是唯一能看到 `result` 明细的入口"这一说法**不成立**——至少在这些任务上不成立。已知唯一能拿到 `result` 明细的入口是 `diagnose_media`（它内部轮询后返回 `task.result`）。

### 🟡 偏差 4：同类错误码的形状不一致

| 调用 | `code` | `data` |
|---|---|---|
| `get_task(未知 id)` | -32005 | `{}`（**空**） |
| `cancel/resume/retry/pause(未知 id)` | -32005 | `{"reason":"task_not_found"}` |
| `diagnose_media` / `enqueue_convert`(路径不存在) | -32005 | `{"reason":"input_not_found"}` |

四个控制类工具给出机器可读的 `reason`，而 `get_task` 同样面对未知 id 却**不给出原因**。调用方无法用统一逻辑区分"任务不存在"与其他 `invalid_args` 情形。

### 🟡 偏差 5：`list_tasks` 的字段随任务新旧而变

较新的任务带 `algorithm_label`（如 `"merging"`、`"complete"`），历史的 `compare` / `detect` 任务没有该字段。下游若按固定 schema 解析会踩空。

### 🟡 偏差 6：增强工具不返回 `next_actions`

`enqueue_convert` 返回 `{task_id, next_actions:[{tool:"kwaflux_wait_task",…}]}`，而 `enqueue_enhance` 类工具**只返回 `{task_id}`**。同一套 SOP 提示机制在两个写入路径上不对称。

### 🟡 偏差 7：kwaflux 的 MCP 资源为空

`list_mcp_resources(server="kwaflux")` 返回 `{"resources":[]}`。MCP 客户端确实通告了资源能力，但该服务未暴露任何资源——《工具手册》里列出的两个资源工具对 kwaflux 实际无内容可读。

---

## 四、附带确认的行为

以下为测试中被顺带验证、与文档一致的行为：

1. **输出命名规则**（此前文档未记载，本次实测得出）：`<原名>_kwaflux_<profile>.<ext>`，例如 `input.mp4` + `quality` → `input_kwaflux_quality.mp4`。
2. **`profile` 三档确有实际差异**，且单调：

   | profile | 体积 | 总码率 | 视频编码 | 备注 |
   |---|---|---|---|---|
   | `size` | 872,148 B | 1.72 Mbps | h264 | |
   | `balanced` | 2,219,973 B | 4.38 Mbps | h264 | |
   | `quality` | 4,315,623 B | 8.52 Mbps | h264 | 相对源 1.09 Mbps HEVC 放大约 7.8× |

   三者时长均为 4.053583 s，分辨率 1280×720、30 fps、音频 aac 保留。
3. **`enqueue_convert` 返回的 `next_actions` 指向 `kwaflux_wait_task`**，与文档一致。
4. **`diagnose_media` 的 `next_actions` 在"不可修复"分支指向 `kwaflux_get_task`**，与源码逻辑一致。
5. **`wait_task` 对终态任务立即返回**（105 ms），不浪费等待预算。
6. **`entitlement_allowed` 是实时读取的**：本会话开始时为 `false`，测试中变为 `true`，无需重启即反映变化——这也是本次能测写操作的前提。
7. **速率限制未触发**：累计 8 次计费写操作（1× diagnose、4× convert 有效、2× enhance、2 次无效入队未计）未遇到任何限流响应。

---

## 五、未覆盖项

| 项 | 原因 |
|---|---|
| `enqueue_repair` | 素材健康（`repairable:false`），在健康文件上跑修复测不出真实修复语义；需要真实损坏样本 |
| `probe_download` / `enqueue_download` | 需要外部 URL 与网络访问，且 `compliance_ack` 必须由用户本人确认版权，测试者不得代填 |
| 其余 13 个增强模块 | 增强链路已系统性失败，逐个复测无增量信息 |
| `pause` → `resume` → `retry` 的**成功**路径 | 需要构造可暂停的长任务；本次只覆盖了错误分支 |
| 速率限制的边界值 | 需刻意刷写请求，可能干扰用户正常任务，未做 |
| 并发/重复入队的幂等性 | 未覆盖 |

---

## 六、复现方式

```powershell
$bb = "C:\Users\Admin1\Videos\Kwaflux\test\blackbox"
New-Item -ItemType Directory -Force $bb | Out-Null
Copy-Item "C:\Users\Admin1\Videos\Kwaflux\test\source.mp4" "$bb\input.mp4" -Force
```

随后依次调用（参数见第二节各用例）：

```
kwaflux_get_app_status
kwaflux_list_tasks(limit=3)
kwaflux_get_task(task_id="task-1a0d340727d-1")      # 任意历史 id
kwaflux_get_task(task_id="task-does-not-exist-42")
kwaflux_wait_task(task_id="task-1a0d340727d-1", timeout_s=10)
kwaflux_wait_task(task_id="x", timeout_s=0)          # 期望 schema 拒绝
kwaflux_enqueue_convert(input_path=$bb\input.mp4, profile="quality")
kwaflux_diagnose_media(input_path=$bb\input.mp4)
kwaflux_enhance_fx_smart_cropping(input_path=$bb\input.mp4, aspectTier="1x1")
```

产物校验：

```powershell
& "C:\Program Files\Kwaflux\ffprobe.exe" -v error -show_entries format=duration,size,bit_rate -of default=noprint_wrappers=1 "$bb\input_kwaflux_quality.mp4"
& "C:\Program Files\Kwaflux\ffmpeg.exe" -v error -i "$bb\input_kwaflux_quality.mp4" -f null -   # 无输出即解码无损
```

---

## 七、给使用者的建议

1. **当前不要依赖 15 个增强工具**——先把 `get_app_status` 的 `entitlement_allowed` 与一次真实增强任务跑通再投入生产流程。
2. **不要用 `has_output` 判断产物是否存在**，直接检查文件系统。
3. **不要用 `get_task` 取结果明细**；需要明细时走 `diagnose_media` 这类自带轮询与 `result` 的工具。
4. **不要依赖 `list_tasks` 的固定 schema**，`algorithm_label` 可能缺席。
5. `enqueue_convert` / `enqueue_repair` / `enqueue_download` 与预检工具本次验证工作正常，可放心使用。

---

## 附：测试产物清单

`C:\Users\Admin1\Videos\Kwaflux\test\blackbox\`

| 文件 | 说明 |
|---|---|
| `input.mp4` | 素材副本（551,423 B），未改动 |
| `input_kwaflux_quality.mp4` | quality 档产物（4,315,623 B） |
| `input_kwaflux_balanced.mp4` | balanced 档产物（2,219,973 B） |
| `input_kwaflux_size.mp4` | size 档产物（872,148 B） |

原始 `test\source.mp4` 未被修改。测试未产生意外文件——失败的两个增强任务没有留下任何产物。
