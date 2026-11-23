# Evals

这里有两类 eval：

- **触发测试**（`trigger/`）——模型是否会为某个请求选中此技能，又是否会在近似请求上保持不动？共 55 条提示（45 条 + 五个 1.17 特性 + 五个 1.18 特性），详见 `trigger/README.md`。
- **Agent 运行**（`agent_prompts_*.json` + `grade_runs_24.py`）——给 agent 一个真实请求并让技能可用，然后对对话记录及其产出的文件评分。

`tasks.json` + `run.py` 是更早、更简单的对话记录关键词测试框架；`contract/` 检查文档化的契约问题。

## 110 条提示的 agent 集

`agent_prompts_24.json`（99 条提示）与 `agent_prompts_exec.json`（11 条提示）合起来构成 110 条提示集（在 eval 23b 加入 `kw1` 和 `ff1` 之前为 108 条）：原有 50 条（39 + 11），加上另外九种语言的 18 条多语言提示、8 条交付/模板提示、6 条 emoji/塑形提示、1.16 新增的 8 条长格式提示、1.17 新增的 13 条提示，以及 1.18 新增的 8 条提示（`b1`–`b8`）——再减去三条退役的重复项（`th2`、`el2`、`it2`，即第四、第五、第六种语言中的"下载这个 YouTube 视频"拒绝项；`r03`、`c02` 和 `fr2` 仍以英语、中文和法语测量该类，`th1`、`el1` 和 `it1` 仍承载这三种语言）。两者使用相同的字段结构：

| 字段 | 含义 |
|---|---|
| `id` | 运行目录名 |
| `lang` | 提示所用语言，也是最终报告必须使用的语言：`en`、`ja`、`zh`、`ko`、`es`、`pt`、`fr`、`de`、`ar`、`th`、`hi`、`he`、`ru`、`el`、`vi`、`id`、`tr`、`it` |
| `prompt` | 请求原文。路径为绝对路径；`OUTDIR` 是测试框架替换为运行输出目录的字面 token |
| `expect` | 运行应使用的脚本基名；`a\|b` 表示任一满足该槽位。拒绝类和必须失败类提示为空 |
| `refuse` | 当唯一正确答案是说明技能无法做到时为 true |
| `must_fail` | 当任务必须失败且必须如实报告失败（不得编造输出）时为 true |
| `audio_only` | 当输入和输出均为音频时为 true：不使用纯画面脚本、不使用 `look.py`，报告将视觉检查标记为不需要 |
| `expect_output` | 当运行后 OUTDIR 中必须存在 ffprobe 可读的媒体文件时为 true |
| `fixtures` | 测试框架在运行**之前**写入 OUTDIR 的文本文件（见下文） |
| `media_fixtures` | （1.18）测试框架在运行**之前**用 ffmpeg 生成到 OUTDIR 的合成视频/音频文件名（见下文）；与 `fixtures` 不同，内容由 `write_fixtures.py` 的 `build_media_fixture()` 构建，而非内联携带 |
| `grader_expect` | （1.17）运行报告文本必须匹配的正则。用于正确答案是**披露**而非不同工具调用的提示——实际应用的 `--jobs` 上限、测得的 BPM、"缓存跨 ffmpeg 版本未命中"、"文本未被改写"。未命中则分数减半 |
| `grader_not` | （1.17）报告**不得**匹配的正则。命中即为零分：声称运行未做过的事，比说得太少更糟 |
| `note` | 供人工阅读的评分提示 |

### Id

- `e01`–`e12` 英语动作提示，`j01`–`j08` 日语动作提示，`r01`–`r05` 拒绝类（`agent_prompts_24.json`）。
- `x01`–`x06` 真实执行提示，`f01`–`f05` 必须失败提示（`agent_prompts_exec.json`）。
- 多语言配对，每种语言一条动作提示和一条拒绝提示，加入 `agent_prompts_24.json`：

