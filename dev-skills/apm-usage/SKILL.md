---
name: apm-usage
description: APM CLI 与外置记忆语义规范：persist（跨会话规则，类 AGENTS.md）、dynamic（背景/目的/现状）、role、kb search/index、read/write/validate。在提到 apm、.apm、外置记忆、会话恢复，或其它 skill 需更新 dynamic/persist 时使用。本文件为记忆「写什么」的唯一权威；其它 skill 不得另立字段表。
disable-model-invocation: true
---

# APM 使用指南

对应 CLI：`agent-persistence-memory`（`apm`）≥ 1.0.0。

## 快速开始

```bash
apm init
apm read                         # 会话开始必做
# … 执行任务 …

# 多行写入（推荐 heredoc + --stdin）；dynamic 须含 背景/目的/现状
cat <<'EOF' | apm dynamic validate --stdin   # 可选：写入前干跑
## 背景
…

## 目的
…

## 现状
…
EOF
cat <<'EOF' | apm dynamic write --stdin
## 背景
…

## 目的
…

## 现状
…
EOF
```

- 在**项目根目录**（将创建或使用 `.apm/`）执行。
- 入口：`apm`（或 `npx apm`；源码仓库需先 `npm run build`）。
- **工作区自动补齐**：任意命令首次执行时会创建或补全 `.apm/`；`apm init` 等价且幂等。废弃的 `.apm/dynamic/` 与 `kb/dynamic/` 若存在会被自动删除。

---

## 记忆语义（权威规范）

其它 dev-skill 更新 `dynamic` / `persist` 时 **一律遵守本节**；不得再自造「阶段字段表」覆盖本节。CLI 细节见后文。

### 三段分工

| 段 | 是什么 | 不是什么 |
|----|--------|----------|
| **role** | Agent 身份/口吻 | 任务进度 |
| **persist** | 跨会话仍有效的**持久规则**，写法接近 `AGENTS.md`：约定、边界、已拍板决策、踩坑后的行为约束 | 当前任务进度、DAG、探索流水、验证日志 |
| **dynamic** | **当前任务**的工作记忆；固定用「背景 / 目的 / 现状」三节 | 机器状态库、节点表、wave 计划全文 |

### `dynamic`：背景 · 目的 · 现状

每次 `dynamic write` 用**全量覆盖**，正文须含这三节（可用 `##` 标题）：

```markdown
## 背景
用户原话/命令摘要；相关 PRD/SPEC（或需求）路径；为何开这个任务。

## 目的
本轮要交付什么；验收大致长什么样；当前阶段终点（如「PRD 待确认」「dev-ready」）。

## 现状
做到哪了；卡在哪；解决思路与下一步（人话，短句即可）。
```

| 宜写入 | 不宜写入 |
|--------|----------|
| 用户指令要点、任务目标、关键文档路径 | DAG 节点表 / 边 / `wave_plan` / `dag_version` / `node_status` |
| 当前阶段结论一句话、阻塞原因、下一步思路 | 探索报告全文、verify 命令流水、commit 列表 |
| 确认态可用一句写在「现状」（如「用户已确认 spec」） | 把 `prd_confirmed` 等当 schema 堆字段 |

阶段推进时：**重写三节**，不要把过程日志往下追加。已无 `replace` 子命令——局部改也须 `write` 全量覆盖。

### `persist`：持久规则（类 AGENTS.md）

只写**换会话仍该遵守**的内容，例如：

- 项目术语、模块边界、命名约定
- 用户拍板且跨任务仍有效的决策（「X 模块不负责 Y」）
- 协作/实现约束（「blocking 步骤必须有对应测试 id」）

**不要**把「本迭代做到哪一步」「某次 verify 过了没」写进 persist。路径类信息若只服务当前任务，放 `dynamic`「背景」即可；仅当路径已成为团队约定时才进 persist。

有新规则用 `write` 全量整理；无新规则则**本阶段可不碰 persist**。

### 机器状态放哪（禁止塞进记忆槽）

下列内容属于**工作流状态**，写到 Context Bundle、`docs/.iteration-state.yaml`、对话内 YAML，或 kb 文档——**不要**写入 `dynamic` / `persist`：

- 开发 / 审查 DAG：`wave_plan`、节点表、边、`dag_version`、`node_status`
- 轮次计数、open must-fix 清单、doc_fix_plan / spec_fix_plan
- Bundle-full / Bundle-delta 大段 YAML
- 探索报告原文、验证输出原文

`apm read` 的记忆槽给**人读、接续任务**用；状态机给编排逻辑用。二者分开。

