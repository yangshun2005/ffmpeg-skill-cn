# 脚本参考

每个脚本用 `--help` 打印相同的信息；本文件的存在是为了让 agent 一次读取多个脚本。所有脚本都接受 `--dry-run`、`--json`、`--json-brief`（自 1.11.0 起：相同的 JSON 结果，裁剪为 `status`、`output`、`dry_run`、`verified`、输出探测的紧凑 `summary`——duration_s、width、height、fps、vcodec、acodec、channels，以及测量时的 lufs——工具自身的键，以及命令数量而非命令行；它隐含 `--json`，不影响 `--json` 自身的输出，失败时打印通常的完整失败文档）、`--fast`、`--progress`、`--timeout SECONDS`、`--overwrite`、`--plan FILE`（将 dry run 写为计划文档，稍后由 `render.py FILE` 执行；见 render.py；`render.py` 和 `verify.py` 拒绝它）、`-o OUT`（`probe`、`check`、`cropdetect`、`scenes`、`batch` 和 `verify` 除外，它们不接受）；每个重编码工具和 `export.py` 还接受 `--hw`/`--no-hw`（h264/hevc/prores 在 Apple VideoToolbox 上，仅限 Apple Silicon，CRF 映射到 `-q:v`；AV1 仍在 CPU 上；VideoToolbox 自身拒绝的任务——编码器无法打开或其压缩会话失败——在 CPU 上重编码并报告在 `hw.fallback`/`hw.notes` 中，其他任何失败不重试；GPU 编码会添加一行 `notes` 说明其质量不等同于 CRF；`FFMPEG_SKILL_HW=1` 使其成为除 `export.py` 外所有工具的默认值，`export.py` 的交付预设需要 `--hw` 本身；`render.py`/`batch.py --hw`/`--no-hw` 应用于每个阶段，包括导出；结果会指明运行的 `encoder`）；每个重编码的编辑工具（不是 `export.py`，其预设决定编解码器）还接受 `--codec h264|hevc|av1|prores`（重编码的编码器；SDR 默认 x264，HDR 默认 x265 Main10，不变）和 `--quality N`（CRF 刻度，默认 18——`proxy.py` 为 30；av1 最高 63；prores 忽略）。2.0 从这些工具中移除了 `--crf`（它是 `--quality` 的别名）；`export.py` 保留自己的 `--crf`，它不是别名。每个工具都拒绝（`kind: input`，`error.reason: output_exists`，在任何运行之前）替换现有输出，除非给出 `--overwrite`。拒绝携带 `error.reason` 和 `error.details`（消息背后的事实；`contract --json` → `json_output.error_reasons`），而运行自行决定的内容——接管的过期输出锁、无法加锁、无人指纹的计划输入、缩放为偶数的奇数尺寸帧、重试时丢弃的字幕流（带有第一次尝试的错误）、替换的字体、在 `--dry-run` 下跳过的测量、无法读取的编码器列表、无效的 `FFMPEG_SKILL_TIMEOUT`，以及每一行 `warning:`——都在 `diagnostics: {warnings, fallbacks, skipped}` 中，也在 stderr 上（代码：`contract --json` → `json_output.diagnostic_codes`）。`--codec av1`/`prores` 没有对应编码器时拒绝 `kind: missing_tool`，`error.reason: encoder_missing`；当编码器列表完全无法读取时，`kind: ffmpeg`，`encoder_list_unreadable`。绘制文本的工具（`overlay.py --text`、`graphics.py`、`caption.py`、`grid.py --label auto`）报告 `font_used: {asked, file, family, how, substituted}`。SDR 上的 `--codec hevc` 写入 8 位 BT.709 HEVC（`hvc1`），`av1` 使用 SVT-AV1（libaom 回退），`prores` 是 422 HQ 且需要显式的 `-o NAME.mov`（或 `.mkv`），`h264` 拒绝 HDR 源（`kind: input`，先运行 `color.py --to-sdr`）。`export.py` 保持从其预设选择编解码器，既没有 `--codec` 也没有 `--quality`；`render.py` 项目也不能选择编解码器——但 `--dry-run` 只保证写入工具不写入任何内容：`probe`（只读，`--dry-run` 不改变任何东西）仍运行 ffprobe，使用 `--analyze` 时运行 ffmpeg 采样画面电平，`check`/`sync`/`multicam`/`scenes`/`cropdetect`/`report`/`silence`/`loudness`/`stabilize` 仍运行其 ffmpeg/ffprobe 测量（dry-run 计划基于真实数字；它们只是不写入最终产物），`overlay` 在 FFmpeg 8.0、8.0.1 和 8.1.0 上测量多行 `--text` 的行（在生成的帧上运行一次 drawtext），带 `--auto-chapters` 的 `metadata` 和带 `--snap beats` 的 `cut` 解码输入以进行测量，`render` 和 `batch` 将每个阶段或步骤作为自己的 dry run 运行（因此测量阶段测量存在的输入），`verify` 接受该标志但完全忽略它（其文档说 `dry_run: false`，并有注释说明）。每个工具的精确语义：`contract --json` 的 `dry_run` 字段（或 `docs/contract.md`）。

