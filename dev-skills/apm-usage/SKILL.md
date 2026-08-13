---
name: apm-usage
description: APM（Agent Persistence Memory）使用指南：用 apm init 初始化、apm read 读取规则与最近记忆并触发联想、apm search 实时检索 docs/ 下所有 .md。记忆由 agent 直接写文件到 docs/apm/memory/，不通过 APM 命令；持久规则维护在 docs/apm/RULE.md。在提到 apm、外置记忆、会话恢复、记忆检索，或需要按统一约定记录对话记忆时使用。本文件为记忆「写什么、怎么读」的唯一权威，其它 skill 不得另立字段表或目录约定。
disable-model-invocation: true
---

# APM 使用指南

对应 CLI：`agent-persistence-memory`（`apm`）。

APM 现在只做三件事：初始化目录、读取规则与记忆、实时搜索文档。记忆文件由 agent 直接写到磁盘上，不走 APM 命令。整体围绕一个 `docs/apm/` 目录展开，不再有 `.apm/` 运行态目录。

## 快速开始

```bash
apm init                # 在项目根初始化 docs/apm/（幂等，已存在不报错）
apm read                # 会话开始必做：规则区 + 最近记忆 + 联想区
# … 执行任务 …

# 记录一条新记忆：agent 直接写文件到 docs/apm/memory/，不走 APM 命令
#   文件名：yyyyMMdd-简短标识.md，例如 docs/apm/memory/20260814-foo-done.md
```

- 在**项目根目录**执行命令；`docs/apm/` 就建在这里。
- 入口：`apm`（或 `npx apm`；源码仓库需先 `npm run build`）。
- 记忆文件由 agent 用写文件工具直接落盘，APM 不提供「写入」命令。

---

## 记忆语义

APM 把记忆分成两层，两层都落在 `docs/apm/` 里，跟着项目一起进版本控制。

### RULE.md：持久规则（类 AGENTS.md）

`docs/apm/RULE.md` 装的是**换会话仍该遵守**的内容，写法和 `AGENTS.md` 一样，偏约定、边界、已拍板的决策。适合写：

- 项目术语、模块边界、命名约定
- 用户拍板且跨任务仍有效的决策（「X 模块不负责 Y」）
- 协作/实现约束（「blocking 步骤必须有对应测试 id」）

**不要**把「本会话做到哪一步」「某次验证过了没」写进 RULE.md——那是具体对话记忆的范畴。路径类信息如果只是服务当前任务，写成一条记忆就好；只有当某条路径已经是团队约定时，才值得进 RULE.md。

RULE.md 由 agent 直接维护：有新规则就整理进去，没有就别动它。

### memory 文件：带时间的对话记忆

`docs/apm/memory/` 下的每个 `.md` 文件是一条**对话记忆**，文件名形如 `yyyyMMdd-简短标识.md`。它记录的是某次具体的交流——用户问了什么、我回了什么、当时的关键结论是什么。文件靠 front matter 的 `date` 排序，`apm read` 会取最近的几条做摘要。

记忆文件适合写：

- 某次讨论的关键问答与结论
- 用户透露的偏好、踩过的坑、当时的决策理由
- 当前任务的阶段性进展（作为事后回顾，而不是工作流状态）

不要把 DAG 节点表、wave 计划、验证日志这类**工作流状态**塞进记忆文件，那些东西属于 Context Bundle 或其它编排文件。

---

## 目录结构

```
docs/
  apm/
    RULE.md          # 持久规则，agent 直接维护
    memory/          # 对话记忆目录
      yyyyMMdd-name.md
      .gitkeep       # apm init 创建，保证空目录能进版本控制
```

- `docs/apm/RULE.md`：持久规则，`apm read` 的规则区会原样输出它。
- `docs/apm/memory/*.md`：每条一个文件，`apm read` 取最近 5 条做摘要，`apm search` 实时扫描它们。
- 不再有 `.apm/` 目录，也不再有 role/persist/dynamic 之分；持久规则统一在 `RULE.md`，对话记忆统一在 `memory/`。
- `apm read` 的联想区和 `apm search` 都扫描**整个 `docs/`**（不只是 `docs/apm/`），所以项目里其它 `.md` 文档同样会被检索到。

---

## 命令详解

只有三个命令：`apm init`、`apm read`、`apm search`。没有 role/persist/dynamic/write/show/index/config，也不再需要预先建索引。

### `apm init`

幂等地初始化记忆目录：

```bash
apm init
```

它会创建：

- `docs/apm/RULE.md`（若不存在，给一个可写的初始模板）
- `docs/apm/memory/.gitkeep`

已经存在的文件不会被覆盖，重复执行也不报错。新项目第一次接入 APM 时跑一次即可。

### `apm read`

无参数。会话开始时执行一次，拿到三段上下文：

1. **规则区**：`docs/apm/RULE.md` 的全文。
2. **最近记忆区**：`docs/apm/memory/` 下最近 5 条记忆的摘要，按 front matter 的 `date` 降序排列。每条给标题、日期、关键词、摘要。
3. **联想区**：以 `RULE.md` 的内容作为查询上下文，**实时**搜索整个 `docs/` 目录，返回相关片段。