### 与其它 skill 的关系

- 本文件 = `dynamic` / `persist` **写什么**的唯一权威。
- 其它 skill 阶段结束时：按本节刷新 `dynamic` 三节；**仅当**出现可跨会话复用的规则时才更新 `persist`。
- 无 APM 时：可用 `docs/.iteration-state.yaml` 或对话内等价维护「背景 / 目的 / 现状」与规则条文，语义相同。

---

## Agent 写入正文：多行优先 heredoc + `--stdin`

| 场景 | 推荐方式 |
|------|----------|
| 多行 / 长正文（记忆段） | **bash heredoc** 管道到 `--stdin` |
| PowerShell 多行 | **here-string** `@'…'@` 管道到 `--stdin`（中文勿直接 `$str \| apm`） |
| 已有文件作记忆正文 | 重定向或 `Get-Content -Raw -Encoding UTF8` 管道 |
| 单行短句 | `--text "…"` |
| 写入前检查长度 | 同样用 heredoc / 文件管道到 `validate --stdin` |
| 知识库 `.md` | **直接写文件**到 `.apm/kb/docs/`，再 `apm kb index rebuild`（**无** `kb write` / `kb import`） |

- **`write` / `validate` 均不强制 `--text`**；`--text` 与 `--stdin` **互斥**。
- **不要**在 `--text` 里手工拼多行 `\n`；heredoc / here-string 传**真实换行**。
- **无 `--file`、无 `replace`、无 `kb write`**。

```bash
# bash：多行 dynamic（Agent 首选；须含 背景/目的/现状）
cat <<'EOF' | apm dynamic write --stdin
## 背景
用户要求实现 foo；SPEC：Iterations/foo/spec.md

## 目的
按 spec 完成 foo 并跑通相关测试。

## 现状
核心逻辑已合入；下一步补单元测试。
EOF

# bash：写入前干跑
cat <<'EOF' | apm dynamic validate --stdin
…
EOF

# PowerShell：here-string
@'
## 背景
…
## 目的
…
## 现状
…
'@ | apm dynamic write --stdin

# PowerShell：从 UTF-8 文件读入记忆段
Get-Content .\draft.md -Raw -Encoding UTF8 | apm dynamic write --stdin
```

## 工作区

```
.apm/
  config.json          # 各段 max 上限；initializedAt / updatedAt / lastReadAt
  memory/role.md       # 角色
  memory/persist.md    # 持久规则（类 AGENTS.md）
  memory/dynamic.md    # 当前任务：背景 / 目的 / 现状
  kb/docs/             # 知识库 .md（可嵌套；直接写文件）
  kb/archive/          # memory 三段 write 时写入的分层快照
  kb/index/search.json.gz
```

知识库文档写入 `.apm/kb/docs/`（如 `Iterations/foo/prd.md`），再 `apm kb index rebuild`。检索与联想中的路径相对 `kb/`（如 `docs/foo.md`、`archive/2026/06/18/dynamic/143052127.md`；旧版扁平 `archive/dynamic-….md` 仍可被索引）。

**已废弃（存在则 CLI 自动删除）**：`.apm/dynamic/`、`.apm/kb/dynamic/`。

## 默认长度上限（仅 max，无下限）

| 段 | config section | 默认 max |
|----|----------------|----------|
| 角色 | `role` | 100 |
| 持久记忆 | `persist` | 800 |
| 动态记忆 | `dynamicDetail` | 1500 |

- **无下限**：任意短文本（含 1 字、空串）均可写入。
- **仅上限**：超过 `max` 时拒绝写入，报错含 `got n, max m, need k fewer chars`。
- `kb/docs/` 下文件**不受**上述 max 限制（直接写文件，不经 CLI write）。

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

`show` · `write` · `validate`（**无** `replace`）

