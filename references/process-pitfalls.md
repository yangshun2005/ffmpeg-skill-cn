# 流程陷阱

开发本仓库时犯过（或差点犯下）的错误，与 FFmpeg 或平台行为无关——而是关于安全地做出变更的*流程*。记录下来的理由与 `references/ci-platform-pitfalls.md` 相同：未被记录的错误，会在下一次全新会话毫无记忆地启动时重演。

**本文件是活记录。** 每当因为未先检查某项既有护栏——固定的测试、环境约束、平台在重复尝试下的真实行为——而做出（或差点做出、在落地前被拦下）某项变更时，就在下面添加一条。不要等人来问。

## 在收窄 `required`/`optional` 能力列表前，先 grep 检查它的固定测试

`scripts/_contract.py` 的 `TOOL_META[...]["required"]` 驱动 `doctor` 对每个工具的 `usable` 回答（`_contract.py` 中的 `_tool_usability()` 只读 `required`，从不读 `optional`）。把某项能力从 `required` 移到 `optional`——即使新标志确实使其变为条件性——会静默改变 `doctor` 在缺少该能力的机器上报告的 `usable: no`，对工具的*默认*调用也是如此。

`tests/test_contract.py` 的 `DoctorDetectionTests` 用真实捕获的 `ffmpeg -filters`/`-encoders` 固定数据钉住了特定的 `usable` 结果（例如一个普通的 Homebrew macOS 构建因缺少 `filter:subtitles` 而正确报告 `caption.usable: "no"`）。对 `required` 的变更若未先对照这些检查，可能通过快速单元测试，却仍破坏这一基于固定数据的保证。

在为新增标志添加能力元数据时被抓到两次（#51 中的 `caption.py --mode mux`，#52 中 `doctor` 的 `gpu_encoders`）——两次的修复方式都是*在*编辑 `TOOL_META` 之前 grep `tests/test_contract.py` 中的 `usable` 和 `_doctor(`，而不是等测试失败后才去查。每次 `required`/`optional` 变更，都先做这个 grep。

## Git 标签推送和 GitHub Release 创建在此环境中不可达——因此不在此环境中执行

此处可用的 git 凭据能推送到 `refs/heads/*`（分支），但不能推送到 `refs/tags/*`——已由 git-receive-pack 端点直接返回的 403 确认，不是认证失败，意味着这是刻意的范围限制，不是可以绕过的 bug。GitHub MCP 工具面也没有 `create_release`/`create_tag` 等价物，而用原始 token 直接调用 GitHub REST API 的 `/releases` 端点会被出站代理本身拦截。

已确认一次（在接受之前又重试了一次标签推送，“以防万一”）。不要第二次重试任一路径。自 0.16.11 起，对发布而言这已无意义：`.github/workflows/release.yml` 在每次推送到 `main` 时从 GitHub Actions 创建标签、GitHub Release 和 npm 发布（README，“Development” → “Releasing”），因此会话根本不需要推送标签——合并 PR 就是发布。仍值得知道的一点：该工作流自身的 `git push origin HEAD:main`（自动版本号递增提交）及其标签运行在内置 `GITHUB_TOKEN` 下，按设计不会触发后续工作流运行，因此该递增提交的 `tests` 运行变红并非“缺失”——它从未被调度；合并前的 PR 运行才是算数的那个。

## 一次性探针必须指向一个你刚用 `pwd` 验证过的目录中的副本，绝不能指向“大概是仓库吧”

在验证 `release.yml` 的自动递增脚本时，一次临时运行本应针对仓库的临时副本执行。它却以真实检出作为工作目录运行——`cd` 进入临时路径发生在工作目录在命令之间被重置的 shell 中——其 `git add -A && git commit`、`git tag v0.16.12` 和两个空提交落在了真实分支上，并移动了真实的本地 `v0.16.12` 标签。没有推送任何内容，`git reflog` 加上从 `origin` 对标签的 `git fetch --force` 恢复了一切，但之所以被发现，只是因为 `git status` 显示了会话未编辑过的文件。

