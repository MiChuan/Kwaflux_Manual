# KwaFlux Manual

[KwaFlux](https://kwaflux.com) 的产品说明，以及把 KwaFlux MCP 接到 Codex、Cursor、DeepSeek Harness 和 Hermes 的配置文档。

KwaFlux 是 Windows 与 macOS 上的桌面应用。画质增强、老片修复、抠像和导出都在本机 GPU 上完成，素材不上传。功能索引见 [kwaflux.com/features](https://kwaflux.com/features)。应用安装包从官网下载；本仓库不包含应用源码。

## 功能

官网把 30 多个模块收成四条工作流。下面是各模块在产品介绍里的用途，细节以对应功能页为准。

### Enhance

处理偏软、压缩过的画面：补细节、校颜色、提帧率。

| 模块 | 作用 |
| --- | --- |
| [AI Video Enhancer](https://kwaflux.com/features) | 重建软缩放拉不回来的纹理和边缘 |
| AI Ultra HD | 神经超分，朝 1080p / 4K 恢复纹理，而不是插值放大 |
| AI Color Enhance | 在本机把发灰、褪色的画面拉回色彩 |
| Frame Interpolation | 在已有帧之间插入新帧，倍率为 2×、3× 或 4× |
| AI Image Enhancer | 放大并修复压缩、有噪点或分辨率不足的静帧 |
| AI Video Quality Repair | 清理压缩损伤、噪点和整体发糊 |

### Restore

稳住抖动、修复老素材、清理对白，并处理打不开的文件。

| 模块 | 作用 |
| --- | --- |
| [Restore Old Video](https://kwaflux.com/features/restore-old-video) | 面向数字化老片：上色、胶片颗粒降噪、去隔行、细节重建；另有去雾、去雨 |
| AI Video Stabilization | 压住手持抖动，并标明裁切代价 |
| Audio Enhance | 降噪、去混响、分离人声，和画面在同一条流水线里处理 |
| File Repair | 修复无法播放或无法打开的视频、图片和音频。这是文件修复，不是画质增强 |

老片建议的顺序是先去隔行，再降噪，然后重建细节，最后视需要上色。文件本身打不开时，先走 File Repair。

### Cutout

在没有绿幕的素材上分离主体、换背景、擦掉杂物、修饰人脸。

| 模块 | 作用 |
| --- | --- |
| [AI Smart Cutout](https://kwaflux.com/features/video-background-removal) | 逐帧抠人像或物体，边缘在时间上保持稳定；可换背景、换天空，或导出带 alpha 的素材 |
| AI Person Replace | 抠出人像后直接换成纯色、图片或视频背景 |
| AI Video Beauty | 面向视频的人脸修饰：预设风格、妆容和皮肤处理，而不是把照片工具逐帧套上去 |
| Object Replace | 标记道具或车辆后，在本机替换其所在画面 |
| Sky Replace | 替换户外镜头里发灰或过曝的天空 |
| [AI Object Remover](https://kwaflux.com/features/video-object-removal) | 擦掉走动的人或停着的杂物。动态目标用跟踪，静止杂物用静态填充 |

### Deliver

转换、字幕、风格化，以及保存你有权保留的文件。

| 模块 | 作用 |
| --- | --- |
| Video Converter | 批量转换 MP4、MKV、MOV 等，作为同一条流水线的最后一步 |
| Subtitle Edit | 添加或导入字幕，再烧录或导出软字幕。不做翻译 |
| Smart Effects | 重构图、转风格，或调整人脸光照 |
| File Download | 保存你有权保留的文件 |

导入、预览和导出在同一个工作区里完成。未登录也可以浏览、导入、做质量检测，并渲染 1 秒、3 秒、5 秒预览。导出和转换需要登录以及有效的付费方案。

## 运行环境

| 项 | 要求 |
| --- | --- |
| 系统 | Windows 10 / 11（64 位）；macOS 12 及以上（Apple Silicon） |
| GPU | Windows：2016 年及以后的 Intel、AMD 或 NVIDIA（核显或独显均可；RTX 额外使用 TensorRT）。Mac：Apple Silicon |
| 内存 | 至少 8 GB；更长的素材和队列建议 16 GB |
| 处理位置 | 推理在本机 GPU。网络只用于登录、许可证检查和更新 |

增强只能恢复画面里还在的信息。严重损坏的素材会改善，但不会变成原生 4K。

## 仓库内容

四套客户端文档结构相同：配置说明、工具手册、黑盒测试报告，以及一份可安装的 `SKILL.md`。

| 路径 | 内容 |
| --- | --- |
| [`codex/`](codex/) | Codex：`config.toml` 的 `[mcp_servers.kwaflux]` |
| [`cursor/`](cursor/) | Cursor：用户级 `mcp.json` 的 `mcpServers.kwaflux` |
| [`deepseek-harness/`](deepseek-harness/) | DeepSeek Harness：profile 的 `cordis.patch.yml` |
| [`hermes/`](hermes/) | Hermes：`HERMES_HOME/config.yaml` 的 `mcp_servers.kwaflux` |

| 文档 | Codex | Cursor | DeepSeek Harness | Hermes |
| --- | --- | --- | --- | --- |
| 配置说明 | [guide](codex/kwaflux-mcp-setup-guide.md) | [guide](cursor/kwaflux-mcp-setup-guide.md) | [guide](deepseek-harness/kwaflux-mcp-setup-guide.md) | [guide](hermes/kwaflux-mcp-setup-guide.md) |
| 工具手册 | [reference](codex/kwaflux-mcp-tools-reference.md) | [reference](cursor/kwaflux-mcp-tools-reference.md) | [reference](deepseek-harness/kwaflux-mcp-tools-reference.md) | [reference](hermes/kwaflux-mcp-tools-reference.md) |
| 黑盒报告 | [report](codex/kwaflux-mcp-blackbox-test-report.md) | [report](cursor/kwaflux-mcp-blackbox-test-report.md) | [report](deepseek-harness/kwaflux-mcp-blackbox-test-report.md) | [report](hermes/kwaflux-mcp-blackbox-test-report.md) |
| Skill | [SKILL.md](codex/kwaflux-mcp-setup/SKILL.md) | [SKILL.md](cursor/kwaflux-mcp-setup/SKILL.md) | [SKILL.md](deepseek-harness/kwaflux-mcp-setup/SKILL.md) | [SKILL.md](hermes/kwaflux-mcp-setup/SKILL.md) |

MCP 服务随应用安装，入口是 `<KwaFlux>\Tools\kwaflux-mcp\kwaflux_mcp.mjs`。本机校验版本为 kwaflux-mcp 0.5.0，暴露 28 个工具。依赖已经放在安装目录的 `node_modules` 里，需要 Node.js 18 或更高版本，不需要再执行 `npm install`。各客户端的配置落点和工具名前缀不同，按上表里对应的配置说明操作。

使用 MCP 时有两条已测到的限制，详见黑盒报告：

- 15 个 `kwaflux_enhance_*` 工具当前会失败（`error_code: -10505`，`error_domain: vendor:bo`），没有输出文件。
- 导出容器不含 GIF。`enqueue_convert` 不能指定目标格式或时间范围。需要 GIF 时用应用自带的 `ffmpeg.exe`，步骤写在各配置说明的附录里。

调用 `kwaflux_enqueue_download` 时，`compliance_ack` 必须由用户本人确认自己拥有该内容的权利。

## 许可

[MIT](LICENSE)
