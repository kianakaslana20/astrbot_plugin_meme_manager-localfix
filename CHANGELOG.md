# CHANGELOG

## 5.2.11-localfix.1

基线：`anka/astrbot_plugin_meme_manager` v5.2.11（MIT）。
补丁文件：`patches/0001-main-py.patch`、`patches/0002-event-handlers-py.patch`。
推荐配置：`config-snippet.json`。

### 修复

- **语义检索模式无法进入**：`semantic.tool_probability_enabled = false` 时 `_semantic_pack_ready(require_tool=True)` 会要求当前请求的工具集里已有 `search_memes`，不满足时静默返回 `False`，插件持续回落到旧版分类键模式。改为 `true` 后 `require_tool=False`，只按 embedding provider + 索引就绪度判定，语义模式正常生效。
- **候选描述稀释选图指令**：新增 `_compact_search_result()`，把候选压成 `id / category / caption(≤70 字) / 4 个 tags`，并移除 `_searched_packs`、`max_selectable`。在此之前模型虽然拿到了候选，也几乎不输出 `&&meme:ID&&`。
- **表情指令被后半段长 prompt 压制**：`_semantic_system_prompt()` 的注入位置由 `system_prompt` 末尾改为前置（`main.py::_apply_request_prompt`、`mixins/event_handlers.py::_prepare_probability_tool_search` 两处同步修改）。
- **提示词力度不足**：候选分支文案改为明确的选图倾向，并追加 `[必须执行]` 段，只保留「纯事实 / 纯代码 / 用户要求纯文字 / 候选完全不贴合」四类例外。

### 配置建议变更

- `semantic.enabled`: `false` → `true`
- `semantic.tool_probability_enabled`: `false` → `true`（关键）
- `semantic.min_score`: `0.25` → `0.2`
- `generation.emotion.probability`: 保持 `100`
- `generation.prompt.quantity_guidance_enabled`: `false` → `true`
- `generation.prompt.category_example_enabled`: `false` → `true`
- `generation.prompt.reply_example_enabled`: `false` → `true`
- `generation.matching.enable_loose_matching`: `false` → `true`
- `generation.message.enable_mixed`: `false` → `true`
- `generation.trigger.scope`: `only_chat_llm` → `chat_and_plugin_llm`
- `generation.prompt.quantity_guidance` 文案：节奏由「每约 4 条 1 张」调整为「每约 2 条 1 张」，并保留「通常 1 张、不超过 2 张」的数量约束

### 验证

DeepSeek-flash 对话 + 本地 bge-m3（1024 维）+ 单包 163 张索引、`top_k=5`、`min_score=0.2`，5 条测试 4 条配图（加班、搞笑、委屈、犯困四类均命中；被质问「怎么这么慢才回」的条目未配图，符合预期）。日志可见 `[meme_manager] 已选择语义表情：[...]`。

### 未改动

`backend/`、`config.py`、`_conf_schema.json`、`pages/`、`tests/`、`requirements.txt`、`metadata.yaml` 与上游 v5.2.11 完全一致（逐文件 md5 校验）。不修改版本号，避免与上游 tag 混淆。
