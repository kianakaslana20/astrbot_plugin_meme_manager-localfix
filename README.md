# meme_manager（本地修复版）

> AstrBot 表情包管理器 —— **语义检索触发修复 + 候选压缩** 补丁版
>
> 基线：`anka/astrbot_plugin_meme_manager` **v5.2.11**（MIT）
> 本版标识：**5.2.11-localfix.1**

## 这是什么

上游 `meme_manager` 在「聊天时很少发表情包 / 几乎不发」的场景下有几个叠在一起的问题：语义检索模式实际上进不去，插件静默回退到旧版分类键模式；旧版模式的频率引导和 few-shot 示例都被关掉；即使语义模式生效，候选里过长的描述也会把「输出选图标记」这条格式指令淹没。

本仓库是上游 **v5.2.11 的完整源码** + 针对上述问题的两处代码补丁 + 一份推荐配置，用来把表情包触发率从「偶尔」拉到「每 2 条左右 1 张」。除括号内两处文件外，与上游 v5.2.11 **逐字节一致**：

- `main.py`（候选压缩、提示词强化、表情段前置注入、`import json`）
- `mixins/event_handlers.py`（表情段前置注入）

其余文件（`backend/`、`config.py`、`_conf_schema.json`、`pages/`、`tests/` 等）与上游完全相同。

## 问题根因

### 1. 语义检索模式从未真正生效

`_semantic_pack_ready()` 在 `semantic.tool_probability_enabled = false` 时，会以 `require_tool=True` 调用，此时它要求**当前 LLM 请求的工具集里此刻必须有 `search_memes`**。这个前置判断在 `OnLLMRequestEvent` 阶段不成立时不会抛异常，只是静默 `return False`，于是每次都回落到旧版 `&&分类键&&` 模式。

判定证据：运行日志里反复出现 `[meme_manager] 去重后的最终表情列表: []` —— 这行只由旧版分支（`_resp_impl` 的 legacy 路径）打印，语义分支零命中；插件日志级别开到 DEBUG 后，`本次回复未选择语义表情` 也一次没出现。

### 2. 旧版模式的引导被关得一干二净

回落到 legacy 之后，模型要靠自己输出 `&&分类键&&`。而当时：

| 配置 | 原值 | 后果 |
|---|---|---|
| `generation.prompt.quantity_guidance_enabled` | `false` | 唯一那段「每约 4 条回复配 1 张、连续几条没配就优先配一张」的频率引导被整段丢弃 |
| `generation.prompt.category_example_enabled` | `false` | 分类键 few-shot 示例缺失 |
| `generation.prompt.reply_example_enabled` | `false` | 回复 few-shot 示例缺失 |
| `generation.emotion.probability` | `80` | 额外 20% 的随机丢弃门 |

实测 6/6 次回复解析出的分类键为 0。

### 3. 候选太长，把格式要求淹没了

配置修好、语义模式确实生效之后（能进 tool 分支、预检索返回 5 个候选、候选 JSON 已注入 `system_prompt`），模型**仍然**不输出 `&&meme:ID&&`。原因是每个候选自带 200+ 字的 `caption` 和 10 个 `tags`，5 条候选把 prompt 塞满，注意力被描述文本吃掉，「在正文最后另起一行写标记」这条格式指令被稀释。

`_compact_search_result()` 把每条候选压成 `id / category / caption(≤70 字) / 4 个 tags` 之后，模型立刻稳定遵守。

## 改动清单

### 代码（见 `patches/`）

**`main.py`**

1. 新增 `import json`。
2. 新增 `_compact_search_result()`：解析候选 JSON，截断 `caption` 到 70 字、`tags` 只留 4 个、只保留 `id/category/caption/tags`、删除 `_searched_packs` 与 `max_selectable`。解析失败时原样返回，不影响其他调用方。
3. `_semantic_system_prompt()` 的候选分支：
   - 调用 `_compact_search_result()` 压缩候选；
   - 文案从「可从本轮候选中选择符合回复意图的图片，也可只回复文字」改为更明确的选图倾向，并追加 `[必须执行]` 段（只在纯事实、纯代码、用户要求纯文字、或候选完全不贴合时可以例外）。
4. `_apply_request_prompt()` 的 `semantic_mode == "tool"` 分支：表情段由「拼在 system_prompt **末尾**」改为「**前置**」，避免被后半段长内容稀释。

**`mixins/event_handlers.py`**

5. `_prepare_probability_tool_search()` 末尾的注入顺序同样改为前置（与第 4 条保持一致，两次注入不会互相覆盖）。

### 配置（见 `config-snippet.json`）

| 配置项 | 上游默认 | 本版推荐 | 理由 |
|---|---|---|---|
| `semantic.enabled` | `false` | `true` | 启用语义检索 |
| `semantic.tool_probability_enabled` | `false` | **`true`** | **关键项**：令 `require_tool = not True = False`，跳过「回复模型工具集里必须有 `search_memes`」这个静默失败的前置；同时启用插件内部强制预检索 |
| `semantic.min_score` | `0.25` | `0.2` | 放宽候选池（实测相关 query 的 top5 相似度在 0.51–0.57，留足余量） |
| `semantic.top_k` | `5` | `5` | 保持 |
| `generation.emotion.probability` | `100` | `100` | 去掉随机丢弃门 |
| `generation.prompt.quantity_guidance_enabled` | `false` | `true` | 恢复频率引导 |
| `generation.prompt.category_example_enabled` | `false` | `true` | 恢复分类示例 |
| `generation.prompt.reply_example_enabled` | `false` | `true` | 恢复回复示例 |
| `generation.matching.enable_loose_matching` | `false` | `true` | 容忍分类键大小写/别名偏差 |
| `generation.message.enable_mixed` | `false` | `true` | 图文同条，聊天里更自然 |
| `generation.trigger.scope` | `only_chat_llm` | `chat_and_plugin_llm` | 插件自身发起的 LLM 请求也允许配图 |

