# 隐私

ffmpeg-skill 是一组在你自己的机器上运行的命令行工具。本页说明随附版本对数据的处理方式。

- **不收集数据。** 工具不发送遥测、分析或使用数据，项目也没有可接收这些数据的服务器。无需账号，也无需登录。
- **你的媒体保留在本地。** 视频、音频、字幕和项目文件只从你指定的路径读取并写入。不会上传任何内容。
- **工具本身不使用网络**，也不读取 API 密钥。唯一例外：转录（`caption.py --transcribe`、`silence.py --transcribe` 配合 `--filler` 或句子标志）会运行你自己安装的语音引擎（whisper.cpp、faster-whisper、openai-whisper 或 parakeet）。这类引擎首次运行时可能从其提供商处下载模型，Hugging Face 客户端会使用你已登录的令牌。这些引擎和提供商的行为受其各自政策约束。
- **安装**：从 npm、GitHub 或插件目录安装即从相应服务下载，受该服务政策约束。
- **环境变量**：工具读取的环境变量仅用于配置：`FFMPEG_SKILL_*` 开关，语音模型用的 `PARAKEET_MODEL` 和 `PARAKEET_CPP_MODEL`，以及用于查找程序和字体的 `PATH`、`PATHEXT` 和 `WINDIR`。

如有疑问：请在 https://github.com/kajisho5/ffmpeg-skill/issues 提交 Issue。