```bash
apm role show
# 多行：cat <<'EOF' | apm role write --stdin
apm role write --text "单行短句"
# 多行：cat <<'EOF' | apm persist write --stdin
# 多行：cat <<'EOF' | apm dynamic write --stdin
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

#### 写入说明

- 全量覆盖用 `write`；局部更新也须读出后改完整正文再 `write`（无 `replace`）。
- 写入时不要手写 YAML front matter；若磁盘上 FM 损坏/缺失，`write` 会自愈。

### memory 三段 write 与 archive 快照

每次 `role` / `persist` / `dynamic` 的 **`write`** 会将**本次落盘全文**（含 YAML front matter）同时写入目标文件与分层 archive 快照；快照与目标文件**完全相同**（存新版，非覆盖前的旧版）。

| 路径模式（相对 `kb/`） | 说明 |
|------------------------|------|
| `archive/{yyyy}/{MM}/{dd}/role/{HHmmssSSS}.md` | role write 快照 |
| `archive/{yyyy}/{MM}/{dd}/persist/{HHmmssSSS}.md` | persist write 快照 |
| `archive/{yyyy}/{MM}/{dd}/dynamic/{HHmmssSSS}.md` | dynamic write 快照 |

| 命令 | archive 快照 |
|------|--------------|
| `role` / `persist` / `dynamic` **`write`** | 每次 +1 条分层快照 |
| `dynamic write --text ""` | 目标变为空模板，仍 +1 条空模板快照 |
| `validate` | **不**写盘、不归档 |

### 知识库

```bash
apm kb search --q "<查询>"
apm kb index rebuild
```

- **无** `kb write` / `kb import` / `kb dynamic`：知识库正文直接编辑/复制到 `.apm/kb/docs/**/*.md`，再 `rebuild`。
- 可用 Agent 写文件工具落盘 PRD/SPEC，然后 `apm kb index rebuild`。

| 操作 | 自动 `kb index rebuild` |
|------|-------------------------|
| `role` / `persist` / `dynamic` 的 write | 是 |
| 直接写入/移动 `kb/docs/` | 否（须手动 `rebuild`） |
| `validate`（各段） | 否 |

### 配置

```bash
apm config show
apm config set --section role|persist|dynamicDetail --max <n>
```

- 各段 limits **仅含 `max`**；旧 config 中的 `min`、`kbDynamicDetail` 已移除/忽略。
- `config set` 写回时只持久化 `{ "max": n }`。

## 典型场景

**恢复会话：** `apm read` → 读 `persist`（规则）与 `dynamic`（背景/目的/现状）及联想区 → 按「现状」继续；勿把 archive / Bundle 当记忆正文重写一遍。

**切换或推进任务（重写 dynamic 三节）：**

```bash
cat <<'EOF' | apm dynamic write --stdin
## 背景
…

## 目的
…

## 现状
…
EOF
```

**写入跨会话规则（persist）：** 仅在有新约定/拍板决策时；正文宜短、像规则列表，勿贴任务流水。

**写入知识库单文件：**

```bash
# 直接写到 .apm/kb/docs/（Agent 写文件工具或编辑器）
# 例：.apm/kb/docs/Iterations/<名>/prd.md
apm kb index rebuild
apm read
```

**写入前干跑：**

```bash
cat <<'EOF' | apm dynamic validate --stdin
## 背景
…
## 目的
…
## 现状
…
EOF
# OK: n/max 后再 write（同一 heredoc 正文）
cat <<'EOF' | apm dynamic write --stdin
## 背景
…
## 目的
…
## 现状
…
EOF
```

## 勿混淆的路径

| 路径 | 用途 |
|------|------|
| `.apm/memory/` | CLI 外置记忆，`apm read` 使用 |
| `.apm/dynamic/`（根下） | **已废弃**；存在时会被 CLI 自动删除 |
| `kb/dynamic/` | **已废弃**；存在时会被 CLI 自动删除 |
| 仓库内其他 `memory/` | 不参与 `apm read`，不要当作 `.apm` 使用 |

## 故障排查

| 现象 | 处理 |
|------|------|
| `Old .apm layout detected` | 备份后删除/替换旧 `.apm`，再 `apm init`（不支持自动迁移） |
| `Knowledge index missing` | `apm kb index rebuild` |
| 联想区无结果 | 记忆与 kb 有共同词；`rebuild` |
| 改完 `kb/docs/` 搜不到 | `apm kb index rebuild` |
| 长度报错 `got … max … need … fewer` | 缩短正文，或 `config set --max` 调大；可先 `validate` 干跑 |
| `\n` 未换行 | **改用 heredoc / here-string 传真实换行**；短句才用 `--text` + `\n` |
| PowerShell 管道中文乱码 | 用 `@'…'@` here-string 或 `Get-Content -Encoding UTF8 -Raw`，勿直接 `$str \| apm …` |
| `Cannot use both --text and --stdin` | 只选其一 |

## 已移除（勿再调用）

| 旧能力 | 替代 |
|--------|------|
| `role` / `persist` / `dynamic` **`replace`** | `show` 后改全文，再 `write` |
| `kb write` / `kb import` | 直接写 `.apm/kb/docs/`，再 `kb index rebuild` |
| `kb dynamic` / `kbDynamicDetail` | 任务记忆只用 `memory/dynamic.md` |
