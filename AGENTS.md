# 面向本仓库的 Agent

- 该技能的运行时用法记录在 `SKILL.md` 中；本文件面向审查、审计或修改仓库本身的 Agent。
- **报告缺陷前，先读 `docs/design-decisions.md`。** 其中列出了看似 bug 实为设计决策的行为，每条都附有理由和固定该行为的测试。只有当理由不再成立时才报告，并指出哪句话有误。
- 对照你所审查提交处的代码树核实发现，不要依据摘要：至少有一次早期审查报告了已提交的媒体文件（实际不在 git 中）和已经验证过的颜色标志。
- 契约（`python3 scripts/_contract.py --json`、`docs/contract.md`）是工具名、标志、dry-run 语义和错误类型的权威来源；README 和 SKILL.md 只是复述，并针对契约做了测试。
- 开发规则（范围、测试、PR 期望）见 `CONTRIBUTING.md`。
- **报告的缺陷在修复前先做清扫。** 一份报告只点名一个工具和一种输入；同样的问题通常也出现在别处，报告者能发现是因为这里没人看过。在不点名工具的情况下说出缺陷的类别，运行 `python3 tests/sweep.py INPUT --baseline PLAIN` 以及 `.claude/skills/sibling-sweep/SKILL.md` 中的路径一致性检查，然后修复共同原因，或在一个 Issue 中登记同类问题。PR 正文说明清扫了什么、结果如何。
- **反复出现的类别要加防护，而不是再修一次。** `docs/defect-guards.json` 列出了 `docs/defect-ledger.jsonl` 中所有出现三次及以上的类别，以及 `tests/test_guards.py` 中运行每个工具、在该类别出现处失败的测试。修复一个防护仍为 `pending` 的类别无法通过 `revert_proof.py`：在同一 PR（或更早）中写好防护，若该类别在工具运行时可见，就在公共结果写入器中检测，并让工具声明它做了什么（`runner.declare_time_map` 即此模式）。账本行的 `mechanism` 要写明防护，而不只是被修复的工具。
- **推送前，自己先问审查者会问的问题。** `.claude/skills/code-review/references/reviewer-findings.md` 将外部审查者对早期 PR 提出的 353 条发现归为二十个问题（对每个输入和输出的路径防护、失败的测量被报告为未测量、声明与代码一样窄、正确的流……）。不符合任何一条的发现，要在回答它的 PR 中新增一个问题。
- **缺陷改变的是机制，而不只是一行代码（#632）。** 每次修复都要在 `docs/defect-ledger.jsonl` 中加一行：缺陷的类别、现有检查为何放过了它、改了什么使该类别不再复发（共享步骤、类型、生成文本、覆盖每个工具的测试）。只修补报告处每个类别允许一次，并说明理由；该类别第二次出现意味着机制有误，需要重建。`.github/scripts/revert_proof.py` 和 `test_every_defect_names_its_class_...` 强制执行此规则。
- **工具的可用性是测出来的，不是假设的。** `.claude/skills/first-try-probe/SKILL.md` 让一个新 Agent 仅凭工具名盲写调用，运行它们（`tests/first_try.py`），并把每次失败转成 `docs/stumbles.jsonl` 中的一行，由测试回放。在新增或重命名选项后、工具从 beta 转为 stable 前运行它。
- **没人报告过的缺陷要主动去找。** `.claude/skills/edge-probe/SKILL.md` 给每个数值选项一个零、一个负值和一个极大值（`tests/edge_values.py`），并检查工具应保持的少数性质。在工具从 beta 转为 stable 前、新增数值选项后运行它。