规则：会运行 `git commit`、`git tag`、`rm -rf` 或写入工作树的探针，要放在*在同一条命令中*创建*并*验证的目录里（`cd "$DIR" && pwd && ...`），而不是假定自先前 `cd` 的目录；优先用几行 `printf` 构建的固定数据，而非整个检出的副本（副本携带真实 `.git`，那里的错误就是真实历史中的错误）；之后在提交任何内容前，对真实仓库运行 `git status`。`.claude/skills/destructive-operations/SKILL.md` 对工具本身说了同样的话——它也适用于测试工具的人。

## 一个定量测试在多次固定数据重设计后以三种不同方式失败，意味着问题在平台，不在固定数据

`test_stabilize_reduces_frame_to_frame_motion`（macOS CI，`stabilize.py`）连续三次以独立重设计的抖动固定数据失败——每次直觉都是“固定数据的频率肯定错了”，每次重新调校的固定数据在下一次 CI 运行中以*不同*方式失败。实际原因（libvidstab 在 Linux 和 macOS ffmpeg 构建间行为不同）从第一次失败就可诊断：一个合成固定数据在一种实现下可靠改善、在另一种实现下可靠恶化，是两种实现不一致的证据，而非固定数据校准错误的证据。

如果定量断言在一个平台上失败、经受住重设计、又在*同一*平台以不同方式失败：停止重设计固定数据。要么把严格断言限制在可证明正确的平台上（在所有地方保留较弱的、平台通用的检查——输出存在、时长正确），要么在花费第三个 CI 周期前上报。

## 在 CHANGELOG.md 当前版本段起草后合并的修复可能静默漏掉它

`CHANGELOG.md` 的 `## 0.12.0` 段只写了一次，覆盖到那时为止合并的所有内容。此后关闭真实 Issue 的两个修复（#62 的 `--audio-stream` 扩展经 PR #72，#77 的 dry-run-dims 修复经 PR #88）落地时没有进一步提醒回去添加条目——#62 的修复实际上得到了条目（其内容被如实描述），但漏掉了 `Closes #62` 链接；#77 则被彻底遗漏，直到一个直接提问（“这不该递增版本号吗？”）促使手动检查。两者都未被 CI 抓到，因为没有任何东西对照实际已关闭的内容检查 CHANGELOG.md。

两次都靠手工抓到，随后用 `tests/test_contract.py` 的 `test_changelog_mentions_every_closed_issue_since_last_tag` 妥善关闭：它沿 `git log` 回溯到最新发布标签，从提交正文中提取每个 `Closes #N.`，若该 Issue 编号未出现在 `CHANGELOG.md` 任何地方则失败。这需要真实历史（`ci.yml` 的 `actions/checkout` 步骤现在正是为此传入 `fetch-depth: 0`——默认浅克隆没有可达标签可供 diff，这会让测试在 CI 中静默跳过自身，而非失败）。如果此测试将来需要豁免一个确实无 changelog 的已关闭 Issue（纯流程说明、重复项、未发布变更的回退），在测试本身中命名豁免并给出理由——不要只是放宽正则或去掉检查。

## 能发布的自动化绝不能从它未编写的文本中获取“major”提示

2026-09-11，三个常规 Dependabot 合并（`actions/upload-artifact` 4→7、`actions/setup-node` 4→7、`dependabot/fetch-metadata` 2→3）被发布到 npm，版本为 **1.0.0、1.0.1 和 1.0.2**。发布流水线（`release.yml`，自 #129 起）从 PR 标签解析下一个版本；当天添加的一条自动标签规则对任何*正文*包含字面破坏性变更标记的 PR 应用 `major`。Dependabot PR 正文逐字引用上游项目的发布说明，而 upload-artifact 的 v5.0.0 说明恰好包含那个短语——说的是*他们的* Node 运行时，与本包无关。一个标签，三次意外 major，九分钟内，所有作业全绿。

两件事使它比一条坏规则更糟：每次 chore 合并都会发布（所以第一次意外之后、有人查看之前又发生了两次），且流水线中没有任何东西把“major 号变了”与任何其他递增区别对待。

