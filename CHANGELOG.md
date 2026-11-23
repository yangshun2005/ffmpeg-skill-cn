# 更新日志

> `main` 分支领先于最近一次发布的 npm/GitHub 版本；依赖本仓库的项目应固定到带标签的版本，而非 `main`。参见 README § 开发，"发布"。

## 未发布

（暂无）

## 4.7.0

_自动发布：版本号和说明由自 4.6.5 以来合并的 Pull Request 生成。_

### 新增

- **`audio.py --track FILE`：在一个文件中放置多条音轨，每条带名称和语言（#420）。**
  输入自身的音轨（`--audio-stream`）和每个 `--track` 的第一条音频流，在画面之后各成为一条音频流（流复制；音频输入不写入任何内容），支持 `.mov`、`.mkv` 或 `.mp4`。每条按其原样写入：各自的声道布局、采样率和长度（不混音、不重采样、不填充、不裁剪），在 `.mov` 和 `.mkv` 中为 24 位 PCM，在 `.mp4` 中为 AAC。`--track-name NAME`（也作 `--track-title`）和 `--track-lang CODE`（也作 `--track-language`）按轨道顺序重复给出，输入自身的音轨在前；语言为三个小写字母的 ISO 639-2 代码。结果的 `tracks` 列出从文件中读回的 `{index, file, title, language, channels, duration}`，`track_tags` 步骤据此校验名称和语言（`.mov` 只保留 QuickTime 有对应代码的语言，并通过使该步骤失败来表明这一点），比画面长的轨道完整保留，并给出 `track_longer_than_picture` 警告。以下情况以 `error.reason` 拒绝：输出不是 `.mov`、`.mkv` 或 `.mp4`（`track_container`；WAV 只容纳一条流，FFmpeg 的 MXF 封装器不存储名称或语言，且会将每条轨道裁剪到最短），改变声音的标志（`track_rebuilds_sound`），标题或语言数量多于轨道（`track_labels`），以及语言不是三个字母（`track_language`）。带起始时间码的 BWF 或多轨 WAV 不写入。
- feat: audio.py --track 添加带名称、带语言标签的音轨（#420）（#714）

## 4.6.5

_自动发布：版本号和说明由自 4.6.4 以来合并的 Pull Request 生成。_

### 修复

- **混音工具也获得 #711 的重新标记。** `grid.py`、`join.py`、`multicam.py`、`broll.py` 和 `overlay.py` 通过各自的辅助函数构建每个源的 BT.709 转换，4.6.4 的修复未覆盖到。在 FFmpeg 9 上，其中的 HDR 源仍可能以 bt709 流写入，却带有 bt2020/PQ 容器标签。现在所有工具共用同一个辅助函数作为 BT.709 线路的滤镜。

### 新增

- **色彩类别的守卫**（`test_guards.ColorGuard`，三条账本记录）。每个写入视频的工具都在 HDR RGB 源上运行，任何标签混合 SDR 和 HDR 的输出都会使其失败。
- fix: 混音工具的 BT.709 链获得 FFmpeg 9 重新标记，色彩类别有守卫（#711）（#713）

## 4.6.4

_自动发布：版本号和说明由自 4.6.3 以来合并的 Pull Request 生成。_

### 修复

- **在 FFmpeg 9 上，HDR 源上的 BT.709 预设写入了冲突的标签（#711）。** 编码器的 VUI 说 bt709，而 mp4/mov 的 `colr` 盒说 bt2020/smpte2084，因为 FFmpeg 9 从帧中获取容器的颜色。该线路现在重新标记帧的 primaries 和 transfer（`setparams`，不改变像素），因此两者都显示 bt709。FFmpeg 6.1 到 8 已经写入 bt709，保持不变。
- fix: BT.709 线路的容器标签在 FFmpeg 9 上与其流一致（#711）（#712）

## 4.6.3

_自动发布：版本号和说明由自 4.6.2 以来合并的 Pull Request 生成。_