三段之间会用标题分隔。规则区给的是「该遵守什么」，最近记忆区给的是「最近聊过什么」，联想区给的是「项目里还有哪些文档和当前规则相关」。

联想区每次都实时扫描 `docs/` 下所有 `.md` 文件并在内存里建索引，不需要预先 `index build`。改了 `docs/` 下的文件，下一次 `apm read` 立即生效。

### `apm search`

实时检索整个 `docs/` 目录下的所有 `.md` 文件：

```bash
apm search "查询内容"
apm search "查询内容" --page 2
apm search "查询内容" --num 20
```

参数说明：

| 参数 | 含义 | 默认 |
|------|------|------|
| `<query>` | 查询关键词（位置参数） | 必填 |
| `--page N` | 第几页，从 1 开始 | 1 |
| `--num N` | 每页返回多少条 | 10 |

输出每条命中包含三部分：

- **匹配片段**：命中的原文片段（带行号或上下文）
- **匹配度**：该条结果与查询的相关度评分
- **路径**：命中文件相对项目根的路径（如 `docs/apm/memory/20260814-foo-done.md`）

同样每次实时扫描建索引，不需要预建索引，也不依赖 `.apm/config/index.gz` 之类的产物。

---

## 记忆文件格式

每个 `docs/apm/memory/yyyyMMdd-name.md` 文件由两部分组成：YAML front matter 和正文。

### front matter 字段

```yaml
---
date: yyyy-MM-dd HH:mm
title: 记忆标题
keywords: 关键词1, 关键词2
abstract: 摘要内容
---
```

| 字段 | 说明 |
|------|------|
| `date` | 这条记忆的时间，格式 `yyyy-MM-dd HH:mm`（24 小时制）。`apm read` 按它降序排最近记忆 |
| `title` | 一句话标题，出现在最近记忆区 |
| `keywords` | 逗号分隔的关键词，便于检索与摘要 |
| `abstract` | 摘要，两三句话讲清这条记忆讲了什么 |

front matter 必须放在文件最前面，用 `---` 包起来。

### 正文：user / assistant

front matter 之后是对话正文，用 `user:` 和 `assistant:` 两段：

```markdown
user:
用户当时的原话或意图摘要。

assistant:
我当时的回复要点、给出的方案、达成的结论。
```

正文不必逐字记全，挑关键的就够了：用户问了什么、我回了什么、结论是什么。啰嗦的过程流水可以省略。

### 完整示例

```markdown
---
date: 2026-08-14 15:30
title: foo 模块用 X 方案替代 Y
keywords: foo, 架构决策, Y 已废弃
abstract: 用户拍板 foo 模块改用 X 方案，Y 方案不再维护，新代码不要再依赖 Y。
---

user:
foo 模块后面是不是就不走 Y 了？新功能我都按 X 写？

assistant:
对，Y 已经废弃了。新功能一律按 X 方案写；老代码迁移可以分批做，不阻塞新需求。RULE.md 里我也补了一条约束，后面别再引入对 Y 的新依赖。
```

---

## 记忆写入方式

**记忆文件由 agent 直接写到磁盘，不通过 APM 命令。** APM 不提供「write」命令，也没有 `--stdin` / `--text` 之类的入参。

写一条新记忆的步骤：

1. 在 `docs/apm/memory/` 下新建文件，命名 `yyyyMMdd-简短标识.md`。
2. 按上面「记忆文件格式」写好 front matter（`date` / `title` / `keywords` / `abstract`）和 `user:` / `assistant:` 正文。
3. 用 agent 的写文件工具落盘。完成。

下一次 `apm read` 就会自动把它纳入最近记忆区，`apm search` 也能搜到它，不需要任何额外的「index build」或「refresh」步骤。

`RULE.md` 也是同理：要加规则就直接编辑这个文件，存盘即生效。

---

## 典型场景

**会话初始化：** 进项目第一件事 `apm read`，拿到规则区（RULE.md 全文）、最近 5 条记忆摘要、以及以规则为查询的联想区。接着按规则和最近记忆继续工作。

**主动回忆：** 想确认某件事以前聊过没有，或者找某个文档，用 `apm search "关键词"` 实时检索整个 `docs/`。需要更多结果就加 `--page` / `--num`。

**记录新记忆：** 一段对话有了值得留住的结论，就在 `docs/apm/memory/` 下直接写一个新文件，按格式填好 front matter 和正文。会话快结束时或者关键节点记一条，别事无巨细都记。

**更新持久规则：** 出现了跨会话仍该遵守的约定，就编辑 `docs/apm/RULE.md` 加进去；没有新规则就别动它。

---

## 勿混淆的路径

| 路径 | 用途 |
|------|------|
| `docs/apm/RULE.md` | 持久规则，`apm read` 规则区原样输出 |
| `docs/apm/memory/*.md` | 对话记忆，`apm read` 取最近 5 条摘要，`apm search` 实时检索 |
| `docs/`（项目根下其余 `.md`） | 项目文档，参与 `apm read` 联想区与 `apm search` 检索 |
| 仓库内其他 `memory/` 目录 | 与 APM 无关，不要往里写 APM 记忆 |
| `.apm/`（旧版运行态目录） | 新架构已移除，不再使用；若遇到旧仓库残留，按需清理 |
