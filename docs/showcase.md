# ffmpeg-skill 展示：十二段视频，靠提问完成剪辑

**其中九段视频由 Google Veo 生成。** 里面的人物和地点并不存在。另外三段（`real_*.mp4`：夜晚的城市、夜晚的海岸、烟花）由维护者用手机拍摄。会议演示中的幻灯片是为此专门制作的，幻灯片上已注明。每一次剪辑都是调用该 skill 的某个脚本，每次调用都验证了它写出的内容。

在网站上观看：<https://kajisho5.github.io/ffmpeg-skill/#showcase>。会议演示、绿幕演示和集锦都有英文和日文版本；网站会根据当前显示的语言播放对应版本。

| 演示 | 请求 | 结果（`demos/out/showcase/`） |
|---|---|---|
| A. 雨夜 | “调亮，白平衡稍微偏冷，降噪，保留霓虹” | `rain/before_after.mp4` |
| B. 篮球 | “放慢镜头，剪出高光，做一个竖版” | `basketball/highlight.mp4`、`vertical_9x16.mp4` |
| C. 会议 | “一台机位拍演讲者，幻灯片是单独的：并排放在一起，再做一个带字幕的 Reels 短视频” | `talk_en/side_by_side.mp4`、`picture_in_picture.mp4`、`cutaway.mp4`、`social_short.mp4`（以及 `talk_ja/`） |
| D. 绿幕 | “去掉绿色，把她放进会议厅，缩小，放在右侧，加一个标题” | `greenscreen/composited_en.mp4` |
| E. 停顿 | “把长停顿剪掉” | `pause/tight.mp4`（7.85 秒剪到 5.9 秒） |
| F. 地平线 | “把地平线调平” | `horizon/straightened.mp4` |
| G. 手持（手机） | “去掉抖动” | `steady/stabilized.mp4` |
| H. 夜景（手机） | “太暗了：调亮并降噪” | `lowlight/clean.mp4` |
| I. 烟花（手机，60 fps） | “放慢” | `slow/slow_motion.mp4`（慢 2.5 倍，每一帧都是真实帧） |

`reel/showreel_en.mp4` 和 `showreel_ja.mp4`（50 秒）把它们串起来，并在首尾卡片上说明素材来源：AI 生成，以及三段手机拍摄。

## 构建

`python3 demos/showcase.py` 会运行全部流程，写出 `demos/out/showcase/` 以及网站用的副本 `site/media/showcase_*.mp4`（这里大约需要九分钟）。`python3 demos/showcase.py --web` 只重写网站用的副本。它只需要 FFmpeg：字幕从 `demos/footage/veo_talk_*.srt` 读取，幻灯片是 `demos/footage/slide_*.png`。

命令如下，其中 `SRC=demos/footage`：

