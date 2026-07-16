# notebooklm-bridge skill

让 Claude Code 直接调 Google NotebookLM 的工作流包。

## 一页上手

### 1. CLI 路径（推荐先装这个）

```bash
pip install notebooklm-mcp-cli
nlm login
nlm notebook list
```

第一次 `login` 会弹浏览器，登一次 Google 账号。完成后 cookies 自动落盘。
**注意**：NotebookLM 暂仅美区 + 每日免费 50 次问答。

### 2. MCP 路径（可选）

编辑 `~/.claude/settings.json`：

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

重启 Claude Code。能调出 `mcp__notebooklm__list_notebooks` 即通。

### 3. 验证

CLI：

```bash
nlm notebook list
```

MCP（在 Claude Code 会话里调）：

```text
mcp__notebooklm__get_health
```

## 文档导航

| 想做的事 | 看 |
|---|---|
| 了解 skill 整体设计 | [SKILL.md](SKILL.md) |
| CLI 命令清单 + 模板 | [references/cli.md](references/cli.md) |
| MCP 工具清单 + 配置 | [references/mcp.md](references/mcp.md) |
| 合并多个 notebook 的脚本 | [references/merge.md](references/merge.md) |
| 出错怎么办 | [references/troubleshooting.md](references/troubleshooting.md) |

## 卸载

```bash
pip uninstall notebooklm-mcp-cli
rm -rf ~/.notebooklm-mcp-cli/
```

MCP 那边只需从 `~/.claude/settings.json` 删掉 `mcpServers.notebooklm` 段。
