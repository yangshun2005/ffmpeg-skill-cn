# 任务：将 clip.mp4 裁剪为正方形，CRF 16

## 执行的命令（按顺序，原文照录）

```
python3 /home/user/ffmpeg-skill/scripts/probe.py clip.mp4 --json
```
关键输出：
```
"duration": 10.0, "video": {"width": 1280, "height": 720, "display_aspect": "16:9", "fps": 30.0, ...}, "audio": {"codec": "aac", "channels": 1}
```

```
python3 /home/user/ffmpeg-skill/scripts/fit.py clip.mp4 --aspect 1:1 --fit crop --quality 16 -o clip_square.mp4 --dry-run --json
```
关键输出：
```
ffmpeg -vf 'scale=720:720:force_original_aspect_ratio=increase,crop=720:720:(in_w-out_w)*0.5:(in_h-out_h)*0.5,setsar=1' ... -c:v libx264 -preset medium -crf 16 ... clip_square.mp4
[dry-run] would write clip_square.mp4 (0.000s, 0x0)
```

```
python3 /home/user/ffmpeg-skill/scripts/fit.py clip.mp4 --aspect 1:1 --fit crop --quality 16 -o clip_square.mp4 --json
```
关键输出：
```
wrote clip_square.mp4 (10.005s, 720x720)
"status": "completed", "verified": true,
"probe": {"duration": 10.005, "video": {"width": 720, "height": 720, "display_aspect": "1:1", "fps": 30.0, "codec": "h264"}, "audio": {"codec": "aac", "channels": 1}}
```

```
python3 /home/user/ffmpeg-skill/scripts/look.py clip_square.mp4 --tiles 3x2
```
关键输出：
```
contact sheet: 6 frames every 1.67s
wrote ./clip_square_sheet.png
```
查看 `clip_square_sheet.png`：源文件是合成测试图案（彩条、棋盘格、移动对角线），没有可识别的偏心主体，因此居中裁剪是正确的——没有裁掉任何有意义的内容。

## 说明
- 未运行 `check.py` 平台检查：用户没有给出交付目标（没有"用于 Reels/YouTube 等"），所以这是普通裁剪，不是导出——按该 skill 的工作流，无目标 + 普通编辑意味着保持源格式。
- 使用居中裁剪（默认）：源文件没有偏心主体，无需用 `--crop-x`/`--crop-y` 调整。
- 原始 `clip.mp4` 未改动；新文件写为 `clip_square.mp4`。

## 给用户的最终消息

已将 `clip.mp4` 用 CRF 16 裁剪为正方形，居中（画面中没有可见的偏心主体，无需偏移）。

- 输出：`/tmp/claude-0/ev24/q1/clip_square.mp4`
- 720x720，10.005 秒，30 fps，H.264（CRF 16），AAC 单声道——时长/fps/音频与源一致，仅画面从 1280x720 裁剪为居中的 720x720 正方形。
- 查看：`/tmp/claude-0/ev24/q1/clip_square_sheet.png`（缩略图拼版）——源文件是合成测试图案（彩条、棋盘格、移动线条），居中裁剪没有裁掉任何有意义的内容。
- 原始 `clip.mp4` 未被修改。