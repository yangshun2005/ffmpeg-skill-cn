# 演示素材

`ai_talk.mp4` 是 AI 生成的视频（一个并不存在的、坐在笔记本电脑前的人），6 秒，640×430，由维护者提供给演示使用。`reframe_talk` 演示使用它。它不属于 npm 包的一部分。

`ai_talk_en.mp4` 同样是 AI 生成的（一个并不存在的、在户外的男人），15 秒，854×480，英语讲话，含两处停顿犹豫。`ai_talk_en.words.json` 保存其词级时间戳，由 faster-whisper（base 模型，按 `silence.py --filler --transcribe` 的方式用填充词列表进行引导）生成，因此 `filler_en` 和 `pauses_en` 两个演示无需语音引擎即可构建，二者都读取此片段。

这两个片段是 `demos/build.py` 中仅有的非合成素材：三个演示（`reframe_talk`、`filler_en`、`pauses_en`）使用它们，其余所有演示均由 FFmpeg 自带的测试源生成。

## 展示片段（由 Google Veo AI 生成）

`veo_rain.mp4`、`veo_basketball.mp4`、`veo_greenscreen.mp4`、`veo_talk_en.mp4`、`veo_talk_ja.mp4`、`veo_pause_en.mp4`、`veo_sea.mp4`、`veo_hands.mp4` 和 `veo_hall.mp4` 是用 Google Veo 制作的 AI 生成视频，由维护者提供。其中的人物和地点均不存在。每个片段为 7.85 秒，1920×1080，24 fps，HEVC，带声音；它们是生成文件的重新编码副本（`export.py --preset h265`），以保持仓库体积较小。`demos/showcase.py` 对它们进行剪辑（[docs/showcase.md](../../docs/showcase.md)）；它构建的合集在第一张和最后一张卡片上注明素材为 AI 生成。`veo_talk_en.srt` 和 `veo_talk_ja.srt` 保存演讲的字幕提示，由 `caption.py --transcribe` 生成，并根据实际讲话内容进行了校正。`slide_en_*.png` 和 `slide_ja_*.png` 是会议演示的幻灯片，专为此制作。

`real_city.mp4`、`real_sea.mp4` 和 `real_fireworks.mp4` 并非生成：由维护者用手机拍摄（夜晚的城市，手持；夜晚的海岸；60 fps 的烟花，从 4K 缩放到 1080p）。它们是去除了相机元数据的重新编码副本。