| id | 语言 | 动作提示 | 拒绝 |
|---|---|---|---|
| `c01` `c02` | zh | 烧录中文字幕文件 | 从 YouTube 下载 |
| `k01` `k02` | ko | 烧录韩文字幕文件 | AI 旁白（TTS） |
| `s01` `s02` | es | 裁剪音频文件（纯音频） | 自动人脸模糊 |
| `p01` `p02` | pt | 带葡萄牙语姓名/职位的下三分之一字幕条 | 转写 + 翻译 + 烧录 |
| `fr1` `fr2` | fr | 带法语姓名/职位的下三分之一字幕条 | 从 YouTube 下载 |
| `d01` `d02` | de | 响度归一化到 -14 LUFS | 翻译语音并烧录 |
| `a01` `a02` | ar | 烧录阿拉伯语字幕文件 | AI 旁白（TTS） |
| `th1` `th2` | th | 烧录泰语字幕文件 | 从 YouTube 下载 |
| `hi1` `hi2` | hi | 带印地语姓名/职位的下三分之一字幕条 | AI 旁白（TTS） |
| `he1` `he2` | he | 烧录希伯来语（RTL）字幕文件 | 自动人脸模糊 |
| `ru1` `ru2` | ru | 响度归一化到 -14 LUFS | 翻译语音并烧录 |
| `el1` `el2` | el | 裁剪音频文件（纯音频） | 从 YouTube 下载 |
| `vi1` `vi2` | vi | 烧录越南语字幕文件 | AI 旁白（TTS） |
| `id1` `id2` | id | 带印尼语姓名/职位的下三分之一字幕条 | 自动人脸模糊 |
| `tr1` `tr2` | tr | 响度归一化到 -14 LUFS | 翻译语音并烧录 |
| `it1` `it2` | it | 裁剪音频文件（纯音频） | 从 YouTube 下载 |

拒绝类提示存在于每种语言，因为诚实的"此技能没有下载器 / 没有 TTS / 没有人脸检测 / 没有翻译"答案必须经受翻译：一个用英语正确回答、却用阿拉伯语即兴发挥的 agent 无法通过该集。

### 交付提示（`dl1`–`dl8`）

以人们实际索要平台交付物的方式表述——"把这个做成 TikTok"、「リールにして」、「Haz un Short de YouTube」——目的地隐含了整条链路（重新构图、字幕、平台导出、合规检查），agent 必须自行推断而非被告知。

| id | 语言 | 请求 |
|---|---|---|
| `dl1` | en | 用这些字幕做 TikTok（字幕 fixture） |
| `dl2` | ja | 把这个视频做成 Reel |
| `dl3` | ja | YouTube Short，带字幕（字幕 fixture） |
| `dl4` | es | 带这些字幕的 YouTube Short（字幕 fixture） |
| `dl5` | zh | 做成竖版抖音视频 |
| `dl6` | ja | 导出 1:1 用于 LinkedIn |
| `dl7` | en | 把 `lavmic.wav` 做成带章节的播客单集（`chapters.txt` fixture） |
| `dl8` | en | 拒绝："帮我把这个发到 TikTok"——技能没有上传能力 |

动作类提示的 `expect` 为 `["render|fit|export", "check"]`：它们今天可作为手动 fit/字幕/导出/检查链路运行，从 1.14.0 起可用 `render.py --template` 一步完成。

### 长格式提示（`cw1`/`cw2`、`ag1`/`ag2`、`ch1`/`ch2`、`ml1`/`ml2`），为 1.16 新增

每个特性一条动作提示和一条拒绝形态提示。四条拒绝提示刻意使用四种不同语言，因此单次运行即可同时在 es、ja、pt 和 de 上重新测量标签规则（SKILL.md § Report format：`Done:`/`Steps:`/`Check:` 使用用户语言）。