`generation.prompt.quantity_guidance` 的文案也建议一并替换（原文节奏是「每约 4 条 1 张」，偏保守）：

```text
[表情数量偏好]
通常配 1 张，一般不超过 2 张。
参考节奏为每约 2 条助手回复中有 1 条配图；近期连续两条回复都没有配图时，这一条优先配 1 张。
只在回答纯事实、纯代码，或用户明确要求纯文字格式时不配图。
```

## 安装

```bash
# 1. 停掉 AstrBot 的 meme_manager 插件（或直接停服务）
# 2. 备份原插件目录
cp -r <ASTRBOT>/data/plugins/meme_manager <ASTRBOT>/data/plugins/meme_manager.bak

# 3. 用本仓库的插件目录覆盖
cp -r meme_manager/* <ASTRBOT>/data/plugins/meme_manager/

# 4. 合并配置（不要直接覆盖已有配置，里面有你装的包和 provider 设置）
#    把 config-snippet.json 里对应的键改动合并进：
#    <ASTRBOT>/data/config/meme_manager_config.json
```

或者只打补丁：

```bash
cd <ASTRBOT>/data/plugins/meme_manager
git apply --verbose /path/to/patches/0001-main-py.patch
git apply --verbose /path/to/patches/0002-event-handlers-py.patch
```

改完**必须重载插件**——`main.py` 里 `semantic_enabled`、`tool_probability_enabled`、`emotions_probability`、`quantity_guidance` 等全部只在 `__init__` 读取一次：

```bash
# WebUI：插件页 → 重载；或用 dashboard API
curl -X POST http://127.0.0.1:<DASHBOARD_PORT>/api/v1/plugins/meme_manager/reload \
  -H "Authorization: Bearer <JWT>"
```

## 验证

重载后随便聊几句带情绪的话，日志里应出现：

```text
[meme_manager] 已选择语义表情：['meme:7c92741069f5']
```

实测 5 条（DeepSeek-flash + bge-m3 本地向量 + 单包 163 张索引，`min_score=0.2`、`top_k=5`）：

| 输入 | 是否配图 |
|---|---|
| 今天加班到现在，好累啊，整个人都困翻了 | ✅ |
| 哈哈哈你这也太搞笑了我笑死了 | ✅ |
| 呜呜呜我今天真的好委屈，被领导骂了 | ✅ |
| 刚吃完饭，好困想睡了 | ✅ |
| 你今天怎么这么慢才回我 | ❌（质问场景不配图，符合预期） |

如果还是不发图，按这个顺序查：

1. `semantic.enabled` 是否 `true`、`tool_probability_enabled` 是否 `true`；
2. 表情包是否已装且 `enabled`（`data/plugin_data/meme_manager/registry.json`）；
3. 语义索引是否就绪：`plugin_data/meme_manager/semantic_indexes/<pack_id>/index-*.faiss` + `index_manifest.json`，且 manifest 里 `embedding_provider_id` / `embedding_model` / `embedding_dimension` 与当前 embedding provider 一致；
4. embedding provider 是否可达（`provider_selection.json` 的 `updated_at` 会随命中刷新，是个很好的活体探针）；
5. 把插件日志级别调到 DEBUG，看是走 legacy（有 `去重后的最终表情列表`）还是语义分支（有 `本次回复未选择语义表情`）。

## 兼容性与提醒

- 目标 AstrBot `>=4.17.0`（随上游 `metadata.yaml`）。
- 改动全部在插件层，不碰 AstrBot 内核。
- **上游更新会覆盖本补丁**。升级前先 `git diff` 对比，或直接用 `patches/` 里的补丁重打。
- 插件市场里的 `meme_manager` 已经是 **5.3.3**，本仓库基线是 **5.2.11**。补丁的上下文行在 5.3.3 上大概率能应用，但未验证；建议要么留在 5.2.11，要么把改动按 `patches/` 手工移植到 5.3.3。
- 表情包会真的发出去，注意人设、群规和对方的接受度；`generation.emotion.probability` 是最后的概率闸门，觉得太频繁就调到 70–80。

## 引用来源

| 项 | 值 |
|---|---|
| 上游仓库 | https://github.com/anka-afk/astrbot_plugin_meme_manager |
| 上游作者 | anka（https://github.com/anka-afk） |
| 基线版本 | v5.2.11（tag `v5.2.11`） |
| 上游 tag 源码 | https://codeload.github.com/anka-afk/astrbot_plugin_meme_manager/zip/refs/tags/v5.2.11 |
| 许可证 | MIT（Copyright (c) 2025 anka），见 `LICENSE` |
| 插件市场条目 | `anka/meme_manager`，显示名「表情包管理器」 |
| 依赖 | `aiohttp` / `tqdm` / `minio>=7.2.20,<8` / `pillow` / `requests` / `faiss-cpu` |

本仓库不是上游官方发布，是本地部署上的问题定位与补丁记录。若上游合并了等价修复，请优先使用上游版本。

## 版本

`5.2.11-localfix.1` —— 见 [CHANGELOG.md](CHANGELOG.md)。
