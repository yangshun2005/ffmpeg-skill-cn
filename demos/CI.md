# 演示画廊的 CI — 需要添加的 YAML

此处描述的改动**尚未应用**：它们涉及 `.github/workflows/`，而用于这些 PR 的 App token 无权合并该目录。维护者需将下面两个代码块复制到指定文件中；本 PR 的其他内容均不依赖它们。

两者成本都很低：`ubuntu-latest` 按 1x 计费，整个画廊在该规格的 runner 上约六分钟即可渲染完成。

---

## 1. `.github/workflows/ci.yml` — 一个 `demos` 任务

将其添加为 `test` / `test-ffmpeg-7` 的同级任务（相同缩进，位于 `jobs:` 下）。它安装 ffmpeg 和字体的方式与现有 Linux 矩阵分支完全一致 — `fonts-dejavu-core` 用于 BEFORE/AFTER 标签，外加字幕演示所需的 CJK 和阿拉伯语字族，否则这些演示会自行跳过，任务静默证明的内容将少于表面所见。

```yaml
  # Renders the whole before/after gallery from synthetic footage on every PR. This is a real
  # end-to-end test of every script under scripts/ except the four inspection-only ones
  # (probe, verify, batch, report), which print a table rather than a picture -- tests/test_orchestration.py
  # pins that split, so a new tool cannot land without either a demo or an entry on that list.
  # It is also the gate on
  # what lands in docs/demos/: build.py fails when a committed preview goes over 500 KB, so a
  # PR cannot quietly grow the repo's binaries. Linux only -- it bills at 1x and the demos are
  # about the tools, not about per-OS ffmpeg differences, which the test matrix already covers.
  demos:
    name: demo gallery (ubuntu-latest)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: "3.10"
      # Same apt line as the matrix job, plus the CJK/Arabic faces: without them the
      # captions_ja / _zh / _ko / _ar demos skip, and a skipped demo proves nothing. The
      # stabilize and sphere demos need an ffmpeg built with vidstab and v360; Ubuntu's
      # package has both, and build.py names the missing filter when it does not.
      - name: Install ffmpeg and fonts
        run: |
          sudo apt-get update
          sudo apt-get install -y ffmpeg fonts-dejavu-core fonts-ipafont-gothic \
                                  fonts-wqy-zenhei fonts-noto-core
      - name: Build the demo gallery
        run: python3 demos/build.py
      # The previews are committed, so CI must fail when the working tree's versions differ
      # from what a fresh render produces -- otherwise docs/demos.md shows a picture of an
      # older build of the tool. Only the generated page is diffed, not the GIFs: GIF bytes
      # vary with the ffmpeg build, the page text does not.
      - name: docs/demos.md is up to date with demos/build.py
        run: |
          python3 demos/build.py --docs
          git diff --exit-code -- docs/demos.md
      - uses: actions/upload-artifact@v7
        if: always()
        with:
          name: demo-gallery
          path: |
            demos/out/*.mp4
            docs/demos/*
          retention-days: 14
```

如果意图是让画廊失败阻止合并，请将 `demos` 添加到分支保护规则集的必需状态检查中（`main` 当前要求 8 项检查；这将使其变为 9 项）。

### 为什么预览不由 CI 重新生成并提交回去

机器人每次 PR 推送数 MB 的 GIF 会搅乱仓库历史，而这些改动通常不可见（即使画面完全相同，不同 ffmpeg 构建产出的 GIF 字节也不同）。作者运行 `python3 demos/build.py` 并提交结果；CI 只证明它仍能渲染，且 `docs/demos.md` 与 `demos/build.py` 中的表格一致。

---

## 2. `.github/workflows/release.yml` — 将完整 MP4 附加到发布版本

`docs/demos.md` 中的画廊显示 480 px 的 GIF，因为它们必须存放在 git 中。640 px 的并排 MP4（以及全分辨率的 `_before` / `_after` 对）是更好的构建产物，GitHub Release 是存放它们的合适位置：它有版本、不在仓库历史中，且附加成本为零。

两个步骤，插入现有 release 任务中 `Create tag and GitHub Release` 步骤（`softprops/action-gh-release@v3`，约第 242 行）**之前**，并对该步骤本身做一处添加。

```yaml
      # Build the gallery at the exact commit being released and zip it, so the release
      # carries watchable proof of what this version does. `|| true` is deliberate: a failed
      # demo render must never block a release that npm and the tag are already committed to
      # -- the missing asset is visible on the release page, a half-published version is not.
      - name: Render the demo gallery for the release assets
        if: steps.check.outputs.already_released == 'false'
        run: |
          sudo apt-get update
          sudo apt-get install -y ffmpeg fonts-dejavu-core fonts-ipafont-gothic \
                                  fonts-wqy-zenhei fonts-noto-core
          python3 demos/build.py || true
          mkdir -p release_assets
          # Side-by-side comparisons only: the _before/_after pairs are another ~60 MB and are
          # reproducible from the same one command, which is named in the zip's README.
          cp demos/out/*.mp4 release_assets/ 2>/dev/null || true
          rm -f release_assets/*_before.mp4 release_assets/*_after.mp4
          cp -r docs/demos release_assets/previews
          printf 'Rebuild these with:\n\n    python3 demos/build.py\n' > release_assets/README.txt
          cd release_assets && zip -r ../demo-gallery-${{ steps.check.outputs.tag }}.zip .
```

以及现有 release 步骤上的改动：

```yaml
      - name: Create tag and GitHub Release
        if: steps.check.outputs.already_released == 'false'
        uses: softprops/action-gh-release@v3
        with:
          tag_name: ${{ steps.check.outputs.tag }}
          target_commitish: ${{ steps.check.outputs.sha }}
          body_path: release_notes.md
          generate_release_notes: true
          files: demo-gallery-${{ steps.check.outputs.tag }}.zip   # <- the one added line
          fail_on_unmatched_files: false                           # <- and its guard
```

`fail_on_unmatched_files: false` 与上面的 `|| true` 配套：如果渲染失败，发布版本仍会在没有该资产的情况下发布。随后发布说明可以在 CHANGELOG 模板中用一行链接画廊 — `Demos: docs/demos.md (full MP4s attached below)`。