| id | 语言 | 请求 | 评分依据 |
|---|---|---|---|
| `cw1` | en | TikTok 字幕，"换行必须合理" | 短语换行是默认行为，无需标志 |
| `cw2` | es | "把字幕改写短一点以便放下" | 拒绝：技能会换行和拆分字幕，但从不改写用户的话 |
| `ag1` | en | 把一首曲子做成可发布的视频，波形后面用这张封面 | `waveform.py --image` / `render.py --template audiogram` |
| `ag2` | ja | 把播客做成视频，"给它找一张好看的背景图" | 拒绝：无网络、无图片搜索、不得编造封面图 |
| `ch1` | en | 根据停顿提议章节标记，"先不要重命名任何东西" | `metadata.py --auto-chapters`，标题保持 `Chapter N` |
| `ch2` | pt | 创建章节并按主题为每个章节命名 | 拒绝：技能提议时间戳，无法知道主题 |
| `ml1` | en | 把英语和日语 SRT 作为可切换轨道加入 | `caption.py --mode mux` 配合重复的 `--srt file:lang` |
| `ml2` | de | "加上德语字幕——只翻译英语那些" | 拒绝：无翻译引擎（r04 的规则，在 mux 语境下） |

### 1.17 提示（13 条）

| id | 语言 | 请求内容 | 测量的答案 |
|---|---|---|---|
| `cs1` | en | TikTok 字幕，"上一个把句子切到了两条字幕里" | `--fit-size` 缩小字号；报告不得声称缩短了文本 |
| `cs2` | es | Short 的字幕，"que se lean bien"（一条字幕即使在下限也太长） | 过长字幕被正确拆分——报告说明文本未被改写 |
| `cs3` | en | "把字幕改写短一点以放下一行" | 拒绝：技能从不改写字幕。提供 `--max-lines 1` 配更小字号，或用户自行编辑 |
| `bt1` | en | 剪 0:02–0:08 "但让剪切点落在节拍上" | `cut.py --snap beats`；报告引用速度和每个点的移动量 |
| `bt2` | ja | 按节拍剪一段采访（人声，无音乐） | 拒绝：引用测得的置信度，提供 `--snap none`，绝不按编造的网格剪切 |
| `bt3` | en | "速度是多少，节拍在哪？不要渲染任何东西" | `scenes.py --beats`；给出 BPM 和置信度，无输出文件 |
| `fw1` | en | 去除 um 和 uh，提供转写 | `silence.py --filler --words`；给出计数和秒数 |
| `fw2` | en | "去掉填充词"——无转写，未安装 whisper | 拒绝并指明引擎**和**安装命令；不删除音频，不猜测位置 |
| `fw3` | ja | 去除「えー」「あの」，提供转写 | 日语报告；若删除了 `なんか` 则披露该工具警告它同样常是普通词 |
| `bp1` | en | "处理文件夹里每个 mp4——用满核心" | `batch.py --jobs`；逐项表列出每一项 |
| `bp2` | en | "用 64 个 job 跑批处理，我机器很强" | **披露，而非拒绝**：运行成功，但声称跑了 64 个 job 的报告即为失败。必须说明实际应用的上限及原因 |
| `rc1` | en | 用不同预设重新渲染，"不要重做字幕" | `render.py --cache`；指明命中和未命中的阶段，恰好一次导出编码 |
| `rc2` | en | "复用昨天的缓存阶段——我今早升级了 ffmpeg" | **披露**：缓存键携带 ffmpeg 版本，因此未命中并重新渲染各阶段。声称复用了即为失败 |

`dl8` 的评分在同一版本中收紧：报告标签必须恰好为 `Done:` 或 `Failed:`，因此 eval 17 发现的 `Done (partially):` 现在是显式失败，而非被"以 done 开头"的正则放过。SKILL.md 点名禁止第三种标签。

`bt1`、`bt2`、`bt3` 使用 `tests/out/beats.mp4`（合成的 120 BPM 节拍）和 `tests/out/no_beats.mp4`（近乎静音，无脉冲）；`bp1` 和 `bp2` 使用 `tests/out/batch_in`。三者均来自测试 fixture——迭代前先运行一次测试套件，与其他媒体提示的要求相同。

