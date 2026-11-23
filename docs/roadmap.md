# 路线图：1.7.1 → 1.21.0 → 2.0

1.x 契约已冻结（docs/contract.md，"稳定性保证"）。1.8 → 1.10 通过可选开关或并行键预发布 2.0 的决策（Issue #189，docs/design-decisions.md "为 2.0 已决定的事项"），以及评估持续暴露的后续问题。1.11 → 1.20 在冻结契约上扩展技能，每个小版本一个主题，每个主题以针对该主题自身提示词的评估迭代和一次审计收尾，与 1.5 → 1.7 的做法相同。2.0.0 随后移除旧拼写并翻转默认值；它不新增任何自有功能。`resolve_version.py` 将 `feat` PR 转为小版本、`fix` PR 转为补丁，因此下面每个区块是一到两个 `feat` PR 加若干修复。

**每个标题带有三种状态之一**，以免"在路线图上"被误认为"在已发布包中"：

- **已发布 + 已评估** — 已发布，并由一个具名评估迭代收尾，结果在 `evals/results/` 中。
- **已发布，评估待定** — 已发布，但尚无评估迭代对其评分。
- **计划中** — 未发布。*计划中*标题下的任何内容在任何已发布版本中都不存在；功能行是意图，不是对代码的描述。
- **计划主题，已被取代** — 1.x 结束前写下的 1.x 主题。其版本号后来被用于一个无关的发布（见上表），其中确实已发布的条目列在对应发布版本下。

当前已发布版本为 **4.7.0**（说明见 CHANGELOG.md）；**4.0.0**（说明见 CHANGELOG.md）是第三次刻意破坏性变更：12 个核心工具保持稳定承诺，其余为 `beta`；平台导出默认满足响度要求（`--no-normalize` 保留 3.x 的仅报告模式）；AAC 片段的流复制拼接若漂移超过 50 ms 则拒绝；独立审查页面展示项目的计划和结果（docs/contract.md "4.0 变更了什么"）。工具数量仍为 42。

