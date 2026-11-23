# ffmpeg 列表测试夹具

`ffmpeg -hide_banner -filters / -encoders / -bsfs` 的 stdout 捕获或构造结果，供
`tests/test_contract.py` 验证能力检测（`doctor`）能读取 FFmpeg 打印过的每种布局。CI 会将每个 runner 的列表和 `doctor --json` 作为 `ffmpeg-listings-<os>` 构建产物上传；从那里复制新布局到本目录，并加入测试表。

| 文件 | 来源 |
|---|---|
| `ffmpeg_filters_6.1.txt`、`ffmpeg_encoders_6.1.txt`、`ffmpeg_bsfs_6.1.txt` | 从 FFmpeg 6.1.1 **捕获**（Ubuntu 24.04 apt；输出与 ubuntu-latest CI runner 相同） |
| `ffmpeg_filters_8.1.2_macos.txt`、`ffmpeg_encoders_8.1.2_macos.txt`、`ffmpeg_bsfs_8.1.2_macos.txt` | 从 FFmpeg 8.1.2 **捕获**（Homebrew bottle，macos-latest CI runner）。此构建没有 libfreetype / libass / libzimg，因此列表中缺少 `drawtext`、`subtitles`、`ass` 和 `zscale`；测试期望 `doctor` 恰好报告这些为缺失 |
| `ffmpeg_filters_9.0.1_windows.txt`、`ffmpeg_encoders_9.0.1_windows.txt`、`ffmpeg_bsfs_9.0.1_windows.txt` | 从 FFmpeg 9.0.1 **捕获**（gyan.dev essentials 构建，windows-latest CI runner，保留捕获时的 CRLF 行尾） |
| `ffmpeg_filters_7.1_constructed.txt` | **构造**：6.1 捕获未改动，保留其独立名称，以便版本矩阵明确列出 FFmpeg 7（7.x 打印 6.x 布局：三个标志字符 `T.. .S. ..C`）。并非 FFmpeg 7 二进制的捕获。 |
| `ffmpeg_filters_garbage.txt` | 根本不是列表：必须产生 `unknown`，绝不能是 `missing` |

FFmpeg 8.1.2 和 9.0.1 捕获所展示的内容（早先构造的 8.0 文件猜错了图例，已被捕获替换）：图例仍打印三字符键（`T.. = Timeline support`、`.S. = Slice threading`），而行携带两个字符（` TS aap  AA->A`），图例后跟一行 ` ------` 分隔线。解析器以 io-spec 标记（`A->A`、`AA->A`、`|->V`、`N->N`）为键，忽略标志列和图例。

7.1 文件是仅剩的构造文件：它复现 6.x 行格式，使用 6.1 过滤器集，并非 FFmpeg 7 二进制的捕获。手头有捕获时请替换它，并保留文件名以确保测试继续工作。

## mcp_tools.json

不是 ffmpeg 列表：冻结的 1.x MCP/CLI 接口（按工具列出参数名及哪些为必需），由 `tests/test_contract.py` 的 `test_mcp_tool_surface_matches_the_frozen_1x_snapshot` 读取。仅在新增时用 `UPDATE_MCP_SNAPSHOT=1 python3 tests/test_contract.py` 重新生成；参见 docs/contract.md 的“Stability guarantee (1.x)”。