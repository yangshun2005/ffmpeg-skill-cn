# 请求 → 脚本，详细版

SKILL.md 请求表中的每一行，包含其替代方案、注意事项和额外参数。SKILL.md 每行只保留一条命令；当某行指向此处，或某个请求需要表中未展示的变体时，打开本文件。

| 用户说 | 执行 |
|-----------|----|
| "从 1:20 剪到 2:05"，"剪掉前 10 秒" | `cut.py input.mp4 --start 1:20 --end 2:05` |
| "只保留这些部分"，"去掉中间" | `cut.py input.mp4 --segments 0-1:00,1:30-2:00` 或 `--remove 1:00-1:30` |
| "做成正好 60 秒" | 先决定剪什么。简单裁剪用 `fit.py in.mp4 --duration 60 --method trim`。 |
| "剪这段并输出 HEVC / AV1 / ProRes"（指定输出编码） | `cut.py input.mp4 --start 0:10 --end 0:40 --codec hevc`（ProRes 需 `-o NAME.mov`） |
| "剪掉停顿"，"跳剪" | `silence.py input.mp4 [--threshold -40 --min-silence 0.8]` |
| "剪掉嗯嗯啊啊"，"去掉填充词" | `silence.py input.mp4 --filler --words words.json`（使用实测词时间戳；`--transcribe` 可生成） |
| "不要在句子中间剪，只剪真正的停顿" | `silence.py input.mp4 --speech-aware` — 句子内部短于 `--min-silence` 的换气保留，只剪句子边界处的停顿；可与 `--filler` 合并为一个列表 |
| "从这一小时里剪 60 秒高光" | `scenes.py long.mp4 --highlights 6 --target 60 --edl picks.txt` → `cut.py --segments` |
| "踩点剪"，"跟着音乐剪" | `scenes.py track.mp4 --beats --json > beats.json`，然后 `cut.py input.mp4 --segments ... --snap beats --snap-source beats.json`（`--snap-source` 传入实测节拍网格） |
| "这里加速，那里慢放"（已知片段） | `speedramp.py action.mp4 --segment 0-3:1.0 --segment 3-4:0.25 --segment 4-8:2.0` |
| "定格这一帧"，"冻结最后一帧" | `freeze.py clip.mp4 --hold 2` |
| "平滑慢动作"，"半速但流畅" | 10 秒片段用 `fit.py input.mp4 --duration 20 --smooth interpolate` 或 `--smooth blend` |
| "倒放这段" | `reverse.py input.mp4` |
| "循环这段填满 30 秒" | `loop.py bg_loop.mp4 --duration 30` |
| "做成竖屏 / 9:16 / 方形" | `fit.py input.mp4 --aspect 9:16 --fit pad`（或 `--fit crop`；`--pad-fill blur` 用模糊边框） |
| "调整到指定高度/宽度" | `fit.py input.mp4 --height 1080`（或 `--width`，或两者同时指定精确尺寸） |
| "裁剪到这个精确区域"（已知 x/y/w/h） | `crop.py input.mp4 --x 100 --y 0 --width 1080 --height 1920` |
| "用模糊背景代替黑边" | `fit.py input.mp4 --aspect 9:16 --fit blur`（保留完整画面，边框为模糊变暗的副本） |
| "旋转 90 度"，"镜像" | `fit.py input.mp4 --rotate 90` / `fit.py input.mp4 --flip h` |
| "地平线歪了"（已知角度） | `straighten.py tilted.mp4 --degrees -2.5` |
| "从这个 360 视频取平面视角"（已知 yaw/pitch/fov） | `sphere.py insta360.mp4 --yaw 90 --pitch 0 --h-fov 100 --v-fov 70` |
| "这段老素材是隔行的" | `deinterlace.py input.mp4` |
| "有噪点/颗粒，清理一下" | `denoise.py input.mp4 --strength medium` |
| "模糊/马赛克这个人脸/车牌"（已知区域） | `redact.py input.mp4 --x 820 --y 140 --width 240 --height 240 --mode pixelate` |
| "去掉这个固定 logo"（已知区域） | `redact.py input.mp4 --x 40 --y 20 --width 180 --height 60 --mode delogo` |
| "稳定这段抖动素材" | `stabilize.py input.mp4` |
| "这上面有黑边吗？" | `cropdetect.py input.mp4` |
| "iPhone 杜比视界片段显示异常" | `color.py clip.mov --to-sdr` 或 `--strip-dovi`（保留 HDR，去掉 DV 层） |
| "颜色发灰 / iPhone HDR" | `color.py input.mov --to-sdr`（探测显示 `hdr: true`） |
| "应用这个 LUT"，"转换 S-Log 素材" | `color.py input.mp4 --lut grade.cube [--lut-strength 0.7]` |
| "颜色标记错了" | `color.py input.mp4 --retag bt709`（流复制；仅当复制无法承载时才重新编码 — 见 `reencoded`） |
| "调亮 / 增强对比 / 修正白平衡" | `color.py input.mp4 --correct --exposure 0.3 --contrast 1.1 --saturation 1.05 --temperature 5600 --tint -0.05` |
| "用这个 SRT 加字幕"，"烧录字幕" | `caption.py input.mp4 --srt subs.srt` |
| "用这些台词加字幕"（带时间的文本） | `caption.py input.mp4 --text cues.txt` |
| "字幕可开关"，"混流一个 SRT" | `caption.py input.mp4 --srt subs.srt --mode mux`；多语言重复 `--srt file:lang`，超过两种用 `.mkv` |
| "字幕太小 / Short 上占三行"，"不要断句" | `caption.py` 在拆分前先缩小字号直到适配 `--max-lines`（`--fit-size off` 保留 `--size` 改为拆分，`--min-size` 为下限）。保留模板的 `--max-lines`（竖屏为 2）并让字号下降；调高它来避免缩小会导致每行堆两个词。即使到下限仍宽于列的词会在边缘切分（优先连字符），绝不改写，自动处理；JSON 中的 `broken_inside_word` 会计数 |
| "转录并加字幕" | `caption.py input.mp4 --transcribe --animate pop --karaoke`（Parakeet/whisper；否则用 `--text`） |
| "TikTok 风格弹词字幕" | `caption.py input.mp4 --text cues.txt --animate pop --karaoke` |
| "我们的 logo 放右上角"，"水印" | `overlay.py input.mp4 --image logo.png --position top-right --scale 200` |
| "前 4 秒加标题" | `overlay.py input.mp4 --text "Title" --position top --start 0 --end 4 --fade 0.4` |
| "角落放摄像头画面"，"画中画"，"插入"（叠加画面） | `overlay.py input.mp4 --video webcam.mp4 --position bottom-right --scale 480` |
| "去掉绿幕" | `overlay.py bg.mp4 --video greenscreen.mp4 --chromakey 0x00ff00` |
| "带名字的下三分之一字幕条"，"倒计时片头"，"进度条" | `graphics.py input.mp4 --template lower-third --name "..." --title "..." --start 2 --end 8` |
| "贴纸"，"前 3 秒钩子卡"，"梗图文字" | `graphics.py input.mp4 --template sticker --text "NEW" --platform tiktok` / `--template hook --title "..." --duration 3` / `--template meme --top "..." --bottom "..."` |
| "用我们的品牌字体/颜色/logo" | 在 caption/overlay/graphics 上加 `--brand brand.json`，或在 project.json 中设 `"brand"` |
| "同步领夹麦"，"对齐两个机位" | `sync.py camera.mp4 mic.wav --replace-audio` / `sync.py camA.mp4 camB.mp4 --trim-second`；第三路及以上录音作为额外位置参数（`sync.py ref.mp4 mic.wav cam2.mp4`），输出一个偏移 JSON |
| "修正音频电平"，"标准化到 -14 LUFS" | `loudness.py input.mp4`（播客 `-I -16 --tp -1.5`，广播 `-I -23`；`--lra N` 指定范围） |
| "清理音频"，"去掉嘶声" | `audio.py input.mp4 --voice`（语音；`--voice light\|medium\|strong`）或 `--denoise` |
| "在人声下加背景音乐" | `audio.py input.mp4 --music bed.mp3 --duck --fade-out 3`（`--effects sfx.wav` 加第三层，不闪避；项目电平：`audio.stems`） |
| "音乐闪避更强 / 恢复更快" | 加 `--duck-amount 18 --duck-threshold -30 --duck-release 250`（还有 `--duck-attack`） |
| "混音听起来窄"，"更宽的立体声" | `audio.py band.wav --stereo-widen 0.5`（需要真正的立体声源；5.1 需 `--downmix`） |
| "把 5.1 转成立体声" | `audio.py input.mov --downmix` |
| "换入旁白轨" | `audio.py input.mp4 --replace narration.wav` |
| "对白、音乐和音效分轨"，"多轨在一个文件"，"NLE 工程包" | `audio.py input.mov --track music.wav --track fx.wav --track-name Dialogue --track-name Music --track-name FX --track-lang eng --track-lang eng --track-lang eng -o stems.mov`（输入自带声音为轨道 0；每条轨道保留其声道和长度；`.mov`/`.mkv` 用 PCM，`.mp4` 用 AAC；与 `--voice`、`--gain`、`--music` 及其他改变声音的参数同时使用会被拒绝：先处理每条分轨；不写 MXF 和 BWF） |
| "提取音频"，"把声音给我 WAV" | `audio.py input.mp4 -o input.wav`（音频扩展名会丢弃画面；`--audio-stream 1` 选择轨道） |
| "压缩人声"，"限制峰值"，"门限房间噪声" | `audio.py input.mp4 --compress --comp-threshold -20 --comp-ratio 4` / `--limit --limit-ceiling -1` / `--gate --gate-threshold -45` |
| "音频在一小时内逐渐失步" | `sync.py camera.mp4 recorder.wav --fix-drift --replace-audio` |
| "把这个播客变成视频"，"音频可视化" | `render.py --template audiogram ep.m4a --image cover.png` — 波形叠加在静图或色板上；不抓取任何内容 |
| "做成 TikTok / Reel / Short / YouTube / X / LinkedIn / 播客" | `render.py --template tiktok\|reels\|shorts\|youtube-shorts\|youtube\|x\|linkedin\|facebook\|podcast input.mp4 [--cues cues.txt] [--title "..."]` — 画幅、字幕、响度、导出、检查一条命令完成（`--list-templates`；`--write-project` 先编辑） |
| "到处发布"，"一次编辑适配所有平台" | `render.py --template all input.mp4 --cues cues.txt`（或逗号列表）→ 每个目标一个文件加 `<name>_pack.md`（`report.py --pack` 渲染 HTML） |
| "导出给 YouTube / Reels / X"，"ProRes 母版"，"小于 25 MB" | `export.py input.mp4 --preset youtube\|reels\|tiktok\|shorts\|linkedin\|facebook\|x\|prores\|h265`（自 4.0.0 起平台预设在同一次调用中满足其响度规范，`--no-normalize` 仅报告；`--max-mb 25` 适配大小；`youtube-hdr` 保留 HDR，`youtube-av1` 写 AV1） |
| "做个 GIF 预览"，"录制文件损坏了" | `export.py input.mp4 --preset gif` / `--preset copy --repair`（重封装；统计损坏） |
| "小的代理 / 廉价预览文件" | `proxy.py input.mp4 [--width 640 --no-audio]` — 不是交付预设（那是 `export.py`） |
| "这个能上传吗？"，"损失了多少质量" | `check.py final.mp4 --platform reels`（`--compare master.mp4`：SSIM/PSNR） |
| "带章节的播客单集" | `loudness.py ep.wav -I -16 --tp -1.5` → `metadata.py ep.m4a --chapters chapters.txt` → `check.py ep.m4a --platform podcast`（章节和声道行） |
| "给我一份你做了什么的摘要" | `report.py --before raw.mov --after final.mp4 --platform youtube -o report.html` |
| "把这张图变成片段"，"标题卡" | `insert.py title.png --duration 3` |
| "照片缓慢缩放"，"Ken Burns" | `insert.py photo.jpg --duration 6 --zoom in --pan right --width 1920 --height 1080` |
| "做空白/纯色背景片段" | `background.py -o bg.mp4 --duration 3 --width 1920 --height 1080 --color 0x101010` |
| "把这些编号帧变成视频" | `sequence.py --dir frames --pattern "frame_%04d.png" --fps 24` |
| "这条音轨的波形/频谱视频" | `waveform.py podcast.wav -o waveform.mp4` |
| "开头加黑场" | `pad.py clip.mp4 --start 1.5` |
| "切到产品镜头 0:12-0:16"，"这段上 B-roll"，"切出"，"切入"，"反应镜头" | `broll.py talk.mp4 --insert product.mp4 --at 12 --end 16`（重复 `--insert/--at`；`--audio b\|mix`） |
| "加章节"，"YouTube 章节标记" | `metadata.py episode.mp4 --chapters chapters.txt`（每行 `TIME TITLE`；流复制；项目中：`"chapters"`）。`--auto-chapters` 根据实测停顿/场景切换提议（`Chapter N`） |
| "设置标题 / 艺术家 / 注释" | `metadata.py episode.mp4 --title "Episode 12" --artist "Studio"` |
| "把这些视频放进 4x2 网格" | `grid.py t1.mp4 ... t8.mp4 --cols 4 --rows 2` |
| "拼接这些片段"，"加交叉淡化" | `join.py a.mp4 b.mp4 c.mp4 --transition fade --duration 0.5`（TTS：`--list parts.txt`，`--on-missing skip`，`--on-silent fail`；不重新编码：`--copy --transition none`） |
| "J-cut"，"L-cut" | `join.py a.mp4 b.mp4 --transition none --audio-lead 0.5`（J：下一段声音比画面提前 0.5 秒开始）/ `--audio-lag 0.5`（L：上一段声音比画面切换晚 0.5 秒结束）；保持同步，因此每个边界消耗片段 0.5 秒，且该值应用于每个边界；不能与转场、`--copy` 或两个标志同时使用 |
| "一次编辑多处修改"，3 步以上 | `render.py --init project.json`，编辑，`render.py project.json` |
| "我改了一个阶段，不要重做其余" | `render.py project.json --cache DIR` — 相同阶段复用（`--from STAGE` 从该阶段开始） |
| "在 Premiere / Resolve / Final Cut 中打开" | `render.py project.json --export-timeline edit.fcpxml\|.edl\|.otio` — 不渲染任何内容；读取 `not_exported`；没有项目？`--write-project` |
| "对文件夹里每个文件都这样做"，"用所有核心" | `batch.py FOLDER --recipe batch.json --jobs auto`（步骤或渲染项目；带缓存） |
| "三个机位，在它们之间切换" | `multicam.py camA.mp4 camB.mp4 camC.mp4 --switch "0-20:0,20-40:1,40-60:2"`（手动）或 `--switch energy`（`--min-shot`）；`--edl` = 剪辑列表；编辑器：`--write-project p.json` → `--export-timeline` |
| "这个文件里有什么"，"多长" | `probe.py input.mp4` |
| "给我看看效果"，"字幕可读吗" | `look.py output.mp4 --tiles