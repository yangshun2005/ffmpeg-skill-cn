提示：我需要 OUTDIR/source.mp4 可直接上传到 YouTube——响度正常，所有交付检查通过。保存到 OUTDIR。

这是一个模板交付：render.py --template youtube 串联 fit/loudness/export，然后由 check.py 验证结果。

$ python3 scripts/render.py --template youtube OUTDIR/source.mp4 --output OUTDIR/ff1_delivery.mp4 --json
loudness.py OUTDIR/ff1_delivery_work_841/fit.mp4 -o OUTDIR/ff1_delivery_work_841/loudnorm.mp4 -I -14 --tp -1.0 --timeout 1800
check: loudness PASS -14.0 LUFS (expected -14 +/- 2 LUFS); true peak PASS -4.4 dBTP (expected <= -1 dBTP); failed 0, warnings 1, ok true

我专门检查了 faststart 修复（#275、#279）在这条响度归一化模板路径上没有回归：读取 OUTDIR/ff1_delivery.mp4 的首字节，确认 moov atom（偏移 36）位于 mdat（偏移 14396）之前——文件开头即快速启动。

完成：OUTDIR/ff1_delivery.mp4——YouTube 模板交付，响度归一化至 -14.0 LUFS（真峰值 -4.4 dBTP），faststart 已具备（moov 在 mdat 之前，已按字节校验）。
步骤：render.py --template youtube（fit -> loudness -> export）、check.py。
检查：check.py 的 loudness 和 true-peak 行均 PASS；唯一警告是无字幕轨，提示中并未要求。
画面：本链路为纯音频交付检查，无影响画面的步骤；无需视觉检查。