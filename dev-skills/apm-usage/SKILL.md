---
name: apm-usage
description: 指导 Agent 与开发者使用 APM CLI（init、read、role/persist/dynamic、kb 导入/索引/联想区、replace 局部更新、validate 干跑、heredoc/stdin 多行写入）。在用户提到 apm、外置记忆、.apm、apm read、知识库、会话恢复或 Agent 初始化上下文时使用。
disable-model-invocation: true
---

# APM 使用指南

## 快速开始

```bash
apm init
apm read                         # 会话开始必做
# … 执行任务 …

# 多行写入（推荐 heredoc + --stdin）
cat <<'EOF' | apm dynamic validate --stdin   # 可选：写入前干跑
任务：…
下一步：…
EOF
cat <<'EOF' | apm dynamic write --stdin
任务：…
下一步：…
EOF
```

- 在**项目根目录**（将创建或使用 `.apm/`）执行。
- 入口：`apm`（项目内通常需先 `npm run build` 或全局安装）。

## Agent 写入正文：多行优先 heredoc + `--stdin`

| 场景 | 推荐方式 |
|------|----------|
| 多行 / 长正文 | **bash heredoc** 管道到 `--stdin`（见下方示例） |
| PowerShell 多行 | **here-string** `@'…'@` 管道到 `--stdin`（中文勿直接 `$str \| apm`） |
| 已有文件 | 重定向或 `Get-Content -Raw -Encoding UTF8` 管道 |
| 单行短句 | `--text "…"` |
| 写入前检查长度 | 同样用 heredoc / 文件管道到 `validate --stdin` |

- **`write` / `validate` / `kb write` 均不强制 `--text`**；`--text` 与 `--stdin` **互斥**。
- **不要**在 `--text` 里手工拼多行 `\n`；heredoc / here-string 传**真实换行**。
- **无 `--file` 参数**：长文档写 `kb/docs/` 路径（heredoc 或重定向 stdin），或用 Agent 写文件工具后 `apm kb index rebuild`。

```bash
# bash：多行 dynamic（Agent 首选）
cat <<'EOF' | apm dynamic write --stdin
任务：实现 foo
下一步：补测试
EOF

# bash：写入前干跑（同一正文先 validate 再 write）
cat <<'EOF' | apm dynamic validate --stdin
…
EOF

# bash：kb 单文件（已有文件时）
apm kb write --path Iterations/foo/prd.md --stdin < prd.md

# PowerShell：here-string（多行、含中文）
@'
任务：实现 foo
下一步：补测试
'@ | apm dynamic write --stdin

# PowerShell：从 UTF-8 文件读入
Get-Content .\draft.md -Raw -Encoding UTF8 | apm dynamic write --stdin
Get-Content .\prd.md -Raw -Encoding UTF8 | apm kb write --path Iterations/foo/prd.md --stdin
```

## 工作区

```
.apm/
  config.json          # 各段 max 上限；initializedAt / updatedAt / lastReadAt
  memory/role.md       # 角色
  memory/persist.md    # 持久记忆
  memory/dynamic.md    # 动态记忆（当前任务）
  kb/docs/             # 知识库 .md（可嵌套）
  kb/dynamic/detail.md
  kb/archive/          # memory 三段 write 时写入的分层快照（见下文）
  kb/index/search.json.gz
```

`apm kb write --path` 的路径相对 `kb/docs/`（如 `Iterations/foo/prd.md`）。检索与联想中的路径相对 `kb/`（如 `docs/foo.md`、`archive/2026/06/18/dynamic/143052127.md`；旧版扁平 `archive/dynamic-....md` 仍可被索引）。

## 默认长度上限（仅 max，无下限）

| 段 | config section | 默认 max |
|----|----------------|----------|
| 角色 | `role` | 100 |
| 持久记忆 | `persist` | 800 |
| 动态记忆 | `dynamicDetail` | 1500 |
| KB 动态 | `kbDynamicDetail` | 1500 |

- **无下限**：任意短文本（含 1 字、空串）均可写入。
- **仅上限**：超过 `max` 时拒绝写入，报错含 `got n, max m, need k fewer chars`。
- `kb write` **不受**上述 max 限制。

## `apm read` 输出

无内容的段会省略。顺序：

1. `# 角色`、`# 持久记忆`、`# 动态记忆`（正文已去 YAML front matter）
2. `# 联想区`（用三段记忆正文检索 `kb/`，不含 `index/`）

| 联想区 | 说明 |
|--------|------|
| 详细区 | ≤5 条；`[匹配率%] 路径 关键词：…` + ≤3 行 `行号\|正文`（超 120 字截断）；条间空一行 |
| 简略区 | ≤10 条；仅头部；条间无空行；与详细区间空一行 |
| 无命中 | 不输出联想区 |
| 无索引 | 输出提示执行 `apm kb index rebuild` |

## 命令

### 记忆：`role` | `persist` | `dynamic`

`show` · `write` · `validate` · `replace --old <原文> --new <新文> [--all]`

```bash
apm role show
# 多行：cat <<'EOF' | apm role write --stdin
apm role write --text "单行短句"     # 仅短句
apm role replace --old "旧" --new "新"
# 多行：cat <<'EOF' | apm persist write --stdin
# 多行：cat <<'EOF' | apm dynamic write --stdin
apm dynamic replace --old "…" --new "…"
```

