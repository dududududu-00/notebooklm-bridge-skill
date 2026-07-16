# 故障排查

## 网络类

### SSL: EOF / connection reset

**原因**：HTTP 代理与 Google NotebookLM 不兼容。

**修**：

```bash
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY all_proxy
```

加到 `~/.zshrc` / `~/.bashrc` 也行（但不要裸 `export http_proxy=`）。CLI 调 Google 前永远先 unset。

### Cookie expired / 401

```bash
nlm logout
nlm login
```

MCP 路径：

```text
mcp__notebooklm__cleanup_data(confirm=true, preserve_library=true)
mcp__notebooklm__setup_auth()
```

## 容量类

### Could not add text source（~50 份后）

**症状**：单笔记本 ~50 份起，`nlm source add` 偶发 `returncode=1` + `Could not add text source`。

**解法**：

1. **别相信 returncode**，看 stdout 是否含 `"Added source:"` / `"Source ID:"`
2. 没真正加上就重试（通常 2-3 次内过）
3. 加完后立刻 `source list` 验证实际数量
4. 大文件超时 → `--timeout 180` 或拆成多份

### Google 索引没跟上（query 漏内容）

**症状**：`source add --wait` 没传，源还没索引完就 `query`，回答丢失新加的资料。

**修**：粘贴时一律加 `--wait`。

## AI 答案类

### Gemini 编造数字 / 引文

**原因**：NotebookLM 的 Gemini 2.5 答案基于上传源，但仍会幻觉。

**修**：

1. 重要结论**必打开对应 source 核对**
2. 命令：`nlm source content <src_id> --output /tmp/x.txt`
3. 拿到原文本后可 grep 关键数字，确认在哪一份 source 里、有没有上下文
4. 跨资料综合题（"对比 A 和 B"）幻觉概率更高，引用源要逐个验

### 答案太短 / 不到位

**修**：让 question 更结构化，例如：

```
不要:
  "讲一下这个项目"

要:
  "请按以下结构回答：
   1. 项目立项时间和资助方
   2. 覆盖人群规模（数字×单位）
   3. 三个核心方法论及其出处
   每条都引用 source 标题"
```

也可用 "deep research" 风格：

```
"对该项目的 <主题> 方向做 5 段综合：
 段 1: 现状摘要
 段 2: 方法论比较
 段 3: 数据 / 案例
 段 4: 风险与局限
 段 5: 推荐后续动作
 每段至少 150 字。"
```

## 库 vs 云端 同步

### CLI 删了 notebook，MCP 库还在

**原因**：CLI 是云端真操作；MCP `add_notebook` 只是把 URL 加进本地元数据缓存。**两者不联动**。

**解**：`mcp__notebooklm__remove_notebook(id=...)` 手动清。

### MCP 工具消失 / 报错

```text
1. mcp__notebooklm__get_health
2. 不通 → setup_auth
3. 还不行 → cleanup_data(confirm=true, preserve_library=true) → setup_auth
4. 还不行 → 回 CLI
```

## 删除类

### 误删 notebook

**不可逆**。预防：

- 删前 `notebook list` 二次确认 ID
- 重要资料先用 merge 工作流搬到备份笔记本
- 永远不在脚本里并行 `nlm delete notebook`

### 误删 source

**不可逆**。预防：

- `source list` 拿到完整 ID（**别截断前 8 位** — 有些 ID 前 8 位撞了）
- 先备份 `nlm content source <id> --output /tmp/x.txt`
- 删完 `source list` 再核一次

## 配额类

### 每日 50 次问答上限

- 免费 Google 账号每日 ~50 次 NotebookLM 问答
- 超限常见错误：`Rate limit exceeded`
- 解法：换 Google 账号，或等第二天
- 长任务前先规划问题数量，避免乱问

## 调试类

### 看 Chrome 内部状态

```bash
# MCP 路径 — 真打开浏览器看
mcp__notebooklm__setup_auth(show_browser=true)
# 或 ask_question(browser_options.show=true)
```

### 看保存的 cookies（敏感！）

```bash
cat ~/.notebooklm-mcp-cli/auth.json | jq '.cookies | length'
# 含 40+ 个 cookies + CSRF token
```

不要把 `auth.json` 发给任何人或 commit 到 git。
