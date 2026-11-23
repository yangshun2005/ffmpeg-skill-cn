# 执行的命令

```
cat /home/user/ffmpeg-skill/SKILL.md | head -100
ls -la /tmp/claude-0/ev24/h1/
cd /tmp/claude-0/ev24/h1/ && python3 /home/user/ffmpeg-skill/scripts/probe.py wide.mp4 --json
```

# 关键输出（对 wide.mp4 运行 probe.py --json）

```json
{
  "file": "wide.mp4",
  "duration": 4.0,
  "video": {
    "codec": "hevc",
    "profile": "Main 10",
    "width": 640,
    "height": 360,
    "pix_fmt": "yuv420p10le",
    "bit_depth": 10,
    "hdr": false,
    "hdr_signal": false,
    "bt2020_or_hdr": true,
    "hdr_format": "BT.2020 SDR",
    "dolby_vision": null,
    "color_space": "bt2020nc",
    "color_primaries": "bt2020",
    "color_transfer": "bt709",
    "color_range": "tv"
  },
  "audio": null
}
```

另外按 AGENTS.md 的指示查看了 `/home/user/ffmpeg-skill/docs/design-decisions.md`，其中确认这里的 `hdr: false` 正是工具真实、预期的含义：自该 skill 的 2.0 契约起，`hdr` 只报告真正的 HDR 信号（PQ/HLG/Dolby Vision）。此文件的色彩原色是 BT.2020，但传递曲线是 SDR（bt709）——属于"BT.2020 SDR"边界情况，并非真正的 HDR。该组合是 10 位，内部会走支持 HDR 的编码路径，但显示和行为都按 SDR。

# 给用户的最终回复

wide.mp4 **不是 HDR**——可以直接发给仅支持 SDR 的客户端。

探测详情：它是 HEVC Main 10（10 位，640x360，yuv420p10le），色彩原色标记为 BT.2020，但传递特性是 `bt709`（标准 SDR gamma 曲线），且没有 PQ/HLG/Dolby Vision 信号（`dolby_vision: null`）。所以这是"BT.2020 原色 + SDR 传递"的文件——属于边界情况，不是真正的 HDR（PQ/HLG）。交付时无需执行 `color.py --to-sdr` 转换。

一点提醒：它是 10 位（`yuv420p10le`），一些较老的仅支持 SDR 的播放/剪辑系统对此可能比较挑剔，尽管色彩信号本身是 SDR。如果你的客户端流水线明确要求 8 位，告诉我，我可以转换（例如用 `color.py` 或重新编码为 8 位 h264）——但仅就 HDR 问题而言，此文件会正确显示为 SDR。