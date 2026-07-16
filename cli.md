# CLI 参考

CLI 路径：`pip install notebooklm-mcp-cli` 后得到 `nlm` 命令。

## 认证

```bash
nlm login                    # 打开浏览器登一次 Google
nlm logout                   # 清掉本地 cookies
cat ~/.notebooklm-mcp-cli/auth.json   # 看保存的 cookies（敏感）
```

## 笔记本操作

```bash
nlm notebook list                          # 全部笔记本
nlm notebook list --json                   # JSON 输出（便于脚本处理）
nlm notebook list --format=table           # 表格输出
nlm notebook create "标题"                 # 新建
nlm notebook get <nb_id>                   # 单个详情
nlm delete notebook <nb_id> -y             # 删除（不可逆）
```

## Source 操作

```bash
# 列 source
nlm source list <nb_id>
nlm source list <nb_id> --json | jq '.[].title'   # 只取标题

# 读 source
nlm source content  <src_id>                       # 原始文本（可能很大）
nlm source content  <src_id> --output /tmp/x.txt   # 落盘
nlm source describe <src_id>                       # NotebookLM AI 摘要
nlm source get      <src_id>                       # 完整内容（最大）

# 加 source
nlm source add <nb_id> --text "..."     --title "标题" --wait
nlm source add <nb_id> --file /tmp/x.txt              --title "标题" --wait
nlm source add <nb_id> --url  https://example.com/x    --title "标题" --wait
nlm source add <nb_id> --youtube "https://youtu.be/..." --wait

# 删 source
nlm delete source <src_id> -y
```

> `--wait`：阻塞等到 Google 索引完；不加会立刻返回，源头还没索引就 `query` 会丢。

## 问答（Gemini 综合）

```bash
# 基础
nlm notebook query <nb_id> "你的问题"

# JSON 输出
nlm notebook query <nb_id> "你的问题" --json

# 长查询加 timeout
nlm query notebook <nb_id> "深度分析..." --timeout 180

# MCP 风格的 select notebook（开启会话上下文）
nlm query notebook <nb_id> "follow-up..." --session <session_id>
```

输出 JSON 结构样例：

```json
{
  "status": "success",
  "answer": "Gemini 的回答文本...",
  "session_id": "...",
  "sources": [
    {"marker": "[1]", "number": 1, "sourceName": "...", "sourceText": "..."},
    ...
  ]
}
```

## 提取 answer 字段的 Python 模板

CLI 的 JSON 嵌套有时很深，用递归查找兜底：

```python
import json, subprocess
out = subprocess.check_output(["nlm", "notebook", "query", nb_id, "你的问题", "--json"])
data = json.loads(out)

def find(o, key):
    if isinstance(o, dict):
        if key in o: return o[key]
        for v in o.values():
            r = find(v, key)
            if r is not None: return r
    elif isinstance(o, list):
        for v in o:
            r = find(v, key)
            if r is not None: return r
    return None

print(find(data, "answer"))
```

## 出错应急

| 现象 | 处理 |
|---|---|
| `SSL: EOF` / `connection reset` | `unset http_proxy ...` 后重试 |
| `Cookie expired` | `nlm login` 重登 |
| `Could not add text source` | 重试 + 检查是否真没加上（看 stdout 是否有 `"Added source:"`） |
| `Notebook not found` | UUID 用 `notebook list` 重对一下 |
| `Rate limit exceeded` | 免费版每日 50 次问答，要么换号要么等第二天 |
