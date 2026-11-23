# 示例

自然语言请求，以及 agent 为其运行的具体命令。路径假设该 skill 安装在 `~/.claude/skills/ffmpeg-skill`；可用 `S=~/.claude/skills/ffmpeg-skill/scripts` 简化。

运行 `bash examples/make_demo.sh` 可从零生成示例素材，并让它跑过 42 个脚本中的 13 个（输出落在 `examples/out/`；再次运行会替换它们）。

如需对带口播的素材做端到端剪辑，请参阅[带字幕的 Reels 完整演示](reels-walkthrough.md)。

---

### “我在处理什么素材？”
```bash
python3 $S/probe.py footage.mov --compact
python3 $S/probe.py footage.mov            # 完整 JSON，检查 variable_frame_rate_suspected
```

### “从 1:20 剪到 2:05”
```bash
python3 $S/cut.py footage.mov --start 1:20 --end 2:05             # 无损，对齐关键帧
python3 $S/cut.py footage.mov --start 1:20 --end 2:05 --accurate  # 帧精确
```

### “保留开头和结尾，去掉中间”
```bash
python3 $S/cut.py talk.mp4 --segments 0-1:30,18:40-20:00 -o talk_short.mp4
```

### “做成正好 60 秒”
```bash
python3 $S/fit.py talk_short.mp4 --duration 60 --method speed   # 加速/减速（保留音调）
python3 $S/fit.py talk_short.mp4 --duration 60 --method trim    # 保留前 60 秒
```

### “做一个竖版，用于 TikTok / Reels”
```bash
python3 $S/fit.py final.mp4 --aspect 9:16 --fit pad --width 1080 --pad-color black
python3 $S/fit.py final.mp4 --aspect 9:16 --fit crop --width 1080   # 改为居中裁剪
```

### “方形，用于 Instagram 信息流，15 秒”
```bash
python3 $S/fit.py final.mp4 --aspect 1:1 --fit crop --duration 15 --method trim --from-center
```

### “加上这些字幕”
`cues.txt`：
```
0:00-0:02.5 Welcome back
0:02.5-0:06 Today: three tips | for cleaner audio
Tip one: get the mic close
```
```bash
python3 $S/caption.py final.mp4 --text cues.txt --size 26 --bold --position bottom
python3 $S/caption.py final.mp4 --srt subtitles.srt --font "Inter" --color FFFFFF --outline 3
python3 $S/caption.py final.mp4 --ass styled.ass
python3 $S/caption.py --text cues.txt --write-srt cues.srt           # 仅生成 SRT，不输出视频
```

### “日文字幕”
```bash
fc-list | grep -i "cjk"      # 查找已安装的 CJK 字体
python3 $S/caption.py final.mp4 --srt jp.srt --font "Noto Sans CJK JP" --size 28
python3 $S/overlay.py final.mp4 --text "第1話" --font-file /usr/share/fonts/opentype/noto/NotoSansCJK-Bold.ttc --position top --box
```

### “Logo 放右上角，80% 不透明度，淡入淡出”
```bash
python3 $S/overlay.py final.mp4 --image logo.png --position top-right --scale 220 --opacity 0.8 --start 0.5 --end 58 --fade 0.5
```

### “前 4 秒加标题卡”
```bash
python3 $S/overlay.py final.mp4 --text "Episode 12 — Clean Audio" --position top-left --font-size 48 --box --start 0 --end 4 --fade 0.4
```

### “剪掉停顿”
```bash
python3 $S/silence.py talk.mp4 --list                          # 查看会剪掉哪些
python3 $S/silence.py talk.mp4 --threshold -40 --margin 0.2    # 渲染 talk_tight.mp4
python3 $S/silence.py talk.mp4 --edl keep.txt                  # 手动编辑 keep.txt，然后：
python3 $S/cut.py talk.mp4 --segments "$(paste -sd, keep.txt)" --accurate
```

### “把片段拼接起来，加交叉淡化”
```bash
python3 $S/join.py intro.mp4 main.mp4 outro.mp4 --transition fade --duration 0.5 -o final.mp4
python3 $S/join.py phone.mov camera.mp4 screen.mkv --transition fadeblack --width 1920 --height 1080 --fps 30
```

