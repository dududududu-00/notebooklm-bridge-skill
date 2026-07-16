---
name: notebooklm-bridge
description: |
  把 Google NotebookLM 与 Claude Code 接通的工作流。
  支持 CLI（推荐，稳定）+ MCP（VSCode 集成）两条路径，
  覆盖：登录、列笔记本、读 source、跨笔记本合并、AI 综合问答。
  首次使用请先看「快速开始」完成认证（CLI 路径走 `nlm login`，MCP 路径走 `setup_auth`）。
metadata:
  version: "1.0.0"
---

# NotebookLM ↔ Claude Code 联通工作流

> **核心思路**：NotebookLM 把"上传的源文件 + Gemini 2.5 综合问答"封成一个笔记本。
> Claude Code 这边要么用 **CLI**（命令行直接调），要么用 **MCP**（原生工具暴露给会话）。
> 两条路功能一致，CLI 稳定可控，MCP 在编辑器里更顺手，二选一或并行都行。

---

## 快速开始

### 选接入方式

| 维度 | CLI | MCP |
|---|---|---|
| 安装 | `pip install notebooklm-mcp-cli` | 改 `~/.claude/settings.json` |
| 命令 | `nlm ...` | 工具名 `mcp__notebooklm__*` |
| 稳定性 | ✅ 高（脚本可控） | 依赖 session，不通时回 CLI |
| 适合场景 | 自动化、批处理、跨 notebook 合并 | 编辑器里随手问 |

**推荐**：CLI 为主（兜底），MCP 顺手开着（编辑器里直接调）。

### 前置

- Google 账号 1 个（**NotebookLM 暂仅限美区登录**，且免费版每日 50 次问答上限）
- Python 3.10+
- Chrome / Chromium（CLI 内部走 Playwright 调 Chrome）
- 已登录过一次后，cookie 会落盘到 `~/.notebooklm-mcp-cli/auth.json`（CLI 路径）或 MCP server 的浏览器配置目录

### CLI 路径：安装 + 登录

```bash
pip install notebooklm-mcp-cli

# 一键登录（会打开浏览器，让你登一次 Google）
nlm login

# 验证
nlm notebook list
```

### MCP 路径：settings.json

`~/.claude/settings.json` 加一段（用户名/路径按需替换）：

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

重启 Claude Code 让 MCP server 加载。验证：会话里能调出 `mcp__notebooklm__list_notebooks` 即通。

---

## 功能目录

| 功能 | 命令（CLI） | 工具（MCP） | 说明 |
|---|---|---|---|
| 列笔记本 | `nlm notebook list` | `mcp__notebooklm__list_notebooks` | 看 ID + 标题 + source 数 |
| 列 source | `nlm source list <nb_id>` | — | 看每份资料的标题和类型 |
| 读 source（原文） | `nlm source content <src_id>` | — | 原始文本，大文件要 pipe 给 head |
| 读 source（AI 总结） | `nlm source describe <src_id>` | — | NotebookLM 内置摘要 |
| AI 综合问答 | `nlm notebook query <nb_id> "问题" --json` | `mcp__notebooklm__ask_question` | Gemini 2.5 跨资料回答 |
| 加 source | `nlm source add <nb_id> --text "..." --title "..." --wait` | `mcp__notebooklm__add_source` | `--text` / `--file` / `--url` 三选一 |
| 删 source | `nlm delete source <src_id> -y` | — | 删前先拿完整 ID |
| 删 notebook | `nlm delete notebook <nb_id> -y` | — | 不可逆，一次只删一个 |
| 加入本地库 | — | `mcp__notebooklm__add_notebook` | MCP 库是元数据缓存，便于列和选 |
| 选 active | — | `mcp__notebooklm__select_notebook` | 后续 `ask_question` 默认笔记本 |

> 详细参数和示例见 [references/cli.md](references/cli.md)、[references/mcp.md](references/mcp.md)。

---

## 推荐工作流（避免 context 爆掉）

NotebookLM 一个笔记本可能塞几百份资料。**别直接读原始文本**，要走 AI 总结压缩：