```bash
# A. colour
python3 scripts/color.py $SRC/veo_rain.mp4 --correct --exposure 0.35 --gamma 1.08 --temperature 7000 -o white_balanced.mp4
python3 scripts/denoise.py white_balanced.mp4 --strength low -o denoised.mp4
python3 scripts/color.py denoised.mp4 --correct --contrast 1.06 --saturation 1.18 -o color_graded.mp4

# B. speed and reframe
python3 scripts/fit.py $SRC/veo_basketball.mp4 --duration 15.7 --method speed --smooth blend -o slow_motion.mp4   # 50%
python3 scripts/cut.py $SRC/veo_basketball.mp4 --start 1 --end 6.25 --accurate -o shot.mp4
python3 scripts/speedramp.py shot.mp4 --segment 0-2.2:1 --segment 2.2-4.2:0.5 --segment 4.2-5.25:1 -o highlight.mp4
python3 scripts/fit.py a.mp4 --aspect 9:16 --fit crop --crop-x 0.18 --width 1080 --height 1920 -o a916.mp4   # the player
python3 scripts/fit.py b.mp4 --aspect 9:16 --fit crop --crop-x 0.95 --width 1080 --height 1920 -o b916.mp4   # the hoop
python3 scripts/join.py a916.mp4 b916.mp4 --transition none -o vertical_9x16.mp4

# C. a talk and its slides
python3 scripts/loudness.py $SRC/veo_talk_en.mp4 --lufs=-14 -o speaker.mp4
python3 scripts/insert.py $SRC/slide_en_1.png --duration 4.3 --width 1920 --height 1080 --fps 24 -o slide1.mp4
python3 scripts/join.py slide1.mp4 slide2.mp4 --transition none -o slides.mp4
python3 scripts/fit.py speaker.mp4 --aspect 4:5 --fit crop --width 864 --height 1080 -o speaker_4x5.mp4
python3 scripts/background.py --gradient 0x101114:0x1a2a4a --angle 45 --width 1920 --height 1080 --fps 24 --duration 7.85 -o canvas.mp4
python3 scripts/overlay.py canvas.mp4 --video slides.mp4 --position 48,205 --margin 0 --scale 1192 -o pbp1.mp4
python3 scripts/overlay.py pbp1.mp4 --video speaker_4x5.mp4 --position 1288,205 --margin 0 --scale 584 -o pbp2.mp4
python3 scripts/audio.py pbp2.mp4 --replace speaker.mp4 -o side_by_side.mp4        # overlay.py keeps the main input's sound only
python3 scripts/overlay.py slides.mp4 --video speaker_4x5.mp4 --position bottom-right --margin 40 --scale 330 -o pip1.mp4
python3 scripts/broll.py speaker.mp4 --insert $SRC/veo_hall.mp4 --at 4.4 --duration 2 -o cutaway.mp4
python3 scripts/caption.py short3.mp4 --srt $SRC/veo_talk_en.srt --lang en --animate pop --karaoke --platform reels -o short4.mp4
python3 scripts/export.py short5.mp4 --preset reels -o social_short.mp4
python3 scripts/check.py social_short.mp4 --platform reels

# D. green screen
python3 scripts/overlay.py bg.mp4 --video $SRC/veo_greenscreen.mp4 --chromakey 0x50b64f --chromakey-similarity 0.1 --chromakey-blend 0.05 \
    --position 620,216 --margin 0 --scale-percent 80 -o keyed.mp4
python3 scripts/audio.py branded.mp4 --replace $SRC/veo_greenscreen.mp4 -o composited.mp4

# E. pause
python3 scripts/silence.py $SRC/veo_pause_en.mp4 --threshold -30 --margin 0.12 --min-silence 0.4 -o tight.mp4

# F. horizon
python3 scripts/straighten.py $SRC/veo_sea.mp4 --degrees 5 -o crooked.mp4      # the "before" is made here
python3 scripts/straighten.py crooked.mp4 --degrees=-5 -o straightened.mp4

# G. handheld, H. night shot, I. 60 fps played at 24
python3 scripts/stabilize.py $SRC/real_city.mp4 -o stabilized.mp4
python3 scripts/color.py $SRC/real_sea.mp4 --correct --exposure 1.2 --gamma 1.25 -o lifted.mp4
python3 scripts/denoise.py lifted.mp4 --strength high -o clean.mp4
python3 scripts/fit.py $SRC/real_fireworks.mp4 --duration 8.235 --method speed --fps 24 -o slow_motion.mp4
```

## 构建过程中发现的问题

- **抠像颜色从片段中读取，容差很窄。** 绿色是 `0x50b64f`，不是 `0x00ff00`。相似度 0.2 会把演讲者和屏幕一起抠掉；0.1 能保留她。
- **停顿需要更高的阈值。** 房间底噪接近 -32 dBFS，所以默认的 -35 dB 找不到静音。`--threshold -30` 能找到停顿。
- **`overlay.py --video` 只保留主输入的声音。** 承载人声的图层需要用 `audio.py --replace` 把声音找回来。
- **竖版篮球镜头是两个固定裁剪。** 没有主体跟踪：先拍球员，再拍篮筐，用硬切连接。前后对比用 `background.py` 和 `overlay.py --video` 把 16:9 画面和 9:16 画面放在同一画布上，各自保持自己的形状。
- **倾斜的地平线是这里做出来的。** 生成的海景片段是水平的，所以构建时先把它倾斜五度，再调平；网站已在视频下方注明。
- **响度。** 演讲测得 -17.6 LUFS，被提升到 -14。

- **抖动和暗光镜头是真实拍摄的。** 生成的“抖动”和“噪点”片段出来是稳定且干净的，所以这两个演示改用手机拍摄的素材。

## 未展示

语音降噪没有演示：没有一段视频的人声带噪点。