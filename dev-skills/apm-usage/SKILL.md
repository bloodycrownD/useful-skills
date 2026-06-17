---
name: apm-usage
description: 指导 Agent 与开发者使用 APM CLI（init、read、role/persist/dynamic、kb 导入/索引/联想区、replace 局部更新、validate 干跑、stdin 管道写入）。在用户提到 apm、外置记忆、.apm、apm read、知识库、会话恢复或 Agent 初始化上下文时使用。
disable-model-invocation: true
---

# APM 使用指南

## 快速开始

```text
apm init
apm read                         # 会话开始必做
# … 执行任务 …
apm dynamic validate --stdin     # 可选：写入前干跑
apm dynamic write --stdin        # 推荐：多行正文用 stdin
apm persist write --stdin        # 可选：长期结论
```

- 在**项目根目录**（将创建或使用 `.apm/`）执行。
- 入口：`apm`（项目内通常需先 `npm run build` 或全局安装）。

## Agent 写入正文：优先 `--stdin`

| 场景 | 推荐方式 |
|------|----------|
| 多行 / 长正文 | **`--stdin`** 或管道（未传 `--text` 时自动读 stdin） |
| 单行短句 | `--text "…"` |
| 写入前检查长度 | `validate --stdin` 或 `validate --text` |

- **`write` / `validate` / `kb write` 均不强制 `--text`**；`--text` 与 `--stdin` **互斥**。
- **多行换行**：优先用真实换行经 stdin 传入，避免在 `--text` 里手工拼 `\n`。
- **无 `--file` 参数**：长文档写 `kb/docs/` 路径（Shell 重定向 stdin），或用 Agent 写文件工具后 `apm kb index rebuild`。

```bash
# bash：多行 dynamic（推荐）
cat <<'EOF' | apm dynamic write --stdin
任务：实现 foo
下一步：补测试
EOF

# bash：kb 单文件
apm kb write --path Iterations/foo/prd.md --stdin < prd.md

# PowerShell：从 UTF-8 文件读入（中文勿直接管道字符串，易乱码）
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
  kb/archive/          # dynamic 覆盖前自动归档
  kb/index/search.json.gz
```

`apm kb write --path` 的路径相对 `kb/docs/`（如 `Iterations/foo/prd.md`）。检索与联想中的路径相对 `kb/`（如 `docs/foo.md`、`archive/dynamic-....md`）。

## 默认长度上限（仅 max，无下限）

| 段 | config section | 默认 max |
|----|----------------|----------|
| 角色 | `role` | 100 |
| 持久记忆 | `persist` | 800 |
| 动态记忆 | `dynamicDetail` | 1500 |
| KB 动态 | `kbDynamicDetail` | 1500 |

- **无下限**：任意短文本（含 1 字、空串）均可写入。
- **仅上限**：超过 `max` 时默认报错（含 `got n, max m, need k fewer chars`）；可加 `--truncate` 截断后写入（stderr 会警告）。
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
apm role write --stdin              # 推荐多行
apm role write --text "单行短句"     # 可选
apm role validate --stdin
apm role replace --old "旧" --new "新"
apm persist write --stdin
apm dynamic write --stdin
apm dynamic replace --old "…" --new "…"
```

#### 正文输入：`--stdin`（推荐）、`--text`、管道

- **`--stdin`**：从标准输入读取全文（**多行正文首选**）；与 `--text` 互斥。
- **`--text <正文>`**：单行参数；支持转义 `\n` `\t` `\r` `\\`（短句可用，多行优先 stdin）。
- **管道**：未传 `--text` 且 stdin 非 TTY 时自动读 stdin，可省略 `--stdin` 标志。

#### `validate`（干跑，不落盘）

```bash
cat draft.md | apm dynamic validate
apm persist validate --stdin < draft.md
apm role validate --text "短草稿"
```

- 规则与 `write` 相同（仅检查 max）；成功输出 `OK: <当前长度>/<max>`。
- 不写盘、不归档、不触发索引重建。

#### `--truncate`（超长截断写入）

```bash
apm role write --stdin --truncate < long.txt
apm role replace --old "旧" --new "新" --truncate
```

- 正文超过 `max` 时截断至 `max` 后写入；stderr 输出 `Warning: … truncated …`。
- 未加 `--truncate` 时超长仍报错。

#### `replace` 其他说明

- `replace`：`--old` 须与 `show`/`read` 正文**原样**匹配；默认只换第一次，全换加 `--all`。
- 全量覆盖用 `write`；局部更新用 `replace`。
- 写入时不要手写 YAML front matter。
- `--old` / `--new` 仍用 `--text`（本需求不扩展 stdin 到 replace 参数）。

### `dynamic` 与归档

| 命令 | 行为 |
|------|------|
| `dynamic write` | 当前正文非空时先写入 `kb/archive/dynamic-<时间戳>.md`，再覆盖 `memory/dynamic.md` |
| `dynamic write --text ""` | 清空 dynamic（非空则先归档） |
| `dynamic replace` | 只改正文，不归档 |
| `dynamic validate` | 仅校验长度，不归档、不写盘 |

### 知识库

```bash
apm kb import --from <目录>
apm kb write --path <path.md> --stdin     # 推荐
apm kb write --path <path.md> --text "…"  # 短内容可选
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

**切换任务（多行，推荐 stdin）：**

```bash
cat <<'EOF' | apm dynamic write --stdin
任务：…
下一步：…
EOF
```

**写入知识库单文件：**

```bash
apm kb write --path Iterations/<名>/prd.md --stdin < prd.md
apm kb index rebuild
apm read
```

**写入前干跑：**

```bash
cat draft.md | apm dynamic validate
# OK: n/max 后再 write
cat draft.md | apm dynamic write --stdin
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
| 长度报错 `got … max … need … fewer` | 缩短正文、`config set --max` 调大，或 `write --truncate` |
| `\n` 未换行 | **改用 `--stdin` 传真实换行**；短句才用 `--text` + `\n` 转义 |
| PowerShell 管道中文乱码 | 用 `Get-Content -Encoding UTF8 -Raw` 重定向，勿直接 `$str \| apm …` |
| `Cannot use both --text and --stdin` | 只选其一 |
