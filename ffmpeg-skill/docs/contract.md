# ffmpeg-skill 执行契约

`ffmpeg-skill contract --json`（或 `python3 scripts/_contract.py --json`）会打印本 skill 的机器可读描述：有哪些工具、每个工具需要什么、接收什么、写出什么、其结果如何验证，以及 agent 可以就 dry-run、输入保留和可重复性做出哪些假设。这是规划 agent 消费的接口，而不是阅读 `SKILL.md`——后者仍面向手工遵循工作流的编码 agent 编写。

整个文档约 950 KB，几乎全是 42 个 `tools` 条目。`--tool NAME`（可重复；`cut` 或 `cut.py`）只保留 `tools` 下指定的工具，按给定顺序，其他键保持不变：单个工具约 55 到 85 KB。未知名称会被拒绝（退出码 2）并列出工具清单，`doctor` 不接受该标志。

契约派生自实际运行的代码，而非在旁边维护：

- 工具列表是 `scripts/` 下所有不以 `_` 开头的脚本；
- 每个 `input_schema` 在打印契约时从脚本自身的 `argparse` 解析器生成，因此新增标志会出现在契约中，无需其他修改；
- 解析器无法表达的事实（角色、必需的 ffmpeg 组件、验证策略、视觉检查策略）存放在 `scripts/_contract.py` 的一张表中，并由 `tests/test_contract.py` 对照脚本、MCP 服务器和安装器进行检查。

## 版本

| 字段 | 含义 | 变更时机 |
|---|---|---|
| `contract_version` | 本文档的形态（`2.0`） | 键被重命名、删除或含义改变 |
| `skill.version` | npm / package.json 版本（`4.7.0`） | 任何发布 |

新增工具或标志的发布保持 `contract_version` 不变；ToolSpec 形态的破坏性变更会提升它。消费者固定 `contract_version`，并读取 `skill.version` 作为来源信息。消费者还应通过 npm 版本或 git 标签固定 `ffmpeg-skill` 本身，而不是跟踪 `main`——见 README 的 "Development"、"Releasing"。

本文件、README、`SKILL.md` 和 `package.json` 描述中的散文式工具数量不是生成的（它在句子中读起来自然），因此 `tests/test_contract.py` 的 `test_docs_tool_count_matches_the_real_tool_list` 在每次 CI 运行时改为对照 `scripts/` 中的真实数量检查全部五处——过时的数量会让测试失败，而不是悄悄漂移。（`SKILL.md` 是在其 "the 28 scripts" 在十二次工具新增期间一直过时而其他三个文件正确之后才加入该检查的。）

## 稳定性保证（4.x）

1.0.0 于 2026-09-11 发布（意外发布：见 CHANGELOG.md 的 1.0.0 条目；该编号被保留而非作废）。从 1.0.3 起，该编号被视为它所暗示的承诺；2.0.0（2026-09-22）是第一次有意的破坏，列在下方 "What 2.0 changed" 下，3.0.0 是第二次（"What 3.0 changed"），4.0.0 是第三次（"What 4.0 changed"）。

4.x 按工具拆分承诺。`contract --json` 中每个工具的 `stability` 说明它属于哪一半：

- **`stable`**：12 个核心工具（`render`、`look`、`caption`、`export`、`check`、`fit`、`cut`、`audio`、`loudness`、`graphics`、`silence`、`probe`；MCP 服务器默认的 `tools/list`）。下表在整个 4.x 期间对它们成立，如同 3.x 中对所有工具成立一样。
- **`beta`**：其余 30 个。下表同样成立，但参数或输出键可在某个次版本中被删除、重命名或设为必需，前提是它已在一个次版本中列于 `deprecated`（而非下面的三步）。工具名和脚本名在 4.x 中不会被删除。

beta 工具会在某个次版本中变为 stable，并附 CHANGELOG 行；stable 工具在 5.0 之前不会变为 beta。

| 表面 | 承诺 |
|---|---|
| 工具 id（`ffmpeg-skill/<name>`）和脚本名 | 永不删除或重命名 |
| CLI 参数（`argparse` dest、标志、位置参数） | 永不删除、重命名或新设为必需；可新增可选参数 |
| `--json` 输出键，以及 `contract --json` / `doctor --json` 的键 | 永不删除或改变类型；可新增键 |
| 退出码（0 成功，1 失败含 ffmpeg 失败，2 `doctor` 中未知/无法判定，124 超时，127 缺少工具，128+信号中断） | 不变 |
| `contract_version`（`2.0`） | 不变；ToolSpec 形态变更属于主版本 |
| MCP `tools/list` 名称和 `inputSchema` 属性名 | 派生自上述内容，因此受同一承诺覆盖 |
| 相同输入和参数下工具的行为 | 仅可为修复缺陷或跟进 FFmpeg 变更而改变，且每次此类变更都有 CHANGELOG 行 |

不覆盖：`--help` 文本、描述、stderr 消息的精确措辞，以及 JSON 输出的 `details`/`notes` 自由文本字段；`scripts/_*.py` 下的内部实现；evals 测试框架；`.claude/` 下的开发 skill。

