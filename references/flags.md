# 共享标志、编解码器与试运行

SKILL.md 快速上手的详细版。脚本上的 `--help` 仍是其完整标志列表；`references/scripts.md` 覆盖各工具的大部分标志。

## 运行脚本

脚本位于 SKILL.md 旁的 `scripts/` 目录；用 `python3 <skill-dir>/scripts/<name>.py` 运行，交付模板在 `templates/`。SKILL.md 足以完成一项任务：SKILL.md 的请求表负责路由请求，而即将运行的脚本上的 `--help` 是最省事的完整标志列表。参考文件的成本与 SKILL.md 相当；只在有具体问题时才打开：`references/scripts.md`（大部分标志；`--help` 列出全部）、`references/devices.md`（iPhone HDR、GoPro、DJI、屏幕录制、Zoom）、`references/gotchas.md`（SKILL.md 末尾单行规则的详细版）。MCP：`tools/list` 默认只显示核心 12 个；其余 30 个（按 SKILL.md 的请求表）仍可通过 `tools/call` 按名称调用，由 `contract --json --tool NAME` 描述；设置 `FFMPEG_SKILL_MCP_FULL=1` 可显示全部 42 个。

如需更短的 MCP 列表，在服务器环境中设置 `FFMPEG_SKILL_MCP_TOOLS`。
参见[命名列表契约](../docs/contract.md#default-toolslist-surface-core-12-opt-in-for-all-42)。

## 共享标志

共享标志（每个脚本都接受；例外会单独注明）：`--dry-run`（`verify.py` 忽略它）；`--json`（输出路径、其探测结果、运行的命令）；`--json-brief`（status/output/verified + 一个 `summary`）；`--fast`（预览质量）；`--progress`（%/ETA）；`--timeout SECONDS`（`kind: timeout`；每次 ffmpeg 运行，默认 1800，`verify.py` 每步，默认 600，`batch.py` 在给出 `--timeout` 或 `--jobs` 大于 1 时对整个批次使用一个预算；`0` 用于长达数小时的编码；小于 0 或大于 500000 会被拒绝）；`--overwrite`（第 7 步）；`--plan FILE`（稍后运行 `render.py FILE` 的试运行；输入变化时拒绝；`render.py` 和 `verify.py` 拒绝 `--plan`）。`-o` 不是共享的：`probe`、`check`、`cropdetect`、`scenes`、`batch` 和 `verify` 都没有。重编码工具还接受 `--hw`（Apple Silicon GPU，大文件；`FFMPEG_SKILL_HW=1` = 默认，不导出）、`--codec h264|hevc|av1|prores` 和 `--quality N`：未设置时，SDR 为 x264（`cut.py`：HEVC 源保持 HEVC，`--no-keep-hevc` 为 x264），HDR 为 x265 Main10；`prores` 需要 `-o NAME.mov`，`h264` 拒绝 HDR（先 `color.py --to-sdr`）。

## `--dry-run` 运行什么

写入工具在 `--dry-run` 下不运行 ffmpeg（ffprobe 仍会读取存在的输入）；测量工具（`probe` 带 `--analyze`、`check`、`sync`、`multicam`、`scenes`、`cropdetect`、`report`、`silence`、`loudness`、`stabilize`；FFmpeg 8.0-8.1.0 上多行 `--text` 的 `overlay`；带 `--auto-chapters` 的 `metadata`；带 `--snap beats` 的 `cut`）仍运行其 ffmpeg 测量，跳过产物/附属文件（`--edl`、`--sheet`、生成的 `.ass`），而 `render` 和 `batch` 将每个阶段或步骤作为各自的试运行执行；`verify` 忽略该标志。按工具查看：`contract --json --tool NAME` 的 `dry_run`。