### 修复

- **HDR 或 BT.2020 RGB 源按其线路标记的矩阵进行转换，10 位母版保留其 HDR 标签（#559）。** `export.py --preset youtube`（以及其他 8 位预设，和 `grid.py`）标记 BT.709，却用 FFmpeg 的 BT.601 默认值生成此类源的 YUV：红色 C03028 按其 bt709 标签读作 CE3E26；现在用 BT.709 矩阵转换，读作 C02E27（primaries 和 transfer 仍未转换，`colour_not_converted` 警告仍会说明）。`--preset prores`、`dnxhr-*`、`prores4444`、`prores4444xq`、`vp9-alpha` 以及 `graphics.py` / `overlay.py` 的 `--keep-alpha` 未为其写入颜色标签（ffprobe：unknown/unknown/unknown；8 位 DNxHR 线路在 BT.601 像素上标记 bt709）：它们现在携带源的 primaries 和 transfer 以及 bt2020nc（画面转换所用的矩阵），与 YUV 源一致。
- fix: HDR RGB 源按其线路标记的矩阵转换，10 位线路保留其 HDR 标签（#559）（#708）

## 4.6.2

_自动发布：版本号和说明由自 4.6.1 以来合并的 Pull Request 生成。_
### 变更

- **工具启动更快（#679）。** FCPXML 写入器仅在写入时导入 `xml.sax.saxutils`（该模块会引入 `urllib.request`、`http.client` 和 `ssl`），而不是每个工具在启动时都导入。没有工具的输出发生变化。`cut.py --help`，31 次交错运行的中位数：119 ms -> 83 ms。首次使用时导入转录和文本布局模块又节省了 3 到 4 ms，但会使对重新导出名称的 `mock.patch` 在退出时保留补丁，因此未采用。无探测缓存：一次运行对每个输入探测一次。
- fix: audio.py --duck 在随机点抬起，且每次运行不同（#671）（#705）
- test: 被终止的命令报告其进程当时在做什么（#533）（#704）

## 4.6.1

_自动发布：版本号和说明由自 4.6.0 以来合并的 Pull Request 生成。_

### 变更

- **工具启动更快（#679）。** FCPXML 写入器仅在写入时导入 `xml.sax.saxutils`（该模块会引入 `urllib.request`、`http.client` 和 `ssl`），而不是每个工具在启动时都导入。没有工具的输出发生变化。`cut.py --help`，31 次交错运行的中位数：119 ms -> 83 ms。首次使用时导入转录和文本布局模块又节省了 3 到 4 ms，但会使对重新导出名称的 `mock.patch` 在退出时保留补丁，因此未采用。无探测缓存：一次运行对每个输入探测一次。
- refactor: 仅 FCPXML 写入器导入 xml 链（#679）（#703）

## 4.6.0

_自动发布：版本号和说明由自 4.5.3 以来合并的 Pull Request 生成。_

### 新增

- **`join.py --audio-lead S`（J-cut）和 `--audio-lag S`（L-cut）（#675）。** 使用 `--transition none` 时，下一个片段的音频在其画面之前 S 秒开始（lead），或前一片段的音频在画面切换后延续 S 秒（lag），在每个边界处。保持同步，因此每个边界消耗片段 S 秒：两个 8 秒和 6 秒的片段，S = 0.5，连接为 13.5 秒。S 是输出帧率的整数帧；结果携带 `audio_lead_seconds` / `audio_lag_seconds`，`expected_duration` 为缩短后的长度。以下情况以 `error.reason` 拒绝：与过渡一起使用、与 `--copy` 一起使用、两个标志同时使用，以及 S 小于一帧或不短于一个片段。SKILL.md 将 "J-cut"、"L-cut"、"cutaway"、"cut-in"、"reaction cut"、"insert" 和 "smash cut" 路由到各自的工具；`references/requests.md` 有剪辑师名称的完整表格。
- feat: join.py 中的 J-cut 和 L-cut（#675）（#701）

