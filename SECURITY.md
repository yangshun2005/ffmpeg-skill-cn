# 安全策略

## 报告漏洞

请通过 GitHub 的[报告漏洞](https://github.com/kajisho5/ffmpeg-skill/security/advisories/new)流程（Security 标签页 → Advisories → “Report a vulnerability”）私下报告安全问题，不要使用公开 Issue 或 PR。
该流程会创建一个私有安全公告，只有维护者可见，直到修复就绪。

请尽量包含：
- 受影响的文件/工具（例如 `scripts/batch.py`）及版本
- 最小复现（触发问题的输入，以及实际发生了什么）
- 你预期的影响（例如任意文件写入、路径遍历、注入到 ffmpeg 滤镜图）

本项目此前确实通过这种方式修复过真实问题——例如 `batch.py` 的 recipe 步骤可能通过绝对路径或 `../` 遍历解析到 `scripts/` 之外（已在 #124 修复）。这类报告正是该流程的用途所在。

## 支持的版本

仅支持最新发布版本（见 [CHANGELOG.md](CHANGELOG.md) / 最新 npm 发布版本）。没有 LTS 分支——如果你使用的是旧版本，请先升级再报告。