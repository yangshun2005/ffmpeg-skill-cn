# 从源录像到带字幕的 Reels

在仓库根目录用 Bash 运行以下命令，需 Python 3 和 PATH 中的 FFmpeg。
内置源是一段 AI 生成的出镜口播片段。三条英文字幕使用其缓存的 faster-whisper 词级时间戳，并按本次剪辑分组为短语。
本演练需要 FFmpeg 的 subtitles 滤镜，以及已安装的英文字体。

```bash
set -e
OUT="$(mktemp -d)"
SOURCE="demos/footage/ai_talk_en.mp4"

python3 scripts/probe.py "$SOURCE" --compact
python3 scripts/cut.py "$SOURCE" --start 0 --end 9.4 --accurate \
  -o "$OUT/cut.mp4" --json > "$OUT/cut.json"
python3 scripts/fit.py "$OUT/cut.mp4" --aspect 9:16 --width 1080 --height 1920 \
  --fit blur --output "$OUT/vertical.mp4" --json > "$OUT/fit.json"
python3 scripts/caption.py "$OUT/vertical.mp4" --srt examples/reels-walkthrough.srt \
  --platform reels --language en -o "$OUT/captioned.mp4" --json > "$OUT/caption.json"
python3 scripts/export.py "$OUT/captioned.mp4" --preset reels --normalize \
  -o "$OUT/reels.mp4" --json > "$OUT/export.json"
python3 scripts/check.py "$OUT/reels.mp4" --platform reels --content \
  --json > "$OUT/check.json"
python3 scripts/look.py "$OUT/reels.mp4" --at 1.8 --safe reels -o "$OUT/caption-frame.png"
printf 'Delivery %s\nCheck report %s\nReview image %s\n' \
  "$OUT/reels.mp4" "$OUT/check.json" "$OUT/caption-frame.png"
```

剪辑从零开始，因此缓存的时间点仍对齐编辑后的时间轴。重构图在 1080×1920 画幅内以模糊填充保留完整源画面。字幕步骤在画幅确定后绘制字幕。Reels 预设会测量导出音频，必要时施加校正。打开 `check.json` 查看平台和内容行，包括警告。

示例运行输出 1080×1920、30 fps、约 9.43 秒，含三条烧录字幕，校正后测得 −14.0 LUFS。字幕流一行警告没有独立轨道。第一条字幕可在 `caption-frame.png` 中看到，该图对应用 UI 区域做了阴影标注以便检查。

若使用新源，在字幕命令中将 `--srt examples/reels-walkthrough.srt` 替换为 `--transcribe --write-srt "$OUT/captions.srt"`。先运行 `python3 scripts/_contract.py doctor --json` 检查本地语音引擎。引擎首次转写时可能下载模型权重。交付前请检查字幕并观看、试听成片。剪辑范围由调用方选择，字幕、构图和音频由调用方确认。