## 4.5.3

_自动发布：版本号和说明由自 4.5.2 以来合并的 Pull Request 生成。_

### 修复

- **`cut.py --segments`、`cut.py --remove` 和 `silence.py` 将章节随其保留的范围一起移动（#696）。** 多个范围的连接将源的章节保留在源的时间点，超出了输出的结尾。由新的时间守卫发现，而非报告。
- **任何改变长度并遗留章节的工具现在都会说明（#696）。** 通用结果写入器在另一长度的输出携带章节且其工具未声明时间映射时警告 `chapters_not_retimed`。

### 新增

- **每类修复三次的缺陷都有一个运行每个工具的守卫（#698）。** `tests/test_guards.py` 在一个 16:9、5.1、带章节、带字幕的片段上运行所有工具（包括多输入工具），stdin 保持打开，并在以下情况失败：章节留在源的时间点（time），丢失声道或画面大部分为黑边或被裁掉而无任何说明（claim），试运行报告了测量值，或记录评判了它未命名的文件（not_measured），声音在画面之前，或结果的第一行不是 `wrote`（surface），工具等待打开的 stdin（automation）。每一项都通过恢复旧缺陷进行了验证。`docs/defect-guards.json` 列出它们，`revert_proof.py` 拒绝修复一个有三次账本记录但无守卫的类别，因此下一个反复出现的类别在其第四次修复前会获得一个守卫。
- fix: 每类修复三次的缺陷都有一个运行每个工具的守卫（#700）

## 4.5.2

_自动发布：版本号和说明由自 4.5.1 以来合并的 Pull Request 生成。_

### 修复

- **保留画面不到一半的裁剪会被说明（#688）。** `fit.py`、`export.py` 和 `join.py` 使用 `--fit crop` 到另一形状的帧（16:9 片段到 9:16 保留 32%）现在警告 `picture_mostly_cropped` 并给出保留比例，如同自 4.3.4 起使画面变小的填充。文件不变。
- **章节跟随时间变化（#696）。** `pad.py --start`、`reverse.py`、`speedramp.py`、`freeze.py` 和 `fit.py --method speed|trim` 将源的章节留在源的时间点；现在它们随画面移动、镜像或缩放，裁剪到输出，无长度的章节被丢弃。MP4 和 MOV 仍从 0 开始其第一章（容器的章节轨道没有间隙）；Matroska 保留精确时间。
- fix: 保留画面不到一半的裁剪会被说明，章节跟随时间变化（#697）

## 4.5.1

_自动发布：版本号和说明由自 4.5.0 以来合并的 Pull Request 生成。_

### 修复

- **强制帧率会重新标记每个接受 `--fps` 的工具中的起始时间码（#470）。** `proxy.py`、`denoise.py`、`crop.py`、`redact.py`、`straighten.py` 和 `sphere.py --fps 50` 在起始于 01:00:00:12 的 25 fps 源上保留 01:00:00:12；现在在 50 下为 01:00:00:24，与 `fit.py` 已写入的一致。
- **`audio.py` 将画面保留为第一条流（#470）。** 保留画面时（`--voice`、`--music`、`--replace` 等），声音是流 0。
- **`broll.py --audio b|mix` 将 5.1 A-roll 保留为 5.1（#470）。** 它被折叠为立体声并标记 `verified: true`。单声道和立体声 A-roll 如以前一样在立体声中混合。
- fix: 强制帧率重新标记时间码，audio.py 将画面放在首位，broll 保留 5.1（#695）

## 4.5.0

_自动发布：版本号和说明由自 4.4.2 以来合并的 Pull Request 生成。_

### 新增

