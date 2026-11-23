# 各平台的 CI 陷阱

macOS/Linux/Windows CI 运行器及其 ffmpeg 构建之间的行为差异——这些不是本仓库代码的 bug，每一个都曾独立诊断过一次，代价真实（完整 CI 周期、读日志、重试测试夹具）。写下来是为了下一个人（或下一次会话）不必从头重新诊断。每当本仓库的某个修复仅仅因为某平台真实观测到的行为而存在——而非假设的差异——就补充到本文件。

## Windows

### Chocolatey 的 ffmpeg 构建不支持 `-pattern_type glob`

Windows GitHub Actions 运行器上 `choco install ffmpeg` 安装的构建会失败，报 `Pattern type 'glob' was selected but globbing is not supported by this libavformat build`——glob 支持取决于 libavformat 的编译方式，而这个构建完全没有。`-pattern_type glob` 本身没有任何标志或变通办法。

`sequence.py` 中的修复：在 Python 中解析帧列表（`glob.glob` 或遍历连续编号的文件名），向 ffmpeg 传入显式的 **concat 解复用器列表文件**，而不是依赖 `-pattern_type glob`。这在三个操作系统上行为一致，因为它从不依赖 libavformat 自身的 globbing。

### concat 解复用器的"重复最后一个文件"时长技巧会多算一帧

给 concat 列表中最后一个文件指定时长的标准技巧（将其条目重复一次且不写显式 `duration` 行，让 ffmpeg 保持到 EOF）在某些 ffmpeg 构建上会多产生一帧的输出时长——例如本应总计 1.0s 的素材输出 1.2s。在 `sequence.py` 切换到 concat 解复用器后（见上），macOS 和 Windows CI 上都观测到了。

修复：在 concat 列表旁显式传入 `-t <total_duration>`，这样无论解复用器自身的结束标记在给定构建上如何表现，输出都会被截断到预期长度。

### `#!/bin/sh` 的假 ffmpeg PATH 垫片无法移植到 Windows

若干测试通过在 `PATH` 更靠前的位置放置一个名为 `ffmpeg` 的 `#!/bin/sh` 脚本来伪造 ffmpeg 的行为（例如"退出码 0 但不写任何东西"）。这在 Windows 上没有等价物——`cmd.exe`/PowerShell 不会像 POSIX shell 在 `PATH` 上解析 `ffmpeg` 那样执行名为 `ffmpeg` 的 shebang 脚本，因此假二进制会被静默忽略，测试要么因错误原因失败，要么实际运行了真正的 ffmpeg。

修复：对每个依赖此垫片技巧的测试加 `@unittest.skipIf(platform.system() == "Windows", "说明此点的原因")`，而不是试图让垫片本身跨平台。最先在 `test_dry_run_never_runs_ffmpeg_and_writes_nothing` 和 `DoctorDetectionTests` 类上确立；后来另一个未携带此上下文的改动添加 `test_output_verification_failures_are_loud` 时，又需要同样的模式。添加新的基于垫片的测试时，在测试文件中搜索此跳过模式并照搬，而不是在 Windows CI 运行中重新发现失败。

### 仅在 Windows 上，真实 ffmpeg 崩溃的 OS 报告退出码与本仓库自报的不一致

当 ffmpeg 真正崩溃时（例如解析损坏的 `.cube` LUT，或被要求写入不存在的目录），Windows 将子进程的退出码报告为一个大的无符号 32 位值（`4294967295`、`3199971767`、`4294967294` 等），与本仓库自身 `die()` 驱动的 JSON `exit_code` 字段为同一失败捕获的值不完全相等。通过重跑同一提交的同一作业两次得到逐字节相同的数字，确认不是偶发；该不匹配在不同失败子用例上以每次不同的具体数字复现。这是 Windows 通过 Python `subprocess` 报告崩溃子进程退出状态的固有方式——不是 `verify_output()` 或 `die()` 的 bug。

修复：对 `kind == "ffmpeg"` 失败断言 `exit_code` 的测试，在 Windows 上只断言 `!= 0`，在 macOS/Linux 上断言精确期望值。不要在这类失败上追求 Windows 的精确值一致——除非改变 OS 报告崩溃子进程的方式，否则无法实现，而这超出本仓库的控制范围。

## macOS

### `vidstabdetect`/`vidstabtransform`（libvidstab）在不同 ffmpeg 构建间行为显著不同

一个在 Linux CI 上经 `stabilize.py` 处理后可靠地变得*更不*抖动的合成相机抖动测试夹具，在 macOS CI 的 ffmpeg 构建上可靠地变得*更*抖动（测得的帧间运动更大，而非更小）——这在三种不同的夹具设计上独立确认（单一干净的正弦波抖动；包含约 9.1Hz 快速分量的多频抖动；调校到更真实的 1-2Hz 手抖范围的多频抖动），全部在 Linux 上通过，全部在 macOS 上以不同方式失败。这表明两个平台 ffmpeg 之间 libvidstab（或其构建/链接方式）存在真实的行为差异，而非测试夹具可修复的属性——夹具无法调校到同时满足两个互相矛盾的光流实现。