### “让我看看”（也让 agent 看看）
```bash
python3 $S/look.py final.mp4                          # final_sheet.png，12 格带时间码
python3 $S/look.py final.mp4 --at 4.5 --at 12         # 单帧
python3 $S/look.py before.mp4 --compare after.mp4 --at 4
```

### “告诉我你会先运行什么”
```bash
python3 $S/fit.py input.mp4 --duration 60 --method trim --aspect 9:16 --dry-run
python3 $S/export.py input.mp4 --preset reels --json          # 结构化结果，含对输出的 probe
```

### “做完整剪辑，并让我微调”
```bash
python3 $S/render.py --init project.json      # 填写 clips、captions、music、export、check
python3 $S/render.py project.json --dry-run   # 审阅命令计划
python3 $S/render.py project.json --fast      # 预览
python3 $S/render.py project.json             # 最终输出
```

### “处理整个文件夹”
```bash
cat > batch.json <<'EOF2'
{"glob": "*.mp4", "output_dir": "out", "suffix": "_yt",
 "steps": [["silence.py", "{in}", "-o", "{out}"], ["loudness.py", "{in}", "-o", "{out}"], ["export.py", "{in}", "--preset", "youtube", "-o", "{out}"]]}
EOF2
python3 $S/batch.py ~/Shoot --recipe batch.json            # 第二次运行：走缓存，只处理新文件
python3 $S/batch.py ~/Shoot --recipe batch.json --watch 60
```

### “转写并加字幕”（需要本地 whisper）
```bash
python3 $S/caption.py talk.mp4 --transcribe --language en --animate pop --karaoke
```

### “加下三分之一字幕条 / 标题 / 倒计时”
```bash
python3 $S/graphics.py talk.mp4 --template lower-third --name "Ada Lovelace" --title "Analyst" --start 2 --end 8 --brand brand.json
python3 $S/graphics.py talk.mp4 --template title --title "Episode 12" --subtitle "The math of video" --start 0 --end 4
python3 $S/graphics.py intro.mp4 --template countdown --from 5 --start 1 --end 6
python3 $S/graphics.py talk.mp4 --template progress
```

### “使用我们的品牌”
```bash
cp examples/brand.json ./brand.json          # 编辑字体、颜色、logo
python3 $S/caption.py talk.mp4 --text cues.txt --brand brand.json
python3 $S/overlay.py talk.mp4 --logo --brand brand.json
```

### “给我一份操作报告”
```bash
python3 $S/report.py --before raw.mov --after final.mp4 --platform youtube --commands commands.txt -o report.html
```

### “从一小时素材里做 60 秒精彩集锦”
```bash
python3 $S/scenes.py event.mp4 --highlights 6 --target 60 --edl picks.txt --sheet scenes.png
python3 $S/cut.py event.mp4 --segments "$(paste -sd, picks.txt)" --accurate -o digest.mp4
```

### “可以上传吗？”
```bash
python3 $S/check.py final.mp4 --platform reels
python3 $S/check.py spot.mov --platform broadcast      # EBU R128 -23 LUFS
```

### “三台摄像机加一台录音机——剪到一起”
```bash
python3 $S/multicam.py camA.mp4 camB.mp4 camC.mp4 zoom.wav --offsets-only
python3 $S/multicam.py camA.mp4 camB.mp4 camC.mp4 zoom.wav --audio 3 --fix-drift \
  --switch "0-45:0,45-80:1,80-95:2,95-140:0" -o edit.mp4
python3 $S/multicam.py camA.mp4 camB.mp4 --auto 10 -o rough.mp4     # 每 10 秒切换
```

### “先在我的真实素材上跑一遍”
```bash
python3 $S/verify.py ~/Footage/iphone.MOV ~/Footage/gopro.MP4 ~/Footage/obs.mkv --report verify.md
python3 $S/verify.py ~/Footage --quick
```

### “这是 Log 素材吗？”
```bash
python3 $S/probe.py a7s.mp4 --analyze --compact          # 画面平淡且低饱和时显示 [Log?]
```

