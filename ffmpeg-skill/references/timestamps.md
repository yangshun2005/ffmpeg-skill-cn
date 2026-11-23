# 时间戳语法

时间戳参数（`--start`、`--end`、`--at`、`broll.py --from`、`--duration`、`--offset`，以及 cue 和章节文件中的时间）接受秒、`mm:ss(.fff)`、`hh:mm:ss(.fff)` 或带 `@fps` 的 SMPTE `hh:mm:ss:ff`（`00:01:02:15@29.97`）——当 NLE cue 表的时间码为非丢帧时可直接粘贴使用（丢帧时间码 `;` 在参数中和 `--text` cue 文件中都会被拒绝）；长度参数（`--min-silence`、`--margin`、`--fade`）为纯秒数。`render.py --from` 接受阶段名，而非时间。