## 文件名和不可用输入（每个工具）

名称包含 `%` 的静态图片（`a%20b.png`、`logo%d.png`）被读取为该单个文件，绝不作为编号序列。给 `--srt`、`--ass`、`--lut`、`--font-file` 或 `--fonts-dir` 的带冒号的相对路径按原样工作（`./a:b.srt`、`a:b.srt`）。名称以 `-` 开头的输入被读取为文件：在 `--` 之后传递或作为 `./-name.mp4`；以 `-` 开头的输出被拒绝（`output_looks_like_option`；传递 `./NAME`）。对于主媒体输入，在 ffprobe 运行之前：`kind: input` 带提示、`input_is_directory`、`input_empty`（0 字节）或 `input_permission_denied`（文件或通往它的目录不可读；不存在的路径是 "input not found"）；`input_not_media` 在之后出现，当 ffprobe 无法读取它时。附属文件（`--lut`、`--srt`、字体）由其自己的标志检查，工具自己的预检（`render.py` 的项目和模板输入、`join.py --list`）在此之前运行；两者可能以不同方式表述同一问题，而不带这些原因。对于输出，在 ffmpeg 运行之前：`output_extension_missing` 和 `output_extension_unknown`（已安装的 FFmpeg 没有对应复用器的扩展名，如 `out.xyz`；用 `-f` 命名其格式的命令豁免）。

## 时间语法（每个接受时间的标志，1.9）

一个解析器 `time_arg()`，位于每个 `--start`、`--end`、`--at`、`--from`、`--duration`、`--segments`、cue 文件和项目字段之后：秒（`12.5`）、`mm:ss(.fff)`（`1:30`）、`hh:mm:ss(.fff)`（`00:01:30.250`；逗号也可用，如 SRT 中）。四部分 `hh:mm:ss:ff` 是源帧率的 SMPTE 非丢帧时间码；附加 `@fps`（`00:01:02:15@29.97`）自行命名速率，这是没有输入文件的工具（没有视频的 `caption.py --text`，以及那里的 `--fps`）的唯一方式。任何地方都没有 fps 的四部分值是 `kind: input` 并命名该标志。工具之间关于时间的其他方面没有差异。

## 判定阈值（什么通过）

`verified`、`check.py` 的 PASS/WARN/FAIL 行以及工具所做剪切背后的数字。结果还在每个 `verification[]` 步骤中携带其使用的容差（`tolerance`，自 2.18 起）。每一行都命名其来源的常量或标志，测试将表固定到它（`test_the_thresholds_table_matches_the_code`）。