### “iPhone HDR 素材看起来发灰”
```bash
python3 $S/probe.py IMG_0231.MOV --field video.hdr_format        # HDR10/PQ、HLG 等
python3 $S/color.py IMG_0231.MOV --to-sdr                          # 真正的色调映射到 BT.709
python3 $S/color.py IMG_0231.MOV --to-sdr --tonemap mobius --peak 1200
python3 $S/color.py IMG_0231.MOV --strip-dovi                      # 保留 HDR，去掉 Dolby Vision 层
```

### “应用 S-Log3 转换 LUT” / “给它这个风格”
```bash
python3 $S/color.py a7s.mp4 --lut SLog3SGamut3.CineToLC-709TypeA.cube
python3 $S/color.py final.mp4 --lut teal_orange.cube --lut-strength 0.6
```

### “清理音频并在下面加音乐”
```bash
python3 $S/audio.py talk.mp4 --voice                                     # 高通、去齿音、降噪、压缩
python3 $S/audio.py talk.mp4 --music bed.mp3 --duck --music-volume -16 --fade-out 3
python3 $S/audio.py surround.mov --downmix                               # 5.1 -> 立体声
python3 $S/audio.py clip.mp4 --replace narration.wav --fade-in 0.5
```

### “TikTok 风格字幕”
```bash
python3 $S/caption.py final.mp4 --text cues.txt --animate pop --karaoke --bold --size 30
python3 $S/caption.py final.mp4 --srt subs.srt --animate fade --position center
# 生成的 .ass 位于输出文件旁；修改后用 --ass 重新运行
```

### “平滑的半速慢动作”
```bash
python3 $S/fit.py shot.mp4 --duration 10 --smooth interpolate     # 5 秒片段 -> 流畅的 10 秒（渲染较慢）
python3 $S/fit.py shot.mp4 --duration 10 --smooth blend           # 更快的替代方案
```

### “把领夹麦同步到摄像机”
```bash
python3 $S/sync.py camera.mp4 lav.wav --json                       # 只告诉我偏移量
python3 $S/sync.py camera.mp4 lav.wav --replace-audio -o camera_synced.mp4
python3 $S/sync.py camera_synced.mp4 lav.wav --json                # 验证：偏移应约为 0（针对新文件的音频）
```

### “把 B 机对齐到 A 机”
```bash
python3 $S/sync.py camA.mp4 camB.mp4 --trim-second -o camB_aligned.mp4
```

### “这是一小时的录音，音频有漂移”
```bash
python3 $S/sync.py camera.mp4 zoom_h5.wav --fix-drift --json                 # 报告偏移 + 漂移 ppm
python3 $S/sync.py camera.mp4 zoom_h5.wav --fix-drift --replace-audio -o camera_synced.mp4
```

### “修正电平”
```bash
python3 $S/loudness.py final.mp4                    # -14 LUFS / -1 dBTP（YouTube、Spotify）
python3 $S/loudness.py podcast.wav -I -16 --tp -1.5 # Apple Podcasts
python3 $S/loudness.py final.mp4 --measure-only     # 仅报告
```

### “导出 YouTube 和 Reels 版本，外加 ProRes 母版”
```bash
python3 $S/export.py final.mp4 --preset youtube
python3 $S/export.py final.mp4 --preset reels --fit crop
python3 $S/export.py final.mp4 --preset prores -o final_master.mov
python3 $S/export.py final.mp4 --preset h265
python3 $S/export.py final.mp4 --preset gif -o preview.gif
```

### 一次跑完完整流水线
```bash
python3 $S/probe.py raw.mp4 --compact
python3 $S/cut.py raw.mp4 --segments 0:45-3:10,5:00-6:30 -o step1.mp4
python3 $S/fit.py step1.mp4 --duration 60 --method speed --aspect 9:16 --fit crop --width 1080 -o step2.mp4
python3 $S/caption.py step2.mp4 --text cues.txt -o step3.mp4
python3 $S/overlay.py step3.mp4 --image logo.png --position top-right --scale 180 -o step4.mp4
python3 $S/loudness.py step4.mp4 -o step5.mp4
python3 $S/export.py step5.mp4 --preset reels -o final_reels.mp4
python3 $S/probe.py final_reels.mp4 --compact     # 确认 60 秒、1080x1920、30 fps
```