| 版本 | 状态 | 证据 |
|---|---|---|
| 1.8.0 | 已发布 + 已评估 | 1.8.0 的 eval 8（`iteration-8.json`） |
| 1.9.0 | 已发布 + 已评估 | 1.9.0 的 eval 9（`iteration-9.json`） |
| 1.10.0 | 已发布 + 已评估 | 1.10.0 的 eval 10（`iteration-10.json`），语料库重跑 101/101 |
| 1.11.0 / 1.11.1 | 已发布 + 已评估 | 1.11.0 的 eval 11，1.11.1 的 eval 12 |
| 1.12.0 | 已发布 + 已评估 | 1.12.0 的 eval 13（`iteration-13.json`） |
| 1.13.0 | 已发布 + 已评估 | 1.13.0 的 eval 14（`iteration-14.json`） |
| 1.14.0 | 已发布 + 已评估 | 1.14.0 的 eval 15（`iteration-15.json`） |
| 1.15.0 | 已发布 + 已评估 | 1.15.0 的 eval 16（`iteration-16.json`） |
| 1.15.0 后的重构 | 已发布，评估待定 | 契约 + MCP 快照及每个 `--help` 字节级一致；323/323 用例 |
| 1.16.0 | 已发布 + 已评估 | 1.16.0 的 eval 17（`iteration-17.json`）；契约和 MCP 快照仅新增；工具数量仍为 42 |
| 1.16.1 | 已发布 + 已评估 | 来自 eval 17 的字幕断行补丁：泰语连续文本和片假名单词绝不在内部断行，`caption.py` 报告 `overlong` 行；eval 18 对包含该补丁的代码树评分，未报告换行缺陷 |
| 1.17.0 | 已发布，已评估（eval 18） | 1.17.0 的 eval 18（`iteration-18.json`）；工具数量仍为 42；契约和 MCP 快照仅新增。两项发现：`render.py` 将平台表的字幕尺寸作为显式 `--size` 转发，导致 `--fit-size` 在每个带字幕提示词所走的路径上从不触发（且项目 Schema 拒绝 `fit_size`）；SKILL.md 未提及任何 1.17 功能，因此节拍、填充词和 `--cache` 各自最多在一次运行中被使用。1.17.1 是补丁 |
| 1.17.1 | 已发布，已评估（eval 19） | 1.17.1 的 eval 19（`iteration-19.json`），对 eval 18 命名的 18 个提示词运行 26 次；工具数量仍为 42，契约仅新增。补丁有效：`--fit-size` 在模板路径上 12/12 触发（24 → 16，`dl4` 降至 13 单位下限，`split` 0，`text_unchanged` 为 true），填充词和节拍首次即正确路由，第三个标签已移除，触发 50/50。一项发现，且比补丁更早存在：`caption.py write_ass` 将平台的垂直安全边距同时写入 `MarginL`/`MarginR` 和 `MarginV`，在 `PlayResX` 1080 下留下 240 px 文本列，因此每个模板所走的 `--animate`/`--karaoke` 路径上画面仍每行堆叠一个词。自 1.14 起存在。1.17.2 是补丁 |
| 1.17.2 | 已发布，已评估（eval 20） | 1.17.2 的 eval 20（`iteration-20.json`），对八个字幕提示词运行 20 次；工具数量仍为 42，契约仅新增。补丁在画面上有效：0/20 次运行每行堆叠一个词（eval 19：12/12 模板运行），TikTok 的 Style 为 `…,54,151,420,1`，`size_used` 15/16/13 与表格一致，报告与画面 18/20 一致，Opus 质量 4.25（3.65）。遗留问题且非排版器职责：`cs3` 改写用户文本（4/4 迭代），一次 `cs1` 运行将 `max_lines` 提高到 4 并绘制四行堆叠，`cs2` 的 32 字母单词在 13 下限下仍超出画面，已披露 3/3 |
| 1.17.3 | 已发布，评估待定 | 来自 eval 20 的两条 SKILL.md 规则，无代码：字幕文本按原样烧录（绝不改写、缩短或转述，即使被要求"使其适应"——`cs3`，4/4 迭代），垂直交付时保留模板的 `--max-lines` 并让尺寸下降（rep3/cs1 将其提高到 4 并绘制四行堆叠）。SKILL.md 其他部分精简以保持在 30,000 字节以内；工具数量仍为 42，契约不变 |
| 1.17.3 后的重构 | 已发布，评估待定 | `_common/text.py`（1,655 行，111 个顶层定义）拆分为 `fonts.py`、`emoji.py`、`drawtext.py` 和 `wrap.py`，`text.py` 为再导出垫片；契约 + MCP 快照及每个 `--help` 字节级一致，无行为变化；596/596 测试 |
| 1.18.0 | 已发布，已评估（eval 21） | `scenes.py --shots`/`--audio-peaks`/`--speech`、`cropdetect.py --motion-centre`、`silence.py --speech-aware`（通过一个 `keep_ranges()` 与 1.17 的 `--filler` 组合）、`sync.py` N≥1 源（`second` 位置参数完全保留，新增 `more_sources`）、`multicam.py --switch energy`/`--edl`/`--min-shot`；工具数量仍为 42，契约和 MCP 快照仅新增。1109/1109 测试。此版本的 eval 21 发现每个工具都正确但均不可发现——SKILL.md 未提及五个标志中的任何一个。1.18.1 是修复 |
| 1.18.1 | 已发布，已评估（eval 22） | 为 `scenes.py --shots`/`--audio-peaks`/`--speech`、`cropdetect.py --motion-centre`、`silence.py --speech-aware` 添加路由行，并为 `multicam.py --switch energy` 扩展行——无脚本更改。SKILL.md 其他部分精简（与 1.17.3 相同风格）以保持在 30,000 字节以内：29,998。`CHANGELOG.md` 的字幕侧边距说明此前被留在 `1.17.1` 标题下而非 `1.17.2`（该行为 #239 实际发布处）；已更正，仅文档，无版本号变更 |
| 1.18.2 | 已发布，已评估（eval 22） | README 的工具表未提及 1.18.0 的五个标志、`sync.py` 的新增额外源位置参数或 `multicam.py --switch energy`；添加了自 1.18.1 起已在 SKILL.md 中的相同单行事实。无脚本更改 |
| 1.18.3 | 已发布，已评估（eval 22） | MCP `tools/list` 默认返回核心 12 个（`render`、`look`、`caption`、`export`、`check`、`fit`、`cut`、`audio`、`loudness`、`graphics`、`silence`、`probe`，根据 eval 17-20 的 `expect` 字段频率选择），通过 `FFMPEG_SKILL_MCP_FULL=1` 可选启用全部 42 个；无论哪种方式，每个工具仍可通过 `tools/call` 按名称调用，`contract --json` 仍描述全部 42 个。工具数量仍为 42，CLI/MCP 参数名不变 |
| 1.18.4 | 已发布，评估待定 | eval 20 的 `cs2`（一个 32 字母西班牙语单词在 `--min-size` 下仍裁剪画面）：`scripts/_common/wrap.py` 仅在原子即使在最低下限下也无法单独容纳时，在列边缘切片（优先选择现有连字符）；`caption.py` 的烧录路径报告新的 `broken_inside_word` 统计键。可容纳的泰语短语或片假名连续文本（1.16.1）可证明不变——对于已容纳的原子，切片分支不可达。无新 CLI 标志，无新增脚本，工具数量仍为 42 |
| 1.19.0 | 已发布，评估待定 | `multicam.py --write-project FILE`（eval 21 的 `mc2` 缺口）：写出一个 `render.py` 项目，其 `clips[]` 精确复现 multicam 自身的剪切决策（每次剪切的 `src`/`in`/`out`，按各摄像机自身时间线偏移），适用于 `--switch energy`、手动 `--switch` 规格或 `--auto`；multicam 自身的合成渲染和 `--edl`/`--offsets-only` 不变。`feat:` 发布，故小版本号提升——并非本文档下方"1.19.0"主题的开始，该主题无关且仍为计划中 |
| 1.19.1 | 已发布，评估待定 | eval 19 的 `fw1`/`fw3`：单独使用 `silence.py --filler`（无 `--speech-aware`）也会跳剪无关的死区静音间隙，约 5 秒未经请求。已修复：除非同时给出 `--speech-aware`，否则跳过通用静音检测，因此单独 `--filler` 仅移除计时的填充词跨度；单独 `--speech-aware` 和 `--filler --speech-aware` 组合不变 |
| 1.19.2 | 已发布，仅文档 | SKILL.md 为 1.18.3 的 MCP 核心 12（此前未记录）和 1.18.4 的超长词列边缘切片添加行，以及本文档自身的路线图更正。无脚本更改 |
| 1.19.3 | 已发布，评估待定 | `graphics.py` 的 `wrapped()` 辅助函数（用于下三分之一、标题、贴纸、钩子和模因模板）启用了与 `caption.py` 在 1.18.4 中相同的 `slice_overlong` 逃生舱，因此通过模板绘制的单个不可断超长原子不再渲染超出画面安全宽度。`broken_inside_word` 以与 `caption.py` 相同的方式报告。可容纳的泰语短语或片假名连续文本不变，由与 `ShapingTests` 相同的字体可用性跳过保护。工具数量仍为 42，此更改无新 CLI 标志——`look.py --ink` 是单独的、尚未版本化的新增项，在其自己的 PR 中跟踪 |
| 1.20.0 | 已发布，评估待定 | `look.py --ink`（#261）：测量 PNG 中非背景像素的数量和比例（静帧、烧录的字幕/图形叠加层或 `--compare` 并排图像），仅像素扫描——无新依赖。`feat:` 发布，故小版本号提升。工具数量仍为 42 |
| 1.21.0 | 已发布，评估待定 | `contract --json` 为每个工具增加 `examples`（#267）：`scripts/_contract.py` 将 SKILL.md 自身的"用户说" / "执行"请求表解析为 `{tool_name: [{prompts, command}]}`，已缓存，因此全部 42 个工具每个至少有一个机器可读示例，且不会与人读的表格漂移。`feat:` 发布，故小版本号提升——并非本文档下方"1.21.0：2.0 冻结"标题的开始。工具数量仍为 42 |
| 1.22.0 | 已发布，评估待定 | `batch.py` 报告 `cut.py` 在文件夹范围内的流复制与混合重编码比率（#269）：`run_step()` 读取每个步骤自身的 `--json` 结果文档（此前仅用于其输出路径），并将 `cut.py` 每次调用的 `reencoded` 汇总为一个 `cut_stream_copy` 摘要（`{calls, stream_copy, reencoded, stream_copy_rate}`）。`feat:` 发布，故小版本号提升。工具数量仍为 42 |
| 1.23.0 | 已发布，评估待定 | MCP `tools/list` 描述缩短为每个一行（#271）：`MCP_STRUCTURED_NOTE` 此前被逐字附加到全部 42 个描述中；现在改为一次性放入 `initialize` 的 `instructions` 字段。对 `tools/call` 和 `inputSchema` 为新增/无操作；冻结的 1.x MCP 快照不跟踪描述。`feat:` 发布，故小版本号提升。工具数量仍为 42 |
| 1.24.0 | 已发布，评估待定 | MCP `prompts` 能力，含五个工作流配方（#272）：`reel`、`podcast`、`multicam`、`delivery_check`、`hdr`。`initialize` 通告 `capabilities.prompts`；`prompts/list`/`prompts/get` 填充一个由 SKILL.md 自身请求表已陈述的事实构建的模板——是配方而非新工具调用，所命名的每个命令行都是 `tools/call`（或 CLI）已能运行的。`feat:` 发布，故小版本号提升。工具数量仍为 42 |
| 1.24.1 | 已发布，已评估（eval 23b） | `caption.py --karaoke-style word`（#276，#278）：每个词一个 ASS Dialogue 事件，活动词按 `--karaoke-scale`（默认 112%）缩放并加粗，过去/未来词着色——并非仅颜色扫描。`loudness.py` 的流复制分支增加 `-movflags +faststart`（#277，#275），修复了 `render.py --template ... --normalize` 交付丢失 faststart 的问题。Eval 23b 的 `kw1`/`ff1` 提示词（`evals/results/iteration-23b.json`）实际确认了两者：生成的 ASS 在活动词上带有 `\fscx112\fscy112`，且经响度归一化的模板-youtube 交付仍 `moov` 在 `mdat` 之前，`check.py` 的响度/真峰值行 PASS |
| 1.25.0 | 已发布，已评估（eval 23b） | `chore:` 发布版本提升；将剩余的 mp4 流复制路径全部切换为 `-movflags +faststart`（#279）。无脚本 API 变更，工具数量仍为 42。由 eval