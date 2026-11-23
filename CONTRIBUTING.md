# 为 ffmpeg-skill 做贡献

感谢你考虑贡献。本项目只做一件事：确定、安全、可验证地执行智能体明确给出的 FFmpeg 操作。提出任何建议前，请先读 `SKILL.md` 的“本技能决定什么、不决定什么”——它界定了本项目刻意坚守的边界（不做内容理解、不做创作判断、不集成 AI/LLM、不依赖云服务）。

## 开始之前

本项目遵循[贡献者公约](CODE_OF_CONDUCT.md)。

- **先读 `docs/contract.md` 和 `scripts/_contract.py`。** 能力契约（`contract --json`）是工具 Schema、验证策略和能力检测的唯一事实来源——每个对外界面（MCP、安装器、文档、测试）都由它生成或对照它校验。不要手工复制 Schema，要扩展生成器。
- **小而聚焦于加固的改动最容易合入。** Bug 修复、跨平台正确性、边界情况的测试覆盖、文档漂移修正，永远欢迎。新工具或新功能门槛高得多——见下文“范围”。
- **修 Bug 先扫同类。** 修复已报告的缺陷前，先归纳它属于哪一类，再检查还有哪些工具存在同样问题：`python3 tests/sweep.py INPUT --baseline PLAIN` 会把一个输入跑过 31 个单输入工具（`cut` 跑两次，普通和 `--accurate`），并打印与普通片段不同的行；需要第二个文件或项目的工具在其 docstring 中列出，需手工扫描。完整流程见 `.claude/skills/sibling-sweep/SKILL.md`；PR 模板的“Siblings”一节记录扫描了什么。
- **每个改动都需要复现。** 修 Bug 时，要展示修复前的失败用例和修复后的通过用例——可以是新测试，也可以是 PR 描述中记录的手工复现步骤。

## 范围

本项目刻意**不**做：
- 添加基于 AI/LLM 的场景理解、高光检测或内容判断
- 在 `scripts/*.py` 之外回退到裸 `ffmpeg`/`ffprobe` shell 调用
- 依赖云服务或需要 API 密钥
- 修改输入文件
- 替调用方智能体做创作或构图决策

如果你的想法需要上述任一项，它多半属于另一个互补技能（见 `README.md` 的“Standalone, and in an ecosystem”一节），而不是本项目。比 Bug 修复更大的改动，写代码前先开 Issue 讨论——这能省下所有人一个白费的 PR。

## 开发

```bash
git clone https://github.com/kajisho5/ffmpeg-skill
cd ffmpeg-skill
npm run test:fast           # tests/test_contract.py only: contract, docs, dry runs (~3-4 min)
npm test                    # everything: real encodes (test_all.py's modules, run in parallel by tests/run_parallel.py; about 21 min on 4 cores, 35 serially) + contract
python3 tests/run_parallel.py --part 1/3   # one of CI's three machines' share (test_contract included); about a third of the time
npm run release-check       # packaging + installer + MCP + doctor + full suite
python3 scripts/_contract.py doctor   # what this machine can actually run
```

仅用 Python 3.10+ 标准库——不新增运行时依赖。3.10 是 CI 的下限（3.0.0 移除了 3.9），所以代码必须继续在 3.10 上工作，也要在 FFmpeg 6.1 上工作，这是 CI 运行的最老 FFmpeg。每个脚本都必须只靠 `PATH` 上的 `ffmpeg`/`ffprobe` 就能工作。

## 测试

- `tests/test_contract.py` —— 契约 ↔ 实现 ↔ MCP ↔ 安装器 ↔ 文档一致性、JSON/错误结构正确性、dry-run 保证、能力检测。
- `tests/test_all.py` —— 跨编解码器、容器、VFR、HDR、多轨音频的真实媒体一致性，以及每个工具实际的 ffmpeg 调用。它只导入九个分组模块（`test_analysis`、`test_editing`、`test_audio`、`test_picture`、`test_delivery`、`test_orchestration`、`test_accel`、`test_asr`、`test_cut_copy`）；开发某个工具时，单独跑它所在的分组，例如 `python3 tests/test_editing.py`，或跑单个用例 `python3 tests/test_editing.py EditingTests.test_silence_removal`。

任何行为改动都要附带测试。修复若没有能捕获原 Bug 的回归测试，就不算完成。

## Pull Request

- **已发布代码有上限**（`.github/code-budget.json`，`test_shipped_code_does_not_grow`）。给 `scripts/` 或 `mcp/` 增加行的改动，要删掉同样多的行。优先选增加最少的修法：拒绝优于新行为，共享辅助函数优于复制，删除模式优于加保护。删代码时下调上限；上调需在 PR 正文说明理由。