承诺是被强制执行的，而非靠记忆：`tests/test_contract.py` 的 `test_mcp_tool_surface_matches_the_frozen_1x_snapshot` 对照 `tests/fixtures/mcp_tools.json` 固定每个工具的参数名及哪些是必需的。删除、重命名或新设为必需的参数会让 CI 失败；新增参数会失败，直到重新生成快照（`UPDATE_MCP_SNAPSHOT=1 python3 tests/test_contract.py`），因此 fixture 的 diff 会向审查者准确展示增长了什么。

## 弃用策略

必须移除的东西（被更好参数取代的参数、被证明有误导性的输出键）分三步退役，绝不一步到位：

1. **弃用**于某个次版本：旧形式保持不变继续工作，使用时向 stderr 打印一行指明替代项的警告，CHANGELOG 条目写明 "deprecated"，`--help` 将其标记为 `(deprecated: use ...)`。
2. **保留**至少两个后续次版本或 90 天，以较长者为准。
3. **移除**仅在下一个主版本中，列在该版本 CHANGELOG 的 "Removed" 下，并附首次弃用它的版本。

改变行为的缺陷修复不是弃用：它随补丁发布并附 CHANGELOG 行，如果旧行为是调用者可能合理依赖的，该行会说明。

`contract --json` 在 `contract_version` 旁携带顶层 `deprecated` 列表：下一个主版本将移除的每一项一个条目，`{"what", "since", "replacement", "removed_in", "where"}`，`where` 指明表面（`cli`、`json`、`mcp`、`behaviour`）。它在 2.0.0 中为空，在 3.0.0 中再次为空：其两个 2.x 条目——**运行于 Python 3.9**（`since: 2.10.9`）和**运行于 FFmpeg 5.x**（`since: 2.14.0`）——已由 3.0.0 解决并移入 `removed`（`removed_in: 3.0.0`）。2.x 的警告行也随之消失。3.0.0 要求 Python 3.10+ 和 FFmpeg 6.1+：每个工具都会拒绝版本可读为低于 6.1 的 `ffmpeg`/`ffprobe`（`kind: missing_tool`，`error.reason: ffmpeg_unsupported`；版本不可读的 git 或 vendor 构建不会被拒绝），`doctor` 在 `unsupported` 中列出不受支持的 Python 或 FFmpeg 并报告 `ok: false`，`npx` 安装器会跳过低于 3.10 的 Python，并对较旧的 FFmpeg 打印同样的 "not supported" 行。
`tests/test_contract.py` 会拒绝仍列出该主版本应移除条目的 package.json 主版本提升，或早于第 2 步窗口结束的提升——除非豁免写在此处，如 2.0.0 和 3.0.0 的豁免。

**2.0.0 豁免了第 2 步的 90 天部分。** 其弃用始于 1.10.0（2026-09-13），因此策略允许的最早 2.0.0 是 2026-12-12；2026-09-22 维护者决定不再等待。另一半有充裕余量：十六个次版本（1.11-1.26）承载了每一项，每项都已有 1.x 的 2.0 行为选择加入（`--quality`、`FFMPEG_SKILL_MCP_LEAN=1`、`hdr_signal`、`FFMPEG_SKILL_NO_OVERWRITE=1`），而调用者无法机械准备的那一项变更——提升 `result_v2`——被撤回而非提前发布（见下文）。

**3.0.0 豁免了第 2 步的 90 天部分。**

其弃用始于 2.10.9（2026-10-22，Python 3.9）和 2.14.0（2026-10-24，FFmpeg 5.x），因此策略允许的最早 3.0.0 是 2027-01-22；维护者决定不再等待。另一半成立：十个次版本（2.15-2.24）承载了两项弃用，且每项都有准备方式——运行 Python 3.10+ 和 FFmpeg 6.1+。3.0 中改变的 `cut.py` 默认值不是弃用，而是 2.x 中记录的决策（`docs/design-decisions.md`），且每项在 2.x 中已有选择加入标志（`--edit-list`、`--keep-hevc`、`--vfr-guard sampled`）。

## What 2.0 changed

`contract --json` 的 `removed` 列表是本表的机器可读形式，每项变更一个条目，附弃用它的版本。

| 2.0.0 中的变更 | 弃用于 | 2.0 行为 | 从 1.x 迁移 |
|---|---|---|---|
| 在每接受 `--quality` 的重编码工具上，`--crf` 作为 `--quality` 的别名（`export.py` 保留 `--crf`：其预设选择编码器） | 1.10.0 | 移除；`--quality N` 默认采用 `--crf` 的行为（18，`proxy.py` 为 30） | 在曾传 `--crf` 处传 `--quality`；`--crf` 现在是未知参数 |
| MCP `inputSchema` 中的 `json` 和 `progress` | 1.10.0 | 从每个 schema 中省略（如 `FFMPEG_SKILL_MCP_LEAN=1` 所做）；传输层自行设置它们 | 无需操作：仍发送它们的客户端不会被拒绝 |
| `probe` 中 `hdr` 意为 "BT.2020 原色*或* PQ/HLG 传输" | 1.10.0 | `hdr` 仅对 PQ / HLG / Dolby Vision 为 true，等同于 `hdr_signal`（保留）；`bt2020_or_hdr` 承载 1.x 含义，每个编辑工具仍据此路由 | 在曾指旧 `hdr` 处改用 `bt2020_or_hdr`（例如 "需要 10-bit 路径"）；`hdr_format` 仍命名 `BT.2020 SDR` |
| 仅以警告覆盖现有输出 | 1.10.0 | 拒绝（`kind: input`，在任何运行之前，包括 dry run），除非 `--overwrite` | 在意图替换处传 `--overwrite`；`FFMPEG_SKILL_NO_OVERWRITE` 不再起作用 |
| `result_v2` 预览键和 `FFMPEG_SKILL_RESULT_V2` | 1.10.0（它要替换的 v1 键） | 移除；顶层键是 2.0 形态，不变 | 读取顶层键；`--json-brief` 获取简短文档 |

