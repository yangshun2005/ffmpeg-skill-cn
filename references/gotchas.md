# 看似正确实则错误的问题与陷阱

SKILL.md 中"看似正确实则错误"和"陷阱"条目的完整版。SKILL.md 每项保留一行，并用下面的锚点指向此处；当工作涉及该领域时，这里的内容不是选读——它是细节，不是附注。

## 单行索引

下面每节开头的单行形式（SKILL.md 每项保留更短的一行）。

- HDR（iPhone、HDR10）经 SDR 路径重编码会变平；脚本保留 HDR；`hdr: true` 是真正的 PQ/HLG/DV 信号，`bt2020_or_hdr: true` 还包括 BT.2020 SDR。-> [#hdr-and-colour](#hdr-and-colour)
- Log 素材（S-Log/V-Log/C-Log）被标记为 SDR，看起来发灰：先 `probe.py --analyze`，再 `color.py --lut`。-> [#log-footage](#log-footage)
- 重编码写出不透明 H.264：带 alpha 的源需要 `--keep-alpha`（color/graphics/overlay），否则 `alpha` 步骤失败；`background.py --transparent` 是画布。-> [#alpha-channels](#alpha-channels)
- `-c copy` 剪切可能从错误/冻结帧开始；`cut.py` 在超过 0.5 秒吸附时重编码，请尊重它；.mp4/.mov 副本的编辑列表会隐藏预卷（`--no-edit-list` 关闭）。-> [#keyframe-cuts](#keyframe-cuts)
- VFR 手机/屏幕录制：重编码转为 CFR；`cut.py` 对实测 VFR 重编码（`--vfr-guard average`：2.x 规则）；帧率异常：`fit.py --fps`。-> [#variable-frame-rate](#variable-frame-rate)
- 同步/多机位 `confidence` 低于 0.3（或偏移巨大）可疑——检查每台摄像机；这些只对齐音频，从不做口型同步。-> [#sync-multicam-and-drift](#sync-multicam-and-drift)
- "已归一化"的音频仍可能削波（检查真峰值）；-40 LUFS 或以下的环境声绝不能提升到语音目标。-> [#loudness-and-ambience](#loudness-and-ambience)
- 在裁剪/缩放前烧录的字幕会落在画面外；先小尺寸烧录再由 `export.py` 放大则发虚。-> [#captions-fonts-and-text-order](#captions-fonts-and-text-order)
- Emoji 需要 `--emoji-assets DIR`（每个字形一个 PNG）才能彩色渲染；没有则输出单色，并会报告。-> [#emoji](#emoji)
- 非拉丁文本按文字系统选字体（`graphics.py` 通过 libass 塑形天城文、孟加拉文、泰米尔文、泰文；drawtext 不能）；无字体 = 任务失败。-> [#fonts-by-script](#fonts-by-script)
- `--fit crop` 16:9→9:16 丢弃 70% 宽度；按速度还是按裁剪得到"60 秒"是不同答案——说明是哪种及原因。-> [#reframing-fps-and-duration](#reframing-fps-and-duration)
- TikTok/Reels 用自身 UI 覆盖底部五分之一和右侧列——模板让文字避开这些区域；`look.py --safe tiktok` 显示它们。-> [#platform-safe-zones](#platform-safe-zones)
- `scenes.py --highlights` 按响度（或时长）排序，从不按含义：信任选择前先检查联系表。-> [#highlights](#highlights)
- CPU 重编码：x264 `medium`。-> [#chaining-and-speed](#chaining-and-speed)

## 色彩、HDR 与 Log

### HDR 与色彩
SDR 输出是标记为 BT.709 `yuv420p` 的 H.264。标记为 BT.601 矩阵（`smpte170m`、`bt470bg`；还有 `fcc`、`smpte240m`）的源，每次重编码都会把像素转换到该 BT.709 矩阵（直到 2.10.0 只改标签）；`color.py --retag bt709` 是流复制，仍只改标签。未标记的源按 BT.709 处理，不做改动。RGB 画面（PNG/JPEG 静帧、logo、emoji PNG、PNG 序列、qtrle 片段）显式转换为 BT.709 YUV（直到 2.10.1 由 FFmpeg 的 BT.601 默认完成），`grid.py` 按每个单元格自身的标签转换。工具绘制的颜色（填充和背景色、文字、方框、字幕、波形颜色）也以 BT.709 绘制；直到 2.10.2 它在 BT.709 标签下是 BT.601（纯红显示为 (255,24,0)）。当 `probe.py` 报告 `bt2020_or_hdr: true`（HDR10/PQ、HLG、Dolby Vision 或 BT.2020 SDR）时，每个编辑脚本都让输出保持在 10 位路径（HEVC Main10、源色彩标签），以免被静默压平。与用户决定：保留 HDR（适合 YouTube/手机）或先运行 `color.py --to-sdr` 用于仅 SDR 的目标、LUT 工作或 H.264 交付物。自 2.0 起 `hdr: true`（及其 1.9 的孪生 `hdr_signal`）是狭义事实——真正的 PQ / HLG / Dolby Vision 传输；`bt2020_or_hdr` 还包括广色域 SDR 文件（1.x 的 `hdr`），`hdr_format` 命名这个中间情况（`BT.2020 SDR`）。"非 HDR"不等于"可交付 BT.709"：BT.2020 SDR 文件仍带广色域原色，所以仅 SDR 的客户端或期望 BT.709 的平台需要先 `color.py --to-sdr`——`export.py` 只重标 BT.709，不转换（eval 24 h1）。`export.py` 平台预设是 SDR，对 HDR 输入会警告。
iPhone `.mov` 文件还带时间码/元数据轨道；编辑工具保留一条音频轨道（多数映射第一条；`cut.py` 取 ffmpeg 的默认选择，即声道最多的那条；`color.py --strip-dovi` 复制所有轨道），所以额外轨道被丢弃，流复制和重编码都一样（`cut.py` 会报告：`dropped_audio_streams`、`audio_tracks_dropped` 警告）。ProRes 母版保持源色彩：`export.py --preset prores` 不重标。

每次写入都会比较源还带了什么：`source_carry: {kept, dropped}` 列出字幕、章节、封面图、环绕声道、`creation_time`、音频语言标签和起始时间码（仅当它标记输出首帧时保留：从较后帧剪切会把它移到那里，多段拼接则不写），丢失的字幕轨道、章节列表、封面图或环绕布局是 `source_items_dropped` 警告。告诉用户文件完整前先读它；要保留这些项，用保留它们的工具（`export.py` 保留字幕轨道，`metadata.py` 写章节和标签）。

经 SDR 路径重编码 HDR（iPhone、HDR10）源会压平颜色。脚本保留 HDR；若手写 ffmpeg（永远不要——见 SKILL.md），不要在 BT.2020 像素上标 BT.709。

### Alpha 通道
ProRes 4444、QuickTime Animation、PNG 和 VP9 可带 alpha 通道；H.264——每个重编码工具的默认——不能。`color.py --correct/--lut`、`graphics.py` 和 `overlay.py` 接受 `--keep-alpha`（默认 ProRes 4444；`--alpha-codec qtrle|png|vp9`，各用能容纳它的容器）并在写入后读回 alpha。没有它，使用 alpha 的源输出为不透明，结果报 `verified: false` 和失败的 `alpha` 步骤（`alpha_dropped`）：仅当你要不透明输出时才传 `--drop-alpha`。要带透明度的逐元素图层，从 `background.py --transparent --duration 5 --width 1920 --height 1080 -o canvas.mov` 开始，用 `graphics.py`/`overlay.py --keep-alpha` 在上面绘制一个元素。alpha 线保留源的色彩标签（如 `--codec prores` 那样）；既非 HDR 也非 BT.2020 的 RGB 源（qtrle、PNG）写入 ProRes 4444 或 VP9 时变为 BT.709 YUV，标记 BT.709。`--alpha-mode premultiplied` 将写入颜色乘以 alpha；源按直通处理。带 alpha 的 VP9 图层用 libvpx-vp9 解码；FFmpeg 自带的 VP9 解码器丢弃 alpha。

### Log 素材
S-Log、V-Log 和 C-Log 看起来发灰、低对比，但被标记为 SDR。运行 `probe.py --analyze`；`looks_like_log: true` 表示先在做任何事之前用 `color.py --lut` 应用厂商 `.cube`。

## 剪切

### 关键帧剪切
无损 `cut.py` 结果可能比请求早最多一个 GOP（常 1–10 秒）开始（`start_snapped: true`）；偏差超过 0.5 秒时脚本自动重编码。单段 `.mp4`/`.m4v`/`.mov` 复制默认保留 MP4 编辑列表（自 3.0.0；`--no-edit-list` 让文件从关键帧开始）：回到关键帧的预卷被存储但隐藏，所以画面从请求时间开始（`edit_list`、`stored_preroll_seconds`、`start_snapped: false`）。忽略编辑列表的播放器或工具会显示该预卷。**结尾**仍落在包边界，长几帧，这些额外帧可能跳帧。`.mkv` 输出没有编辑列表，`--segments` 拼接每段从其关键帧显示。B 帧视频上，`.mp4`/`.mov` 的 `--segments` 段尾移到其处或之后的首个关键帧，容差相同（`segment_end_snap_seconds`）；无 B 帧和 `.mkv`/`.ts` 段不吸附。每次复制拼接都会检查（帧、步进、每段声音对画面），错误时重切。Open-GOP HEVC（iPhone "高效"）总是重编码 `--segments` 拼接，即使 `--tolerance -1`；B 帧 `.mp4`/`.mov` 拼接若后段在关键帧之间开始也一样（让它从关键帧开始，或写 `.mkv`）。若用户坚持无损输出，传 `--tolerance -1` 并告知剪切落在最近的较早关键帧。
复制从何处开始按 ffmpeg 的 seek 测量：B 帧 `.mp4`/`.mov` 上，关键帧前几帧的开始可能从该关键帧或早一个 GOP 开始（两种都 `start_snapped: true`）。`av_start_skew_seconds` 在复制的声音和画面分开开始时警告（Core Media HEVC 曾给出 3.7 秒声音无画面）；它只报告不修复：`--accurate` 是修复，或无损 `.mp4`/`.m4v`/`.mov` 用默认编辑列表（而非 `--no-edit-list`）。VFR 或非关键帧边界上的 `-c copy` 剪切产生"能用"但从冻结或错误帧开始的文件——尊重自动重编码，不要强推复制。

### 可变帧率
`r_frame_rate` 与 `avg_frame_rate` 不一致时（手机和屏幕录制），`probe.py` 设置 `variable_frame_rate_suspected`。该标志下 `crop`、`denoise`、`deinterlace`、`proxy`、`sphere`、`stabilize`、`straighten`、`redact`、`overlay`、`caption`、`color`、`graphics`、`pad`、`freeze`、`loop`、`broll`、`cut`（重编码、`--segments`）、`export`、`fit` 和 `speedramp` 用 `fps` 滤镜统一画面（每个槽位取最近帧；仅 `-fps_mode cfr` 会让一帧相对声音晚最多两帧，#512）；`join`、`multicam` 和 `grid` 用各自的 `fps` 滤镜把每个输入放到输出帧率。`reverse.py` 未改：其 `fps` 滤镜在 `reverse` 之后，`reverse` 按顺序把前向时间戳交给帧，所以反转的 VFR 画面保留其在前向时间中的间隙（27 fps 平均片段上每个闪光晚 0.19 秒；见 docs/design-decisions.md）。帧率取 `cfr_args()`：探测的平均值，在距整数 0.02 内时写为最近整数（29.98 统一为 30，29.97 保持 29.97），未探测到平均值时为 30。该标志是全文件平均，29.98 对名义 30 的手机会触发，所以 `cut.py` 改为在最多五个窗口采样包时间戳（`vfr_check`）判断：默认（`--vfr-guard sampled`，自 3.0.0）仅在不规则或太少无法判断时切到 `--accurate`；`--vfr-guard average` 恢复 2.x 仅凭标志重编码，`--vfr-guard off` 仍保留复制。平均值异常时（如丢帧导致 23.4 fps）用 `fit.py --fps 30|60` 显式选帧率。

## 音频

### 同步、多机位与漂移
同步或多机位对齐 `confidence` 低于 0.3，或偏移大于分析窗口的 60%，很可能错误：增大 `--analyze-seconds` 或找拍手。`multicam.py` 每台摄像机报一个 `confidence`——信任剪切前检查全部，而不只是命令成功。

`sync.py`/`multicam.py` 将音轨相互对齐，从不做口型同步（嘴动 vs. 音频）——本技能中没有任何人脸或嘴部检测。高置信度意味着音频匹配好，不是画面看起来对；口型同步是否正确需看视频，而非报告的偏移。

`multicam.py --switch energy` 切到最响的摄像机，是"谁在说话"的替代，不是说话人识别：每个说话人有自己摄像机麦克风时有效，共享房间麦克风或大声观众会选错摄像机。

不要在一个 `cut.py --segments` 拼接中混合不同帧率或采样率的文件而不重编码（`--accurate`）。`sync.py` 后，对输出再运行一次验证：偏移（及 `--fix-drift` 的漂移 ppm）应约 0。来自不同设备、长于约 10 分钟的录制：始终用 `--fix-drift`。

### 响度与环境声
"已归一化"的音频仍可能削波：检查真峰值，不只是 LUFS（`check.py` 两者都做）。不要把环境声或近静音归一化到语音目标——测得 -40 LUFS 或以下的片段是房间底噪、风声或什么都没有；提升 25 dB 提升的是噪声，不是内容。保持电平，说明情况，并提供音乐或旁白。

也不要在测量前加语音门：`loudnorm` 的 EBU R128 积分响度已应用 −70 LUFS 绝对门和 −10 LU 相对门，会丢弃与 `silencedetect` 遍历相同的安静块。语音跨度门已在仓库每个 fixture 上对照全文件测量，包括一个半数字静音的，结果最多移动 0.6 LU——在 `check.py` 自身 ±1 LU 容差内——代价是第二次完整解码。这就是 `loudness.py` 没有语音门标志的原因。

## 文字与构图

### 字幕、字体与文字顺序
在裁剪/缩放前烧录的字幕会落在画面外：先改帧，再文字。在中间尺寸烧录再由 `export.py` 放大的字幕发虚（1280x720 源适配 9:16 是 406x720，直到导出缩放到 1080x1920）——先适配交付尺寸（`fit.py --width 1080 --height 1920`），再字幕，再导出。

自 1.12 起字幕长度和时序已为你处理：`caption.py` 按所选 `--size` 的实测宽度把每条字幕换行到安全区，超过 `--max-lines`（默认 2）的字幕拆成连续字幕，短于 `--min-duration`（默认 1.0 秒）的字幕保持——绝不越过下一条——并按 `--offset SECONDS` 整体平移。它在一行 `cues:` 上报告改动，并把调整副本写在输出旁，绝不覆盖你传入的文件。换行需要输入视频（行宽来自其真实帧尺寸）；仅 `--write-srt` 时只应用时序标志。

Windows drawtext 在某些真实构建上崩溃（#100）：drawtext 工具默认解析具体 `--font-file`，可避免；若仍崩溃，显式传 `--font-file`。详情：`references/ci-platform-pitfalls.md`。

### 按文字系统选字体
libass 和 drawtext 对没有字形的字符各画一个空框，ffmpeg 仍退出 0——满是豆腐块的视频是经典的"它成功了"失败。自 1.12 起 `caption.py`、`graphics.py` 和 `overlay.py --text` 检测即将绘制文本的文字系统（日文