- PR 保持聚焦——每个 PR 一个修复或一个小功能。
- 开 PR 前本地跑 `npm test`；CI 在 Linux、macOS 和 Windows 上运行，三者都必须通过。
- PR 描述里解释*为什么*，而不只是*做了什么*——推理是未来维护者（人或智能体）最需要的。
- `feat` PR 要更新该功能出现的每一处：`README.md`（工具表、契约表、注意事项）、`SKILL.md`、`references/scripts.md`、`docs/contract.md` 和 `CHANGELOG.md`。README 的读者可能从不打开 SKILL.md，所以只有 SKILL.md 知道的功能等于只发布了一半。
- `SKILL.md` 必须保持在 **30,000 字节**以内——它会被加载进每个会话。自 1.15 起这是测试（`tests/test_contract.py::test_skill_md_stays_under_the_30kb_budget`），不是约定：在那里加一行就要删一行，PR 要说明删了哪行。
- `feat` PR 至少给 `demos/build.py` 和 `docs/demos.md` 加一个前后对比演示（该页面是生成的：把演示加到 `demos/build.py` 的表中，然后运行 `python3 demos/build.py --docs`）。没人能看到效果的功能难以评审，更难信任；演示还会端到端运行新工具，所以坏掉的参数会让构建失败，而不是让读者踩坑。`scripts/` 下的每个脚本都必须出现在某个演示的命令行中——有测试断言这一点。唯一例外是整个输出就是表格、JSON 文档或 HTML 文件的工具：改为把它加到 `demos/build.py` 的 `INSPECTION` 中，它会在画廊的 Inspection 一节获得一条命令和一句话。
- 发布 PR 要在**版本号提升的同一个 PR** 中移动 `docs/roadmap.md` 的“the released version today is ...”一行，这样路线图永远不会把已发布的版本描述为计划中。标签驱动的发布（release.yml 的自动提升）只移动版本号，保留手写文字，并标注其描述的版本（`.github/scripts/bump_roadmap_md.py`）；下一个手写发布 PR 替换该句。同一次提升会移动 `server.json` 中的两个 `version` 字段（`.github/scripts/bump_server_json.py`）。
- 每次发布还会把可安装插件树提交到 `plugin` 分支（`.github/scripts/publish_plugin_branch.sh`）：npm tarball 的内容加上插件清单和 `skills/`，位于该分支的 `ffmpeg-skill/` 下（`git-subdir` 市场源指向该目录）。永远不要手工提交到该分支，也不要强制推送；托管方会快进它。`bash .github/scripts/publish_plugin_branch.sh --tree-only DIR` 在本地构建该树，不碰 git。
- npm 发布失败的发布保留其标签，重跑会跳过发布。通过在该标签上派发 `release.yml` 来完成：`gh workflow run release.yml --ref vX.Y.Z -f version=X.Y.Z`。其 `recover` 任务会拒绝 npm 已有的版本、缺失的标签以及在任何其他提交上的运行，带来源信息发布（仅当版本高于 npm 上所有稳定版本时用 `latest`，否则用 `backport`），然后仅当其顶端是较旧版本时才发布插件分支，且仅对 `latest` 版本发布 MCP Registry 条目：backport 两者都不填。该任务存在之前的标签（直到 v3.6.3）无法用此方式恢复。不要手工发布：本地发布没有来源信息，且不带 `--tag` 会移动 `latest`。README 的“Releasing”和该任务的注释说明了为何这样构建。

## 发布 MCP 列表

MCP Registry 条目由 CI 发布：自 3.5.0 起，`release.yml` 在每次发布的 npm 发布之后发布 `server.json`，通过 GitHub 的 OIDC 令牌登录（不存储密钥），重跑会补上 registry 尚未列出的版本。下面的第 1 步是该步骤失败时的手动兜底；第 2 到 4 步是手动的，CI 不做。

registry 检查的是**已发布**的 npm 包。`package.json` 中的 `mcpName` 必须在该确切版本上等于 `server.json` 的 `name`（`io.github.kajisho5/ffmpeg-skill`），所以先发布 npm（发布工作流会做），再发布 registry。客户端运行 `npx -y ffmpeg-skill mcp`，即 stdio 上的 `python3 mcp/server.py`。

1. 仅当工作流的“Publish to the MCP Registry”步骤失败且重跑无法修复时：安装 [`mcp-publisher`](https://modelcontextprotocol.io/registry/quickstart)，以 **kajisho5** 身份运行 `mcp-publisher login github`（GitHub 认证只允许 `io.github.kajisho5/` 命名空间）。先 `mcp-publisher validate`，再 `mcp-publisher publish`。
2. Smithery：无需运行任何东西。其 API、CLI 和 SDK 于 2026-10-31 停止工作（smithery.ai/future）；该站点仍是精选目录，未记录任何提交服务器的方式，所以本仓库不提供 `smithery.yaml`，也不提供 `.mcpb` 包。
3. Glama：提交 GitHub 仓库，然后认领它。`glama.json` 列出维护者 `kajisho5`。再次认领才会拾取该文件的改动。
4. Scorecard 将 SARIF 上传到代码扫描。该上传需要仓库启用代码扫描。README 中的徽章会在带 `publish_results` 的扫描运行后填充。

## 报告 Issue

先读 [docs/design-decisions.md](docs/design-decisions.md)：它列出了看起来像 Bug 但其实是决策的行为，附有理由和固定每项的测试。如果你的报告涉及其中之一，请指出其中哪句话不再成立。

Bug 报告应包含：运行的确切命令、实际与期望的输出/行为，以及如果问题可能与能力相关，附上 `python3 scripts/_contract.py doctor --json`。参考现有 Issue 了解这里有用的详细程度——具体复现胜过描述猜测。