v1 成功键在 1.10.0 中为 `result_v2` 而弃用。2.0 撤回了该弃用：`result_v2` 根据值决定键落在 `metrics` 还是 `details`（数字进 `metrics`，同一键持有 `null` 则进 `details`），因此调用者不先知道值就无法知道去哪里找，每个工具的每个 JSON 消费者都将为此重写。扁平键按工具在 `output_schema` 中类型化，保留。

## What 3.0 changed

`contract --json` 的 `removed` 列表承载两项下限变更（`removed_in: 3.0.0`）；`cut.py` 各行在 2.x 中决定并记录于 `docs/design-decisions.md`（"Decided for 3.0"）。

| 3.0.0 中的变更 | 弃用于 | 3.0 行为 | 从 2.x 迁移 |
|---|---|---|---|
| 运行于 FFmpeg 5.x（`requirements.ffmpeg` `>=5.0`） | 2.14.0 | `requirements.ffmpeg`/`ffprobe` 为 `>=6.1`。每个工具都会拒绝版本可读为低于 6.1 的 `ffmpeg`/`ffprobe`（`kind: missing_tool`，退出码 127，`error.reason: ffmpeg_unsupported`，`details: {version, required}`）；版本不可读的 git/vendor 构建不会被拒绝。`doctor` 将其列在 `unsupported` 中并 `ok: false`；`npx` 安装器打印同样的 "not supported" 行。CI 移除了其 5.1.1 任务 | 安装 FFmpeg 6.1 或更高版本（Ubuntu 24.04 的 apt 包为 6.1） |
| 运行于 Python 3.9（`requirements.python` `>=3.9`） | 2.10.9 | `requirements.python` 为 `>=3.10 (standard library only)`。`npx` 安装器跳过较旧解释器（找不到时警告）；`doctor` 将其列在 `unsupported` 中并 `ok: false`。CI 下限为 3.10 | 用 Python 3.10 或更高版本运行脚本 |
| `cut.py` 默认值：单个 `.mp4`/`.mov` 流复制上的 `-avoid_negative_ts make_zero`；SDR HEVC 源重编码用 x264；`--vfr-guard average` | 2.x 中决定（`docs/design-decisions.md`） | 默认保留编辑列表（`edit_list: true`，画面从 `--start` 开始）；SDR HEVC 源的重编码保持 HEVC；`--vfr-guard` 默认为 `sampled`。`--edit-list` 和 `--keep-hevc` 仍被接受 | `--no-edit-list`、`--no-keep-hevc` 和 `--vfr-guard average` 恢复 2.x 行为 |
| `cut.py` 的 `precision` 和 `keyframe_snapped` | 2.x 中决定（`docs/design-decisions.md`） | `precision` 是最不精确段的，等于 `least_exact_precision`（保留）；`keyframe_snapped` 等于 `start_snapped`：复制的起点确实移动了（布尔值，`--dry-run` 下为 `null`）。`contract_version` 为 `2.0` | 读取 `start_snapped`/`keyframe_snapped` 判断 "起点是否移动"；2.x 的 "端到端流复制" 测试是 `reencoded: false` |

## What 4.0 changed

每一行都是调用者可能依赖的 3.x 行为，因此等到主版本（#632）。没有一项移除工具、标志或输出键。

| 4.0.0 中的变更 | 3.x 行为 | 4.0 行为 | 从 3.x 迁移 |
|---|---|---|---|
| 承诺按工具拆分（`stability`） | 所有工具在同一承诺下 | 12 个核心为 `stable`，其余 30 个为 `beta`（见 "Stability guarantee (4.x)"） | 读取 `contract --json` 中的 `stability`；若依赖 beta 工具的参数，固定版本 |
| `export.py`，单文件视频交付（非 GIF）缺少必需属性：缺少画面或声音流、视频编解码器错误（编辑母版还包括其 profile 和像素格式）、预设尺寸或固定帧率，或时长或 `--max-mb` 限制（#620） | 退出码 0，`verified: false` | 退出码 1，`kind: verification`，列出失败步骤并保留文件。失败的响度步骤仍只设 `verified: false` | 对缺少必需属性的交付读取退出状态（或 `status`），并继续读取 `verified`：响度未达标仍是退出码 0 且 `verified: false` |
| `fit.py --duration`