修复（本次提交）：不再有任何自动标签规则产生 `major`；`chore`/`ci`/`docs`/`dependencies` PR 被排除在版本解析之外，因此不发布任何内容；`release.yml` 拒绝跨 major 边界自动递增，无论标签如何；发布作业被串行化（`concurrency`），使背靠背合并无法在递增推送时竞争。

通用规则：发布流水线绝不能从它未编写的文本——PR 正文、提交消息和发布说明既是陈述也常是引用——推导不可逆决策（major 递增、发布、标签）。匹配人应用的标签或变更的文件，并让不可逆步骤拒绝任何意外，而非假定意外是故意的。接好此类自动化后，观察头几次真实运行的*结果*（npm、标签），而非其退出码：这里三次运行按作业拥有的每项检查都是“成功”。

## 不存在的 action 输入是警告，不是错误——而“从说明中排除”不等于“不发布”

上述意外 major 的修复（#145）仍为其自身的、仅工作流的合并发布了 **1.0.4**。`release.yml` 中两个假设是错的，且没有任何东西检查任一：

- `release-drafter/release-drafter@v6` 被以 `dry-run: true` 调用，以“只读计算下一个版本”。该 action 没有 `dry-run` 输入。GitHub Actions 将 `Unexpected input(s) 'dry-run'` 记录为*警告*并照常运行该步骤——所以每次发布运行都在实时改写草稿发布，注释中的“只读”是虚构。
- `release-drafter.yml` 中的 `exclude-labels` 被期望使仅 chore 的合并解析为与上一标签相同的版本。它只把这些 PR 从草稿*说明*中移除；版本解析器仍应用 `default: patch` 并报告 last+patch。因此工作流的“相同版本 → 无操作”护栏从未触发。

两者在首次运行的日志和 action 文档化的输入中都可见，两者都被漏掉，因为 PR 的测试计划验证的是 YAML *能解析*、配置*包含*预期键——而非 action *做了*注释声称的事。修复方式是把决策从 action 拿走：`.github/scripts/resolve_version.py` 通过 `gh api` 读取已合并 PR 的标签，无可发布内容时返回空，拒绝 `major`，并在 `tests/test_contract.py` 中有针对每条规则的假标签数据单元测试。

通用规则，现在已两次：接第三方 action 时，先读其 `action.yml` 输入（或首次日志中的 `Unexpected input(s)`）再信任参数；把任何输出决定不可逆动作的步骤视为需用固定输入单元测试的东西，而非靠读其 YAML 确认的东西。并观察首次真实运行的*效果*（标签、npm），两次事故实际都是这样被注意到的。

### 发布递增步骤要求 `## Unreleased` 下有字面 `(nothing yet)` 行

在 #163 后的首次运行中发现（2026-09-11）。`release.yml` 的自动递增用 `assert "## Unreleased\n\n(nothing yet)\n\n" in changelog` 定位 CHANGELOG 插入点。#163 做了自然的事，把说明写在 Unreleased 下，于是递增步骤在推送、标签或发布之前就因断言失败——干净的无操作，但运行变红且无发布。脚本现在把 Unreleased 下的任何内容取入新版本的段，并放回占位符，因此手写说明在那里是受欢迎的。教训：既是锚点又是正文的东西会被编辑；锚定标题，而非占位符文本。

### 内置 GITHUB_TOKEN 无法通过规则集推送发布递增

在主规则集生效后的首次发布中发现（2026-09-11，#166 的运行）：release.yml 的 `git push origin HEAD:main` 被以 GH013 拒绝（“Changes must be made through a pull request”，“8 of 8 required status checks are expected”）。GitHub Actions 不能被添加为规则集绕过参与者（导入拒绝该参与者，UI 不列出它），因此递增推送现在使用 `RELEASE_PUSH_TOKEN` secret——一个仓库管理员的细粒度 PAT，对本仓库有 Contents: read/write——其“Repository admin”绕过生效。PAT 推送会触发工作流（GITHUB_TOKEN 的不会），因此递增提交带有 `[skip ci]`；标签、Release 和 npm 发布都在发起运行中发生。在 PAT 过期前轮换它，否则下次发布会在同一步骤干净地失败，在发布任何内容之前。