<!-- thresholds:start -->
| 什么 | 值 | 来源 | 使用者 |
|---|---|---|---|
| 真峰值可超过 dBTP 上限 | 0.1 dB | `_platforms.TP_TOLERANCE_DB` | `loudness.py`、`check.py`、`export.py` |
| 归一化文件在其 `-I` 目标内 | 1.0 LU | `loudness.LUFS_TOLERANCE` | `loudness.py` |
| 电平处理保持声音长度至（或一个 AAC 帧，1024 采样，如果更长） | 0.03 s | `loudness.AUDIO_LENGTH_TOLERANCE` | `loudness.py` |
| 拼接画面与剪辑减去转场一样长，至（帧） | 1.5 | `join.DURATION_TOLERANCE_FRAMES` | `join.py` |
| ... 且至少 | 0.05 s | `join.DURATION_TOLERANCE_MIN` | `join.py` |
| 画面和声音一起结束，超出输入原有的间隙，至 | 0.25 s | `_common.emit.STREAM_LENGTH_TOLERANCE` | `cut.py`、`join.py`、`export.py`（`stream_lengths`） |
| 每个保留的流在其第一个和最后一个 | 1.0 s | `_common.emit.DECODE_WINDOW_SECONDS` | `cut.py`、`join.py`、`export.py`（`decode`） |
| 无损剪切在重编码前可偏离请求时间 | 0.5 s | `cut.py --tolerance` | `cut.py` |
| 静音是声音在或低于 | -35 dBFS | `silence.py --threshold` | `silence.py` |
| ... 持续至少 | 0.6 s | `silence.py --min-silence` | `silence.py` |
| 移除的静音是数字静音，当其中 99% 在或低于 | -90 dBFS | `silence.DIGITAL_SILENCE_DB` | `silence.py`（`silence_kinds`） |
| 当剪切周围的电平在此范围内接近 `--threshold` 时标记剪切 | 10 dB | `silence.CUT_SOUND_MARGIN_DB` | `silence.py`（`cut_warnings`） |
| 当轨道峰值为或低于 | -50 dBFS | `_common.probe.SILENT_PEAK_DBFS` | `check.py`；`join.py`、`audio.py`、`waveform.py` 的 `--silence-threshold` 默认值 |
| 长于此的填充词是保持的元音 | 1.2 s | `_common.decision.FILLER_MAX_WORD` | `silence.py --filler` |
| 无空格脚本中两个 token 之间此长度的停顿是词边界 | 0.1 s | `_common.decision.FILLER_BOUNDARY_GAP` | `silence.py --filler` |
| `--content` 黑场：WARN 高于此时长占比 | 0.10 | `check.BLACK_WARN` | `check.py --content` |
| `--content` 黑场：FAIL 在此占比 | 0.95 | `check.BLACK_FAIL` | `check.py --content` |
| `--content` 黑场：像素低于亮度范围此占比为黑 | 0.10 | `_common.probe.CONTENT_BLACK_PIX_TH` | `check.py --content` |
| `--content` 黑场：短于此的黑场段不计 | 0.1 s | `_common.probe.CONTENT_MIN_SPAN` | `check.py --content` |
| `--content` 冻结：最长跨度高于此，或高于 30% 时 WARN | 3.0 s | `check.FROZEN_WARN_S` | `check.py --content` |
| `--content` 冻结：一个跨度覆盖此占比时 FAIL | 0.95 | `check.FROZEN_FAIL` | `check.py --content` |
| `--content` 冻结：画面至少此长度不变为冻结 | 0.5 s | `_common.probe.CONTENT_FREEZE_MIN` | `check.py --content` |
| `--content` 冻结：此接近的帧计为不变（freezedetect 噪声） | -60 dB | `_common.probe.CONTENT_FREEZE_NOISE` | `check.py --content` |
| `--content` 静音：低于静音电平的此占比以上 WARN | 0.50 | `check.SILENCE_WARN` | `check.py --content` |
| `--content` 静音：此占比 FAIL | 0.95 | `check.SILENCE_FAIL` | `check.py --content` |
| `--content` 静音：短于此的静音段不计 | 0.5 s | `_common.probe.CONTENT_SILENCE_MIN` | `check.py --content` |
| `--content` 丢音：声音间隙在或低于 | -80 dBFS | `_common.probe.CONTENT_DROPOUT_DB` | `check.py --content` |
| ... 在（秒）和 1 秒长之间，两侧有声音 | 0.02 s | `_common.probe.CONTENT_DROPOUT_MIN` | `check.py --content` |
| `--levels`：当一帧有超过此占比的亮度（或色度）采样超出合法范围时 FAIL | 1.0 % | `check.py --levels-tolerance` | `check.py --levels` |
| `--legalize`：写入文件可保留此占比的一帧亮度（或色度）采样超出范围（步骤 `legal_levels`） | 1.0 % | `color.LEGALIZE_TOLERANCE_PCT` | `color.py --legalize` |
| 低于此置信度的节拍网格不吸附 | 0.5 | `_common.decision.BEAT_MIN_CONFIDENCE` | `cut.py --snap beats`、`scenes.py --beats` |
<!-- thresholds:end -->