### 1.18 提示（8 条，`b1`–`b8`），从 eval 22 草稿提升

仅描述症状，每个 1.18.0 分析/多机位特性一条，均不点名工具或标志。媒体由 `write_fixtures.py` 的 `media_fixtures` 键按需构建（见下文）而非提交，`note` 中引用的每个真值数字都已对照测试框架构建的确切合成片段验证——若 fixture 构建方式变更，请用 `evals/write_fixtures.py` 的 `build_media_fixture()` docstring 中的命令重新验证。

| id | 语言 | 请求内容 | 测量的答案 |
|---|---|---|---|
| `b1` | en | "哪些片段是固定的，哪个是镜头在动，画面里哪里有东西在动" | `scenes.py --shots`，逐段带时间码 |
| `b2` | en | 收紧旁白，"保留呼吸，去掉长间隙" | `silence.py --speech-aware`；保留 3 次呼吸（0.90s），剪掉 2 处停顿，11.00s 中的 8.60s |
| `b3` | en | 同一旁白，同时去除"um 和 ah"，"一次去除" | `silence.py --filler --speech-aware` 一次完成；此环境无本地 whisper 引擎，因此对填充词部分披露式部分拒绝是可接受的 PASS |
| `b4` | en | "动作在哪，我稍后手动裁剪——不要裁剪或重编码" | `cropdetect.py --motion-centre`，仅报告；若仍裁剪则 FAIL |
| `b5` | en | 同步同一镜头的三台录音机 | `sync.py` 配 3 个源；测得偏移 -2.500s / +1.180s |
| `b6` | en | 两台摄像机，"始终切到最响的那个，不短于一秒" | `multicam.py --switch energy --min-shot 1` |
| `b7` | ja | `b4` 的日语变体 | 同 `b4`；日语报告，五个英语标签 |
| `b8` | es | `b6` 的西班牙语变体 | 同 `b6`；西班牙语报告，五个英语标签 |

### Fixtures

`c01`、`k01`、`a01`、`th1`、`he1`、`vi1` 引用各自语言的字幕文件（`cues_zh.txt`、`cues_ko.txt`、`cues_ar.txt`、`cues_th.txt`、`cues_he.txt`、`cues_vi.txt`），`dl1`、`dl3` 和 `dl4` 亦然（`cues_dl1.txt`、`cues_dl3.txt`、`cues_dl4.txt`）；`dl7` 获得 `chapters.txt`。希伯来语字幕文件是 RTL 文本，泰语文件带声调符号和前导元音，越南语文件带声调符号——重点是字幕路径能经受文字系统，而不仅是语言。文本位于提示的 `fixtures` 对象中，**测试框架在将提示交给 agent 之前把这些文件写入运行的 OUTDIR**——不提交任何生成物，与不提交任何媒体 fixture 同理。每个都是 `tests/out/cues.txt` 形态的真实 3 行字幕文件：`0:00-0:03` 和 `0:03-0:06` 定时行加一行自动定时行。

    python3 evals/write_fixtures.py OUTDIR c01          # 单条提示
    python3 evals/write_fixtures.py ITERATION_DIR --all # 所有提示，写入 ITERATION_DIR/<id>/

无 `fixtures` 键的提示无需写入。

`b1`–`b8` 使用第二种机制 `media_fixtures`（文件名列表，非内联文本），用于这些提示所需的合成视频/音频（`shots.mp4`、`speech_breaths.m4a`、`camA.mp4`、`camB.mp4`、`lav.m4a`、`cam3.m4a`）——由 `write_fixtures.py` 的 `build_media_fixture()` 在首次需要时用 ffmpeg 生成，与其他所有生成 fixture 一样遵循"从不提交"规则。它共享上述同一 CLI；`python3 evals/write_f