# 执行的命令

```
python3 /root/.claude/skills/ffmpeg-skill/scripts/probe.py iphone.mp4 --json
```
关键输出：
```
"video": {
  "codec": "hevc", "profile": "Main 10", "width": 640, "height": 360,
  "hdr": true, "hdr_signal": true, "bt2020_or_hdr": true,
  "hdr_format": "HDR10/PQ", "color_space": "bt2020nc",
  "color_primaries": "bt2020", "color_transfer": "smpte2084", "pix_fmt": "yuv420p10le"
},
"audio": null
```
确认 iphone.mp4 是 HDR10/PQ（BT.2020，SMPTE2084 传输特性）——这就是它在不支持 HDR 的笔记本屏幕上显得发白/暗淡的原因：SDR 显示器和大多数播放器渲染 PQ 标记的素材时，不会做必需的色调映射。

```
python3 /root/.claude/skills/ffmpeg-skill/scripts/color.py iphone.mp4 --to-sdr -o iphone_sdr.mp4 --json-brief
```
关键输出：
```
$ ffmpeg ... zscale=tin=smpte2084:pin=bt2020:...,tonemap=tonemap=hable:desat=0,... -c:v libx264 ... -colorspace bt709 -color_primaries bt709 -color_trc bt709 ... iphone_sdr.mp4
wrote iphone_sdr.mp4 (4.000s, 640x360, bt709/bt709, sdr)
{"status": "completed", "output": "iphone_sdr.mp4", "verified": true,
 "summary": {"duration_s": 4.0, "width": 640, "height": 360, "fps": 30.0, "vcodec": "h264"}}
```

```
python3 /root/.claude/skills/ffmpeg-skill/scripts/check.py iphone_sdr.mp4 --json-brief
```
关键输出（未指定目标平台，因此按 YouTube 的通用规范检查；未指定平台时，判断类检查项为 WARN 而非 FAIL）：
```
10 checks, 0 failed, 3 warnings
PASS duration, aspect, fps, vfr, video codec, colour (bt709/bt709), file size
WARN resolution 640x360 (source is only 640x360 — inherent to the original file, not something re-encoding can fix)
WARN subtitles none (optional, not requested)
WARN audio none (source iphone.mp4 has no audio stream at all — nothing to carry over)
```

```
python3 /root/.claude/skills/ffmpeg-skill/scripts/look.py iphone_sdr.mp4 --tiles 3x2
```
输出：`iphone_sdr_sheet.png`（已查看）。SDR 转换后，色条、棋盘格和移动线条渲染正常、色彩饱和度正确（不再发白/发灰）——修复生效。这是合成测试片段（色条 + 移动线条 + 棋盘格），不是真实拍摄素材，但 HDR->SDR 的机制与内容无关，同样适用。

# 给用户的最终消息

完成：iphone_sdr.mp4 — 4.00 秒，640x360，30 fps，H.264（bt709/bt709，SDR），无音频（源文件本就没有）
步骤：probe（发现 HDR10/PQ、BT.2020）-> color.py --to-sdr（色调映射到标准 SDR/Rec.709）
检查：youtube（假定，未指定平台）— 7/10 通过，3 个警告：分辨率 640x360 是源文件的原生尺寸（放大不会增加真实细节）、无字幕（未要求）、无音频（iphone.mp4 本身就没有音轨）
查看：iphone_sdr_sheet.png — 色彩现在渲染为正常的饱和色，不再是 HDR 素材在非 HDR 屏幕上那种暗淡/发灰的样子
说明：iphone.mp4 被标记为 HDR10/PQ（BT.2020 + SMPTE2084）——这正是它在你的笔记本上显得发白的原因：SDR 屏幕/播放器不会应用必需的 HDR 色调曲线。iphone_sdr.mp4 是标准 SDR H.264/Rec.709，因此在任何笔记本、手机或发布平台上都会显示正确且一致。如果你想针对特定目标（如 Reels、YouTube）调整尺寸/导出，告诉我目标平台，我会运行对应的模板。