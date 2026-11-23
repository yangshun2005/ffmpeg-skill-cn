# 报告格式完整说明

SKILL.md 中报告格式的完整版：语言规则、日语示例，以及失败、拒绝和部分结果的规则。

用请求所用的语言回复——其他语言的字幕仍用请求的语言报告。标签（`Done:`、`Steps:`、`Check:`、`Look:`、`Notes:`）保持英文，其余内容用用户的语言。绝不因任务简短或失败而偏离：哪怕只有一行"文件不存在"。对话中途切换语言时，跟随用户最新消息。

每个任务都以这个格式收尾（数字来自 `--json` 或 `probe.py`/`check.py`，不凭记忆）：

```
Done: final.mp4 — 59.98 s, 1080x1920, 30 fps, H.264, AAC stereo, -14.1 LUFS
Steps: cut 0:12-1:12 (lossless) -> fit 9:16 crop -> captions (pop, karaoke) -> loudness -14 -> export reels
Check: reels — all 12 checks pass (verified: true)
Look: final_sheet.png (captions inside the safe area, logo top-right)
Notes: source was VFR, conformed to 30 fps; audio was mono, made stereo
```

日语请求的同样五行：

```
Done: final.mp4 — 59.98 秒、1080x1920、30 fps、H.264、AAC ステレオ、-14.1 LUFS
Steps: 0:12-1:12 をカット（無劣化）-> 9:16 にクロップ -> 字幕（ポップ、カラオケ）-> ラウドネス -14 -> Reels 書き出し
Check: reels — 12 項目すべて合格（verified: true）
Look: final_sheet.png（字幕はセーフエリア内、ロゴは右上）
Notes: 元は VFR だったので 30 fps に揃えた。音声はモノラルだったのでステレオにした
```

这五行就是报告本身。包含用户需要做出的任何决定。绝不描述你没运行过的修复。`verified` 只覆盖已测量的内容：在 `Notes:` 中列出用户会以为已检查的 `unverified` 项。用 `attribution: unresolved` 报告某一步；不要重跑它。

对于独立编辑，先交付结果和五行报告。当宿主能展示审阅且这样做有帮助时，用用户的语言提议一次。日语为「編集前と編集後を、プレビューで見比べますか？」。当宿主能显示波形且媒体有音轨时，提及波形。仅在用户要求或接受提议后才准备审阅。若用户已要求，检查宿主能展示什么，并在编辑和报告后直接继续，无需再次提示。当宿主没有审阅视图时，说明限制并分享用户要求的任何受支持的预览。

当宿主支持并排播放时，若原始和编辑后的视频都有画面，则同时展示两者。对于纯音频编辑，提供可用音频的播放。当宿主支持波形时，从可用音频绘制波形，包括静音轨道的平直波形。当宿主能使用已知的源到输出时间映射时，同步播放和波形跳转。否则，分别标注并展示两个时钟。宿主负责视图、编号的编辑项，以及如何将反馈映射到新的编辑请求。使用 ffmpeg-skill 提供媒体、测量结果和可用的预览文件，并执行请求的更改。Velisa 及其他负责审阅的调用方通过自己的工作流处理审阅。

当某一步失败时，将 `Done:` 替换为 `Failed:`，其余保持诚实：

```
Failed: color.py --lut grade.cube exited 1 — ffmpeg: "Unable to parse LUT file" (the .cube is not a valid LUT)
Steps: probe -> color (failed); nothing written
Check: nothing to verify
Look: not needed
Notes: send a valid .cube, or say if you want the clip left as is
```

这些填充行是句子，不是标签：日语请求的同样报告结尾为 `Check: 検証するものなし` / `Look: 不要`。

拒绝使用相同格式：`Failed:` 说明拒绝了什么及原因，`Steps:` 列出实际运行了什么，`Look: not needed`。部分结果是 `Done:`，不足写在 `Notes:` 中；仍交付了东西的拒绝是 `Failed:`——绝不用第三种标签如 `Done (partially):`。失败的 `--json` 是 `status: failed` 加上 `error.kind`、`error.message` 和 `error.hint`：在 `Failed:` 中引用消息，绝不转述，提示放在 `Notes:` 中。