# MCP 参考

MCP 路径：在 `~/.claude/settings.json` 配 `mcpServers.notebooklm`，重启 Claude Code 后工具自动可用。

## 配置

`~/.claude/settings.json`：

```json
{
  "mcpServers": {
    "notebooklm": {
      "command": "nlm",
      "args": ["mcp"]
    }
  }
}
```

> 也可以指向 `nlm` 的 Python 解释器绝对路径（`which nlm` 拿到），避免 PATH 不一致。

## 工具清单

| 工具 | 用途 |
|---|---|
| `mcp__notebooklm__get_health` | 健康检查（认证/库/会话状态） |
| `mcp__notebooklm__setup_auth` | 首次登录（打开浏览器） |
| `mcp__notebooklm__cleanup_data` | 清认证/缓存（先 close 浏览器） |
| `mcp__notebooklm__list_notebooks` | 列本地库的笔记本（元数据） |
| `mcp__notebooklm__search_notebooks` | 按关键词搜本地库 |
| `mcp__notebooklm__get_notebook` | 单个详情 |
| `mcp__notebooklm__add_notebook` | **加进本地库**（不是上传到云！） |
| `mcp__notebooklm__update_notebook` | 改 metadata |
| `mcp__notebooklm__remove_notebook` | 从本地库移除 |
| `mcp__notebooklm__select_notebook` | 设为 active，后续 `ask_question` 默认 |
| `mcp__notebooklm__ask_question` | 让 Gemini 2.5 问答 |
| `mcp__notebooklm__list_sessions` | 列活跃会话 |
| `mcp__notebooklm__reset_session` | 清会话历史（保留 session id） |
| `mcp__notebooklm__close_session` | 关闭会话 |
| `mcp__notebooklm__add_source` | 加 source 到 notebook |
| `mcp__notebooklm__generate_audio` | 生成 Audio Overview（异步） |
| `mcp__notebooklm__get_audio_status` | 轮询音频状态 |
| `mcp__notebooklm__download_audio` | 下载生成的音频 |

## 工作流：把云端笔记本同步进本地库

CLI 用 `nlm notebook list` 看云端；MCP 库只是一份 URL 元数据缓存，方便在编辑器里用名字快速调。

> 关键：`add_notebook` 的 `url` 参数 = `https://notebooklm.google.com/notebook/<UUID>`。
> 你**必须先用 CLI 拿到 UUID**（因为 CLI 直接打 Google），再喂给 MCP `add_notebook` 加进本地库。

```text
CLI: nlm notebook list            → 拿到 UUID
MCP: add_notebook(url=...,        → 元数据进本地库
                name=..., 
                description=...,
                topics=[...],
                use_cases=[...])
MCP: select_notebook(id=...)      → 设为 active
MCP: ask_question(question=...)   → 直接问
```

## `ask_question` 关键参数

| 参数 | 说明 |
|---|---|
| `question` | 必填。问 Gemini 的问题 |
| `notebook_id` | 库里的 id（库内 select 后可不传） |
| `notebook_url` | 直接 URL（覆盖 `notebook_id`，ad-hoc 用） |
| `session_id` | 续接上次问答（同一会话保留上下文） |
| `source_format` | 引用格式：`none` / `footnotes` / `inline` / `json` |
| `browser_options.headless` | 是否显示浏览器（debug 用 true） |

返回结构（精简）：

```json
{
  "status": "success",
  "answer": "...",
  "session_id": "074ed1ef",
  "sources": [{"marker": "[1]", "number": 1, "sourceName": "...", "sourceText": "..."}, ...]
}
```

> 留 `session_id`，追问时回传，能让 NotebookLM 保留对话上下文。

## 兜底

如果 MCP 工具消失或调用失败：

1. `get_health` 看认证状态
2. 不对就 `setup_auth` 重登
3. 还不对就 `cleanup_data(confirm=true, preserve_library=true)` 然后 `setup_auth`
4. 最后回 CLI：`nlm ...`
