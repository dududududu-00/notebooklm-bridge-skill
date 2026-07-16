# 跨笔记本合并 — 脚本样例

> **核心事实**：`nlm source add` 不支持跨 notebook 传 source ID。要把 A 的资料搬到 B，必须：① 提取文本 → ② 粘贴为新 source。

## 第一步：列出候选

```bash
nlm notebook list --json > notebooks.json
nlm source list <nb_a> --json > sources_a.json
nlm source list <nb_b> --json > sources_b.json
```

## 第二步：识别重复（按标题）

```python
import json
with open("notebooks.json") as f: notebooks = json.load(f)
with open("sources_a.json") as f: sources_a = json.load(f)
with open("sources_b.json") as f: sources_b = json.load(f)

titles_a = {s["title"] for s in sources_a}
titles_b = {s["title"] for s in sources_b}
overlap = titles_a & titles_b
print(f"重复标题: {len(overlap)}")
print(overlap)
```

> ⚠️ **同名 ≠ 同文件**：同一文件在不同 notebook 里 source ID 不同。要"按内容指纹"严格去重，需 sha256 提取的文本（见下）。

## 第三步：提取-粘贴（Python 子进程）

```python
import subprocess, json, hashlib, os
from pathlib import Path

NB_DEST = "目标 notebook id"
TMP = Path("/tmp/notebooklm_merge")
TMP.mkdir(exist_ok=True)

# 收集 (src_id, title) 列表
items = []
for nb in (sources_a, sources_b):
    for s in nb:
        items.append((s["id"], s["title"]))

# 提取 + 算 hash 去重
seen_hash = {}
records = []
for src_id, title in items:
    out = TMP / f"{src_id[:8]}.txt"
    if not out.exists():
        subprocess.run(["nlm", "content", "source", src_id, "--output", str(out)],
                       check=False)
        # 这里我们不抛错 — 见下"成功判定"
    if not out.exists():
        continue
    text = out.read_text(errors="replace")
    h = hashlib.sha256(text.encode()).hexdigest()
    if h in seen_hash:
        records.append({"src_id": src_id, "title": title, "duplicate_of": seen_hash[h]})
        continue
    seen_hash[h] = src_id
    records.append({"src_id": src_id, "title": title, "path": str(out)})

# 粘贴（先写临时 *.txt 再用 --file，避免命令行参数长度问题）
ok, fail = [], []
for r in records:
    if "duplicate_of" in r:
        continue
    p = r["path"]
    cp = subprocess.run(
        ["nlm", "source", "add", NB_DEST, "--file", p, "--title", r["title"], "--wait"],
        capture_output=True, text=True
    )
    stdout, returncode = cp.stdout, cp.returncode
    # 真正的成功标志：stdout 含 "Added source:" 或 "Source ID:"
    success = ("Added source:" in stdout) or ("Source ID:" in stdout)
    if success:
        ok.append(r["title"])
    else:
        fail.append({"title": r["title"], "stdout": stdout[-300:], "code": returncode})

print(f"成功 {len(ok)}, 失败 {len(fail)}")
for f in fail:
    print(f"  ✗ {f['title']}\n    {f['stdout']}")
```

## 第四步：失败重试

`nlm source add` 偶发失败，**重试常常第二次就过**：

```python
for f in list(fail):
    rec = next(r for r in records if r["title"] == f["title"])
    cp = subprocess.run(
        ["nlm", "source", "add", NB_DEST, "--file", rec["path"], "--title", rec["title"], "--wait"],
        capture_output=True, text=True
    )
    if "Added source:" in cp.stdout or "Source ID:" in cp.stdout:
        print(f"  ✓ 重试成功: {f['title']}")
        fail.remove(f)
```

## 第五步：验证（必须）

```python
res = subprocess.run(["nlm", "source", "list", NB_DEST, "--json"],
                     capture_output=True, text=True)
all_titles = [s["title"] for s in json.loads(res.stdout)]

expected = {r["title"] for r in records if "duplicate_of" not in r}
missing = expected - set(all_titles)
extra   = set(all_titles) - expected
print(f"目标笔记本中: {len(all_titles)} 份")
print(f"期望缺失: {missing or '无'}")
print(f"意外多出: {extra or '无'}")
```

## 第六步：删 source（去重）

```python
from collections import Counter
c = Counter(all_titles)
for title, n in c.items():
    if n <= 1:
        continue
    # 同标题保留最早的一本，删其余
    ids_kept = []
    cp = subprocess.run(["nlm", "source", "list", NB_DEST, "--json"],
                        capture_output=True, text=True)
    for s in json.loads(cp.stdout):
        if s["title"] == title and s["id"] not in ids_kept:
            ids_kept.append(s["id"])
    for sid in ids_kept[1:]:
        subprocess.run(["nlm", "delete", "source", sid, "-y"], capture_output=True)
        print(f"  删重: {title} ({sid[:8]}...)")
```

## 第七步：删原 notebook（最后一步）

```bash
# 必须先验证（上面）再删！
nlm delete notebook <nb_a> -y
nlm delete notebook <nb_b> -y
```

> **一次只删一个**，并行删有竞态。

## 容量陷阱

- 单个 notebook 累积到 ~50 份后**偶发失败** — 重试是必要的
- 大文件（30-40 万字符）也能上传，但耗时长，建议 `--wait` 阻塞
- 一次合并 >100 份时考虑分批 + 中间验证