### PR 正文中任何位置的 `[skip ci]` 字面量都会跳过 squash 合并上的每个工作流

在 #170 的合并中发现（2026-09-11）。PR 正文逐字引用了新的递增提交消息，包括 `[skip ci]`；squash 合并把 PR 正文复制进合并提交，而 GitHub 在提交消息中任何位置都尊重该标记。那次合并后 `main` 上什么都没跑——没有测试、没有 CodeQL、没有发布——发布只在下一个 PR 合并时才发生。在 PR 正文和提交消息中用文字描述该标记（“skip-CI 标记”），或包裹它使其不匹配，并在任何触及 CI 的合并后检查推送确实触发了预期的运行。

### 相隔几分钟的两次合并：首次发布运行在过时的 main 上递增，其推送被拒绝

在 1.4.2 上发现（2026-09-11）。#167（修复）合并，一分钟后 #171（文档）合并，而 #167 的发布运行仍在递增。并发组串行化运行，但运行检出触发它的 SHA，因此首次运行的递增提交落在 #171 的合并之后，`git push origin HEAD:main` 被以非快进拒绝。第二次运行（#171 的）随后发现两个 PR 都未发布，并正确发布了 1.4.2，所以没有丢失——一次红色运行和一条令人困惑的时间线。release.yml 现在检出 `ref: main`，并在推送前立即把递增变基到 main 上。教训：推送到触发它的分支的工作流必须从分支尖端开始，而非从触发提交开始。

### “失败时清理部分输出”删除了用户的文件

由 1.4.2 审查发现（2026-09-12），自 #78（2026-09-07）起存在。编码失败后移除 0 字节残留的清理以“失败后输出路径存在”为键，而这对运行前就存在、ffmpeg 从未打开的可交付文件同样成立（错误的滤镜参数在图初始化时失败，在 muxer 触及输出之前——在 6.1+ 上）。#163 添加的 --overwrite 同意只保护成功路径；失败路径有自己的删除。第一个修复（快照大小/mtime，未变更文件不动）在 6.1 上通过，在 5.1.1 CI 作业中失败（随 3.0.0 移除 5.x 支持）：FFmpeg 5.x 在选项解析期间、任何滤镜初始化之前打开（截断）输出，因此 ffmpeg 本身已销毁了文件。在每个版本上都成立的修复是绝不让 ffmpeg 写入已存在的路径：针对隐藏的同级临时文件运行，仅在成功时用 os.replace() 覆盖原文件。教训：破坏性步骤必须知道它是否创建了它即将销毁的东西，“存在”不是那种知识；且“工具在触及文件前失败”是版本特定的事实，绝非保证。第二次审查发现了第一次——它刚在此代码旁写下覆盖护栏——未发现的：写了修复的审查者读他们修复的文件，不读旁边的文件。

### --timeout 只在 ffmpeg 说话时有效

同一次审查。--progress 运行器迭代进度管道并逐行比较时钟，因此超时存在的唯一情形（死锁的 ffmpeg，什么都不打印）从未到达比较。非进度路径用 subprocess.run(timeout=) 且没问题，而测试只走了那条路径。教训：截止时间属于循环醒来检查的时钟，绝不属于你等待之物的到达；且“挂起”的测试必须用真正静默挂起的 shim，而非快速失败的 shim。

### README 落后三个次版本

1.8.0（--codec/--quality、export --normalize）、1.9.0（时间语法、hdr_signal）和 1.10.0（已弃用列表、FFMPEG_SKILL_MCP_LEAN、NO_OVERWRITE）各自更新了 SKILL.md、references/scripts.md 和 docs/contract.md，却忘了 README.md 的工具表和契约表；结果表是每次发布唯一触及的 README 行。在 eval 10 后由用户抓到。教训：功能的面清单包括 README，CONTRIBUTING.md 中的 PR 检查清单现在已写明。