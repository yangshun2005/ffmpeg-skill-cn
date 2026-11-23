# 契约评测

仅凭 `contract --json`，不读 `SKILL.md`、脚本或 shell 命令，Agent 能否理解 ffmpeg-skill？

`questions.json` 包含十个问题，每个问题都标注了契约中回答它的字段及预期答案：选出正确的工具 id、说出必需的能力、判断是否需要视觉检查、输入是否保留、`--dry-run` 是否可用、区分分析 / 执行 / 验证、根据输入 Schema 和映射构建 argv，以及工具 id 的稳定性。

- `check.py`（确定性，在 `release-check` 中运行）：从实时契约重新计算每个预期答案，因此问题永远不会与 ToolSpecs 脱节。
- Agent 运行：给模型提供契约（`python3 scripts/_contract.py --json --static`）和 `questions.json`，要求输出 `{"<id>": "<answer>"}`，然后用 `python3 evals/contract/check.py --grade answers.json` 评分。

结果：`../results/contract-<date>.json`。