修复：`test_stabilize_reduces_frame_to_frame_motion` 仅在 Linux 上断言定量的"运动减少"主张（`if platform.system() == "Linux":`）；在每个平台上它仍断言工具运行了、产生了输出、输出具有预期时长——因此测试仍能捕获真正损坏的 `stabilize.py`，只是捕获不到超出本仓库控制的 libvidstab 行为怪癖。不要再花一个周期重新调校夹具频率——三次尝试已排除这种可能。

## 通用

当测试需要特判某平台时，优先用 `platform.system()` 门控（`tests/test_all.py` 和 `tests/test_contract.py` 中已为此导入），而不是发明新机制，并将跳过/放宽的原因写成完整句子解释底层平台行为——而不只是"在 Windows 上偶发"——这样未来的读者不必从 CI 日志中重新推导。

## FFmpeg 5.x（5.1.1 静态构建 CI 作业，已在 3.0.0 中移除）

历史记录：3.0.0 移除了 FFmpeg 5.x 支持（2.14.0 中已弃用）并随之移除了此 CI 作业，因此工具现在拒绝早于 6.1 的构建，其针对 `boxborderw` 的版本分支（见下）也已消失。保留作为 5.x 行为差异的记录。
在添加该作业当天发现（#146）；6.1+ 在三个 OS 上行为一致，因此这些此前从未出现过。

- **`scdet=...:sc_pass=1` 只传递分数超过阈值的帧。** 在 5.x 上，每个真正静止的帧（分数恰好为 0：标题卡、彩条）在下一个滤镜前被丢弃，帧号在不含它们的情况下重新计数。`scenes.py` 曾以 `threshold=0` 使用它并期望每帧都通过，因此一个 4 秒的 smptebars 场景使其两侧的剪切从其邻域测试中消失。6.1+ 无论分数如何都传递每一帧。已移除该选项；分数现在也按帧号索引，缺失帧计为 0。
- **`drawtext` 的 `boxborderw=v|h`（及四值形式）是 6.1+。** 5.x 和 6.0 会以 "Error setting option boxborderw to value 9|16" 使整个滤镜失败。在 2.x 期间，`_common.drawtext_boxborderw()` 在较旧构建上输出较大的单值（`_common.ffmpeg_version()` 解析一次 `ffmpeg -version`）；自 3.0.0 起它总是写 `v|h`。两个名称仍完全如此拼写：`_common` 在 1.15.0 之后的重构发布中成为包，其 `__init__.py` 重新导出单模块定义的每个名称，因此 `_common.drawtext_boxborderw()`、`_common.ffmpeg_version()` 和下面的 `_common.bt709_tag_args()` 通过门面原样解析。定义分别在 `_common/drawtext.py`、`_common/runner.py` 和 `_common/color.py` 中。
- **`showwaves` 在音频结束后仍继续输出帧，尽管有 `-shortest`。** 一个 12 秒的源在 5.1.1 上输出为 14.08 秒。`waveform.py` 现在也传入 `-t <source duration>`。
- **`-display_rotation` 是 6.0+。** 只有测试夹具构建器使用它（用于生成旋转的手机风格片段）；在 5.x 上它改为写入流的 `rotate` 标签，此处所有探测读取结果相同。
- **John Van Sickle 的 7.0.2 静态构建没有 `drawtext`**（用 freetype 构建，但滤镜缺失），且 BtbN 不再发布 7.x；因此 7.1 作业在 Debian trixie 容器中运行（apt ffmpeg 7.1.5，带 libass、freetype 和 zimg）。

## FFmpeg 7.1+（debian-trixie 容器 CI 作业）

- **输出 `-colorspace bt709` 不再只是标签：在未标记的源上它会转换。** 7.1 为 libavfilter 添加了色彩空间协商，CLI 将编码器的 `-colorspace/-color_primaries/-color_trc` 送入图的输出约束。一个比特流不携带色彩标签的源（`color_space=unknown`——测试源、屏幕录制、许多相机）随后与请求的 BT.709 *不同*，因此 ffmpeg 自动插入真实的矩阵转换（swscale 为"unknown"猜测 bt601）：每次通过 `x264_args()` 的 SDR 重编码都使画面偏移，`--lut-strength 0` 的空操作调色相对于其源回来约 24 dB PSNR。5.x/6.x 将相同选项仅写为标签（47 dB，无转换）。7.0.2 静态构建*不*显示此问题；7.1.1（conda-forge）在本地复现，因此当 trixie 作业在色彩测试上变红时，应使用该构建。
  修复：自 7.1 起，`_common.bt709_tag_args()` 通过编码器自身的 VUI 参数（`-x264-params colorprim=...:transfer=...:colormatrix=...`，x265 同理）写入标签，libavfilter 从不看到这些；较旧构建保留输出选项，因为那是当时获得 mp4 `colr` atom 的唯一方式。曾先尝试解码器侧的 `-colorspace bt709 ... -i` 覆盖并被否决：它还会给未标记源的 `-c copy` 输出打标签（导出副本必须保持真实副本），并且暴露了一个单独的、预先存在的 `--correct` bug（见下）。已在 5.1.1、6.1.1、7.1.1 和 8.1.2 上验证。8.x 有相同的编码时转换；在那里未被注意到是因为 8.x 的 `psnr` 滤镜*也*协商色彩空间并在测量前撤销了它（重新矩阵化的像素为 40 dB），然后反而将修复后的、字节干净的输出标记为 26 dB。测试的 `_psnr` 辅助函数现在在两个输入上固定相同的色彩标签，使每个构建对相同两个文件报告相同数字。