- **`wrote`：结果的第一行说明写入了什么以及在哪里有偏差。** 每个写入媒体文件的工具现在在其 `--json` 文档的第一行放置一行：`"wrote": "7.620 s (ASKED 7.500 s: 0.120 s longer), 1920x1080, 24.0 fps, picture and sound"`。落在关键帧上的流复制剪切说 `verified: true`，偏差在更下方的 `duration_error_ms` 中，读取顶部的调用者报告剪切为精确。
- feat: 结果的第一行说明写入了什么以及在哪里有偏差（#693）

## 4.4.2

_自动发布：版本号和说明由自 4.4.1 以来合并的 Pull Request 生成。_

### 修复

- **`font_used.substituted` 在未确定字体时为 `null`（#663）。** `caption.py --fonts-dir` 没有所请求名称的字体、`--ass`，以及按名称查找的字体族报告 `file: null` 旁边是 `substituted: false`，读作"未替换"。现在那里为 `null`：`false` 表示文件是所请求的字体族，`true` 表示是另一个。
- fix: font_used.substituted 在未确定字体时为 null（#674）

## 4.4.1

_自动发布：版本号和说明由自 4.4.0 以来合并的 Pull Request 生成。_

### 修复

- **以毫秒给出的位置在 `freeze.py`、`broll.py` 和 `speedramp.py` 中也是其帧（#653）。** 在 30000/1001 下，`freeze.py --at 1.902` 保持 1.9019 秒之后的那一帧，`broll.py --at 1.902` 使切出晚一帧开始，`speedramp.py` 结束于 1.902 的片段晚一帧改变速度。三者现在如 `cut.py` 自 4.3.6 起那样读取时间：当在半个毫秒内时取最近的帧。

### 变更

- **SKILL.md：一次普通编辑就是一次调用。** 工作流的八个步骤（探测、试运行、运行、检查、验证、查看）被读作每个请求的路径，模型将普通的剪切或调整大小走完所有步骤。现在有一段说明哪些请求需要一次带 `--json` 的调用，哪些需要这些步骤（交付、长时间或覆盖作业、改变画面显示内容的编辑）。九行路由合并到相邻行以保持在 220 行预算内；没有请求失去其行。
- fix: 以毫秒给出的位置在 freeze、broll 和 speedramp 中也是其帧（#672）
- docs(skill): 一次普通编辑就是一次调用（#670）

## 4.4.0

_自动发布：版本号和说明由自 4.3.6 以来合并的 Pull Request 生成。_

### 新增

- **`audio.py --mute`：一次调用移除声音。** 画面和字幕被复制；不重新编码。
- **`fit.py --fade-in S --fade-out S`：从黑场淡入和淡出到黑场，画面和声音一起。** 淡出在输出长度处结束，`--duration` 也是如此。`audio.py --fade-in/--fade-out` 仍是仅声音的淡变。
- **`export.py --duration S`：前 S 秒的交付**，用于"前 15 秒的 Reel"（`--preset reels --fit crop --duration 15`）。`--max-mb` 按较短长度计算大小。

### 变更

- **小于交付帧的 9:16 帧会被说明。** `fit.py --aspect 9:16` 在 1280x720 片段上保留源的大小并写入 406x720，如以前一样；结果现在会说明，并指出通常的 1080x1920 使用 `--width 1080`。SKILL.md 的"使其竖屏"行要求这样做。
- **放大使用 Lanczos。** 当 `fit.py` 或 `export.py` 预设写入大于画面的帧时，缩放使用 `-sws_flags lanczos`；源大小或更小的帧使用原来的命令。
- feat: 小的 9:16 帧会被说明，放大使用 Lanczos，静音、淡变和交付长度是一次调用（#669）

## 4.3.6

_自动发布：版本号和说明由自 4.3.5 以来合并的 Pull Request 生成。_

### 修复

- **Windows 上的 MCP：客户端保持管道打开时 `tools/call` 会得到应答（#664）。** 从 3.10.0 起，响应仅在客户端关闭 stdin 时写入：工具运行并写入其文件，但其进程继承了服务器的 stdin（客户端的 JSON-RPC 管道），在保持打开时未完成退出。