#### 正文输入：heredoc + `--stdin`（推荐）、`--text`、管道

- **多行首选**：`cat <<'EOF' | apm <段> write --stdin`（bash）；PowerShell 用 `@'…'@ | apm … --stdin`。
- **`--stdin`**：从标准输入读取全文；与 `--text` 互斥。
- **`--text <正文>`**：单行参数；支持转义 `\n` `\t` `\r` `\\`（短句可用）。
- **管道**：未传 `--text` 且 stdin 非 TTY 时自动读 stdin，可省略 `--stdin` 标志。

#### `validate`（干跑，不落盘）

```bash
cat <<'EOF' | apm dynamic validate --stdin
草稿正文
EOF
apm persist validate --stdin < draft.md
apm role validate --text "短草稿"
```

- 规则与 `write` 相同（仅检查 max）；成功输出 `OK: <当前长度>/<max>`。
- 不写盘、不归档、不触发索引重建。

#### `replace` 其他说明

- `replace`：`--old` 须与 `show`/`read` 正文**原样**匹配；默认只换第一次，全换加 `--all`。
- `replace` **不**写入 archive 快照（仅 `write` 触发快照）。
- 全量覆盖用 `write`；局部更新用 `replace`。
- 写入时不要手写 YAML front matter。
- `--old` / `--new` 仍用 `--text`（不扩展 stdin 到 replace 参数）。

### memory 三段 write 与 archive 快照

每次 `role` / `persist` / `dynamic` 的 **`write`** 会将**本次落盘全文**（含 YAML front matter）同时写入目标文件与分层 archive 快照；快照内容与目标文件**完全相同**（存新版，非覆盖前的旧版）。

| 路径模式（相对 `kb/`） | 说明 |
|------------------------|------|
| `archive/{yyyy}/{MM}/{dd}/role/{HHmmssSSS}.md` | role write 快照 |
| `archive/{yyyy}/{MM}/{dd}/persist/{HHmmssSSS}.md` | persist write 快照 |
| `archive/{yyyy}/{MM}/{dd}/dynamic/{HHmmssSSS}.md` | dynamic write 快照 |

| 命令 | archive 快照 |
|------|--------------|
| `role` / `persist` / `dynamic` **`write`** | 每次 +1 条分层快照 |
| `dynamic write --text ""` | 目标变为空模板，仍 +1 条空模板快照 |
| `replace` | **不**新增快照 |
| `validate` | **不**写盘、不归档 |
| `kb dynamic write` | **不**写入 `archive/`（仅更新 `kb/dynamic/detail.md`） |

### 知识库

```bash
apm kb import --from <目录>
# 多行：cat <<'EOF' | apm kb write --path <path.md> --stdin
apm kb write --path <path.md> --text "…"  # 短内容
apm kb search --q "<查询>"
apm kb index rebuild
apm kb dynamic show|write|validate|replace
```

| 操作 | 自动 `kb index rebuild` |
|------|-------------------------|
| `role` / `persist` / `dynamic` 的 write、replace | 是 |
| `dynamic write` | 是 |
| `kb import` | 是 |
| `kb write`、`kb dynamic` 的 write/replace | 否 |
| `validate`（各段） | 否 |

### 配置

```bash
apm config show
apm config set --section role|persist|dynamicDetail|kbDynamicDetail --max <n>
```

- 各段 limits **仅含 `max`**；旧 config 中的 `min` 读取时忽略。

## 典型场景

**恢复会话：** `apm read` → 读动态记忆与联想区 → 继续任务。

**切换任务（多行，heredoc）：**

```bash
cat <<'EOF' | apm dynamic write --stdin
任务：…
下一步：…
EOF
```

**写入知识库单文件：**

```bash
cat <<'EOF' | apm kb write --path Iterations/<名>/prd.md --stdin
# PRD 正文
EOF
apm kb index rebuild
apm read
```

**写入前干跑：**

```bash
cat <<'EOF' | apm dynamic validate --stdin
草稿
EOF
# OK: n/max 后再 write（同一 heredoc 正文）
cat <<'EOF' | apm dynamic write --stdin
草稿
EOF
```

**批量导入：** `apm kb import --from docs`（导入后自动 rebuild）。

## 勿混淆的路径

| 路径 | 用途 |
|------|------|
| `.apm/memory/` | CLI 外置记忆，`apm read` 使用 |
| 仓库内其他 `memory/` | 不参与 `apm read`，不要当作 `.apm` 使用 |

## 故障排查

| 现象 | 处理 |
|------|------|
| `Incomplete .apm workspace` | `apm init` |
| `Knowledge index missing` | `apm kb index rebuild` |
| 联想区无结果 | 记忆与 kb 有共同词；`rebuild` |
| `kb write` 后搜不到 | `apm kb index rebuild` |
| 长度报错 `got … max … need … fewer` | 缩短正文，或 `config set --max` 调大；可先 `validate` 干跑 |
| `\n` 未换行 | **改用 heredoc / here-string 传真实换行**；短句才用 `--text` + `\n` |
| PowerShell 管道中文乱码 | 用 `@'…'@` here-string 或 `Get-Content -Encoding UTF8 -Raw`，勿直接 `$str \| apm …` |
| `Cannot use both --text and --stdin` | 只选其一 |