每个平台的交付目标（`scripts/_platforms.py`）。文件在其目标的平台容差内通过响度，真峰值在或低于其上限加 0.1 dB。最大长度（以及 `check.py` 从同一表读取的文件大小）是上传限制：表的截至 `_platforms.UPLOAD_LIMITS_AS_OF`，是指导而非要求，因为平台会更改它，上传者检查当前的。`check.py` 超过时 WARN；`export.py` 裁剪到它，除非 `--allow-long`，并无容差地将写入长度保持到它。

<!-- platforms:start -->
| 平台 | 响度 | 容差 | 真峰值 | 最大时长 |
|---|---|---|---|---|
| `tiktok` | -14 LUFS | ±2 LU | -1 dBTP | 600 s |
| `reels` | -14 LUFS | ±2 LU | -1 dBTP | 90 s |
| `shorts` | -14 LUFS | ±2 LU | -1 dBTP | 180 s |
| `youtube` | -14 LUFS | ±2 LU | -1 dBTP | 43200 s |
| `youtube-hdr` | -14 LUFS | ±2 LU | -1 dBTP | 43200 s |
| `youtube-av1` | -14 LUFS | ±2 LU | -1 dBTP | 43200 s |
| `x` | -14 LUFS | ±3 LU | -1 dBTP | 140 s |
| `linkedin` | -14 LUFS | ±3 LU | -1 dBTP | 600 s |
| `facebook` | -14 LUFS | ±3 LU | -1 dBTP | 14400 s |
| `podcast` | -16 LUFS | ±1 LU | -1 dBTP | — |
| `broadcast` | -23 LUFS | ±1 LU | -1 dBTP | — |
| `custom` | — | ±2 LU | -1 dBTP | — |
<!-- platforms:end -->

## 目录
- 判定阈值 — 什么通过，以及每个数字在哪里
- probe.py — 检查
- cut.py — 剪切 / 拼接片段
- fit.py — 目标时长和/或宽高比，旋转/翻转
- crop.py — 裁剪到精确像素矩形
- insert.py — 静态图片到定时静音片段，带 Ken Burns 缩放/平移
- broll.py — 在窗口内切到 B-roll 片段再回来
- metadata.py — 章节标记和标题/艺术家/评论标签，流复制
- background.py — 生成纯色或渐变片段
- reverse.py — 反向播放
- stabilize.py — 运动稳定（vidstab）
- sequence.py — 编号/通配图像序列到视频
- silence.py — 移除死气 / 跳切
- join.py — 带转场拼接
- render.py — 一个 project.json 中的整个编辑
- 交付模板 — 每个目的地一个命令（`--template`）
-