# 上游来源与许可

## 来源

| 项 | 值 |
|---|---|
| 项目 | `astrbot_plugin_meme_manager`（AstrBot 插件，显示名「表情包管理器」） |
| 上游仓库 | https://github.com/anka-afk/astrbot_plugin_meme_manager |
| 作者 | anka — https://github.com/anka-afk |
| 基线 tag | `v5.2.11` |
| 基线获取地址 | https://codeload.github.com/anka-afk/astrbot_plugin_meme_manager/zip/refs/tags/v5.2.11 |
| 基线 commit 参考 | 见上游 tag `v5.2.11` |
| 插件市场条目 | `anka/meme_manager`，市场版本 **5.3.3**，下载地址 `https://astrbot-cloud-test-s3.astrbot.app/plugins/releases/anka/meme_manager/5.3.3/meme_manager-5.3.3-ec80e0a12f76.zip` |
| 许可证 | MIT，Copyright (c) 2025 anka（见 `LICENSE`） |
| 依赖 | `aiohttp`、`tqdm`、`minio>=7.2.20,<8`、`pillow`、`requests`、`faiss-cpu` |

## 基线一致性

本仓库的插件源码目录 `meme_manager/` 与上游 `v5.2.11` tag 解压结果逐文件比对（md5）：

- 文件集合完全一致（无新增、无缺失）；
- 仅两个文件内容不同：`main.py`、`mixins/event_handlers.py`；
- 差异即 `patches/` 下两份 unified diff。

因此 `meme_manager/` 中除这两处之外的文件都可视为上游原文，未做任何改动。

## 许可遵从

上游为 MIT，允许修改与再分发，条件是保留版权声明与许可文本。本仓库：

- 保留原始 `LICENSE`（Copyright (c) 2025 anka）不做修改；
- 在 `README.md`、`CHANGELOG.md`、本文件与每个 commit message 中标注上游来源与基线版本；
- 明确声明本仓库为**非官方修改版**，问题与建议请优先反馈上游。

## 与上游同步

上游 v5.2.11 之后已有 v5.3.x 系列。升级路径建议：

```bash
# 取上游新版本
git clone https://github.com/anka-afk/astrbot_plugin_meme_manager upstream
cd upstream && git checkout v5.3.3

# 尝试应用本仓库补丁
git apply --3way /path/to/patches/0001-main-py.patch
git apply --3way /path/to/patches/0002-event-handlers-py.patch

# 若有冲突，按 patches 里的 hunk 手工移植。关键点只有三处：
#   1. _semantic_system_prompt() 的候选分支注入 _compact_search_result()
#   2. _apply_request_prompt() 的 tool 分支把表情段前置
#   3. _prepare_probability_tool_search() 末尾同样前置
```

配置项（`config-snippet.json`）与版本无关，可直接沿用。