```bash
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY all_proxy
NB=<notebook_id>

# 1) 总览 - 按阶段/类型归档
nlm notebook query $NB "请按主题/阶段归档这批资料，每类列出标题+200字概要" --json

# 2) 主题深入
nlm notebook query $NB "请深入总结 X 主题/阶段的：核心论点、关键数据、主要方法" --json

# 3) 跨主题综合
nlm notebook query $NB "请横向比较 A 和 B 两个方法的优劣、各自适用场景" --json
```

`--json` 输出可 pipe 给 Python 抽取 `answer` 字段，模板见 [references/troubleshooting.md](references/troubleshooting.md)。

---

## 跨笔记本合并（高级工作流）

> NotebookLM **没有**"跨笔记本复制 source ID"的接口。要把 A 的资料搬到 B，必须"提取-粘贴"两步走。

### 标准流程

1. **规划**
   - `nlm notebook list` + `nlm source list` 找重复资料
   - **同样文件名 ≠ 同一文件**（同一文件在源/目标 notebook 里 ID 不同）
   - 合并前先和用户确认：合并范围 / 原笔记本处理（保留/重命名/删除）/ 新名

2. **提取**
   - 用 Python 子进程批量调 `nlm content source <src_id> --output /tmp/<name>.txt`
   - **不要**用 zsh 的 `$(cat)` 传多行文本给命令行（会被拆成多参数）

3. **粘贴**
   - 用 Python `subprocess.run([...])` 顺序粘贴
   - **不要相信 returncode** — `nlm source add` 偶发 `returncode=1 + Could not add text source`，但实际可能已成功
   - **真正的成功标志**：stdout 含 `"Added source:"` 或 `"Source ID:"`
   - 失败要重试（可能第二次就成功）

4. **验证**（必须）
   - 按标题检查实际数量，**别只看 returncode**
   - 用 `Counter[s['title']]` 找重复

5. **清理**
   - `nlm delete source <id> -y`（先 `source list` 拿完整 ID，**别截断前 8 位**）
   - 保留最早添加的那一份

6. **删原 notebook**
   - `nlm delete notebook <id> -y`，一次只删一个（并行有竞态）
   - **删之前必须验证**，删除不可逆

完整脚本样例见 [references/merge.md](references/merge.md)。

---

## 已知坑

1. **网络代理必须 unset**：CLI/MCP 调 Google NotebookLM 前一定要：
   ```bash
   unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY all_proxy
   ```
   否则 SSL EOF / 连接被重置。

2. **大 source 不要直接 cat**：`nlm source content <id>` 可能 1+ MB。必须 pipe 给 `head -80` 或用 Read 工具的 `offset`/`limit`。

3. **~50 份后偶发失败**：单个 notebook 累积到 50 份左右，**新增可能偶发失败**（"Could not add text source"）。但失败可能是误报 — 重试 + 验证是必要的。

4. **AI 答案有幻觉风险**：NotebookLM 的 Gemini 2.5 答案基于上传源，但仍可能编造数字/引文。**重要结论必须打开对应 source 核对原文**（用 `nlm source content <id>`）。

5. **CLI / MCP 库是两套**：CLI 是真访问 Google NotebookLM；MCP 的 `add_notebook` 是本地元数据缓存（一份 URL 列表）。CLI 删了 notebook，MCP 库不会自动失效 — 需要手动 `remove_notebook`。

6. **MCP 工具偶发不可用**：会话里调 `mcp__notebooklm__*` 不见了 / 报错，多半是 MCP server 退出了。兜底用 CLI。

7. **删除不可逆**：删 source / notebook 前**永远先 `source list` 或 `notebook list` 验证要删的就是要删的**。

---

## 安装与给别人用

把这个目录拷贝到对方的 `~/.claude/skills/notebooklm-bridge/`（或任意名字），重启 Claude Code 即可识别为 skill。

最小化清单：

```
notebooklm-bridge/
├── SKILL.md              ← 本文件，主入口
├── README.md             ← 给对接人的 1 页快速说明（可选）
└── references/
    ├── cli.md
    ├── mcp.md
    ├── merge.md
    └── troubleshooting.md
```

---

## 相关链接

- NotebookLM 官方：https://notebooklm.google
- CLI 仓库：搜索 `notebooklm-mcp-cli`（PyPI）
- MCP 规范：https://modelcontextprotocol.io
