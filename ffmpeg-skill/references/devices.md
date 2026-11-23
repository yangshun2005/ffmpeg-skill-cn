# 设备与格式说明（来自真实设备语料库）

`probe.py` 会显示什么，以及该如何处理。这些经验来自在真实文件上运行工具链，而非规格说明书。

| 来源 | 你会看到什么 | 对编辑意味着什么 |
|---|---|---|
| iPhone（iOS 15+）HDR 视频 | `hdr_format: Dolby Vision profile 8`、HLG 传输、10 位 HEVC、`variable_frame_rate_suspected: true`、竖屏时 `rotation` 为 -90/90、额外的时间码/元数据轨道 | 每次重新编码都会自动保留 HDR（HEVC Main10 + 标签）。仅当目标只支持 SDR（X、LinkedIn、大多数网页播放器）时，才先用 `color.py --to-sdr`。只使用第一条音轨。当采样帧时序可变时，无损剪辑会回退为精确剪辑（`cut.py` 的 `vfr_check`）。 |
| iPhone SDR / 旧机型 | HEVC 或 H.264 4K60、单声道 AAC | 单声道音频：若交付物要求立体声，在配乐/闪避前先运行 `audio.py --stereo`。 |
| GoPro HERO | HEVC 4K 10 位 **SDR**（`hdr: false`、`yuv420p10le`）、立体声、GPMF 数据轨道 | 10 位不代表 HDR；不要做色调映射。重新编码为 8 位 H.264（自 3.0.0 起，`cut.py` 的重新编码保持 8 位 HEVC），除非用户想要 10 位母版（`export.py --preset prores`）。 |
| DJI 无人机 | HEVC 4K 50/60、**无音频** | 依赖音频的步骤（同步、静音、响度、闪避）不适用；用 `audio.py --replace/--music` 添加旁白或音乐。D-Log 配置文件看起来发灰：`probe.py --analyze` -> `looks_like_log` -> `color.py --lut`。 |
| Android 录屏 | H.264、非标准尺寸（720x1600、1298x1080）、18-120 fps、强 VFR | 先运行 `fit.py --fps 30`（或 60）统一帧率，再做其他处理；用 `check.py` 检查宽高比。 |
| Zoom / Teams 录制 | H.264 720p-1080p、低码率、时长长、通常为 VFR | `silence.py` 和 `scenes.py` 可用；处理一小时的文​​件会很慢；预览用 `--fast`。 |
| OBS（mkv） | H.264/HEVC、有时有多条音轨、源丢帧时为 VFR | 只使用第一条音轨；若客户需要，先重封装为 mp4（`export.py`）。 |
| HDR10 母版 / 测试图 | PQ（`smpte2084`）、BT.2020、MaxCLL/MDL 元数据、10 位 | `color.py --to-sdr --tonemap hable`（`mobius` 保留更多高光细节）；4K 色调映射约 1 倍实时速度，所以先剪辑。 |
| 广播 / ProRes .mov | ProRes 422/4444、PCM 音频、24p 或 25p | 尽可能流复制；母版用 `export.py --preset prores`；`check.py --platform broadcast`（EBU R128 -23 LUFS）。 |
| 电影 / 网页下载 | H.264 配 MP3 或 AAC、24p、带黑边（1280x534） | `fit.py --aspect` 填充；`check.py` 标记非标准宽高比。 |

由此得出的经验法则：

- 10 位是容器属性，HDR 是色彩属性。相信 `hdr`，而非 `bit_depth`。
- 来自手机或录屏设备的文件，除非证明否则都是 VFR；每个重新编码脚本都会统一帧率，但要规划精确剪辑。
- 在笔记本级 CPU 上，4K 10 位重新编码每 6 秒素材耗时 20-25 秒：先剪辑，再处理，迭代时用 `--fast`。
- 超过 10 分钟的文件：绝不要对整个文件做色调映射或插帧；在剪辑片段上处理。