- **用 `scale=out_color_matrix=bt709` 绘制的测试夹具在 7.1+ 上不是未标记的。** scale 给其输出打标签，x264 写入该标签，因此这样制作的"未标记"片段在 trixie/macOS/Windows 作业上探测为 `color_space=bt709`，未标记用例在那里被静默地未测试（#333 的色彩夹具直到 2.10.3）。在编码前用 `setparams=colorspace=unknown` 重新标记帧，并断言探测结果为 unknown。
- **FFmpeg 8.0.1 和 conda-forge 的 7.1.1 在一个输入同时馈入 `-vf` 图和 `-af` 图且声音既延迟又填充时死锁**（`adelay` + `apad`，pad.py 使用的形态）：无 CPU、无进度，直到 `--timeout`。首次在 conda-forge 的 7.1.1 上发现并归档为该构建的怪癖；Ubuntu 26.04 的 8.0.1（`ubuntu-latest`）也有；Debian 的 7.1.5 和 8.1.3 没有。pad.py 将两个流放入一个 `-filter_complex`，在 5.1.1、6.1.1、7.0.2、8.0.1 和 8.1.3 上运行。延迟并填充音频的工具应做同样的事。
- **FFmpeg 8.0 和 8.0.1 的 `aselect` 传递每一帧**（其 `activate` 回调在滤镜转换中丢失；在 8.0.2/8.1 中修复）。没有失败：画面被剪切而声音没有。不要用 `aselect` 做移除：用 `atrim` 剪切每个范围（silence.py 通过 cut.py 的 `--segments --accurate` 路径重新剪切其范围；直到 3.1.3 它用 `ametadata=mode=add` 标记保留帧并用 `ametadata=mode=select` 保留它们）。
- **FFmpeg 8.0、8.0.1 和 8.1.0 在多行 `drawtext` 的每个换行处绘制字体的缺字形方框**（每行连同其换行符一起塑形；在 8.0.2/8.1.1 中修复）。`ffmpeg_version()` 无法区分 8.1.0 和 8.1.1；`drawtext_draws_newline_glyph()` 读取点发布版本。graphics.py 将此类标签路由到 libass；overlay.py 在 drawtext 运行测得的位置（`drawtext_line_metrics()`）每行绘制一个 drawtext，因为多行块自身的 `text_w` 携带了方框。
- **`-colorspace`/`-color_primaries`/`-color_trc` 在 7.1+ 上不是标签。** 它们馈入滤镜图的色彩空间协商：一个 bt709 标记的 H.264 用 `smpte170m` 选项重编码后，输出为转换后的矩阵和 bt709 原色/传输特性（8.0.1、8.1.3）。改用 `setparams`（或编码器自身的 VUI 参数）标记帧。在流复制时它们只改变容器：H.264/HEVC/MPEG-2/AV1/VP9 保留自己的描述，解码器优先读取它，因此用编解码器的 `*_metadata` 比特流滤镜重写它（`color.py --retag`）。
- **不是 7.1 的问题，在追查它时发现：`color --correct` 在 6.1 和 7.1 上以恒等设置将 bt709-*标记*的源去饱和约 8%**（113.6 → 103.9 saturation_avg）：RGB 阶段（exposure/colortemperature/colorbalance）使 swscale 用帧的 bt709 矩阵走 yuv→rgb，再用其 bt601 默认值走回来。恒等测试只使用未标记的源，两段都选 bt601 并相互抵消。单独跟踪。

### Windows：drawtext 按字体族名解析字体时崩溃（#100）

在某些 Windows ffmpeg 构建上（例如 winget 的 gyan.dev），`drawtext` 每当通过 fontconfig 按字体族名解析字体时就以访问冲突崩溃，即使 `fonts.conf` 有效（#100）。`look.py`、`scenes.py --sheet`、`grid.py` 的标签、`overlay.py --text` 和 `graphics.py` 在有可用字体时默认都解析具体字体文件（`fontfile=` 完全跳过 fontconfig，是确认不崩溃的形式），因此这应已自动处理。如果 drawtext 工具仍崩溃：`overlay.py` 和 `graphics.py` 接受 `--font-file`，显式传入一个，而不是依赖 `--font`/`font=` 解析；`look.py` 和 `scenes.py` 没有字体标志，`--no-timecode` 完全不绘制文本；`grid.py` 只有 `--font`（字体族名，即崩溃的形式），`--label none` 不绘制标签。`doctor` 还运行真实的单帧 drawtext 探测，并报告 `filter:drawtext` 缺失（崩溃详情在 `errors[]` 中），而不是仅凭 `-filters` 列表给出虚假的"可用"。