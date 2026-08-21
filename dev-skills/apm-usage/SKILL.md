---
name: apm-usage
description: APM（Agent Persistence Memory）使用指南：用 apm init 初始化、apm read 读取规则与最近记忆。记忆由 agent 直接写文件到 docs/apm/memory/，不通过 APM 命令；持久规则维护在 docs/apm/RULE.md。在提到 apm、外置记忆、会话恢复，或需要按统一约定记录对话记忆时使用。本文件为记忆「写什么、怎么读」的唯一权威，其它 skill 不得另立字段表或目录约定。
disable-model-invocation: true
---

# APM 使用指南

对应 CLI：`agent-persistence-memory`（`apm`）。

APM 现在只做两件事：初始化目录、读取规则与最近记忆。记忆文件由 agent 直接写到磁盘上，不走 APM 命令。整体围绕一个 `docs/apm/` 目录展开，不再有 `.apm/` 运行态目录。

## 快速开始

```bash
apm init                # 在项目根初始化 docs/apm/（幂等，已存在不报错）
apm read                # 会话开始必做：规则区 + 最近记忆
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
- `docs/apm/memory/*.md`：每条一个文件，`apm read` 取最近 5 条做摘要。
- 不再有 `.apm/` 目录，也不再有 role/persist/dynamic 之分；持久规则统一在 `RULE.md`，对话记忆统一在 `memory/`。
- APM 不扫描 `docs/apm/` 以外的内容；需要项目里其它 `.md` 文档时，agent 直接按路径读取。

---

## 命令详解

只有两个命令：`apm init`、`apm read`。没有 role/persist/dynamic/write/show/index/config/search。

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

无参数。会话开始时执行一次，拿到两段上下文：

1. **规则区**：`docs/apm/RULE.md` 的全文。
2. **最近记忆区**：`docs/apm/memory/` 下最近 5 条记忆的摘要，按 front matter 的 `date` 降序排列。每条给标题、日期、摘要和文件路径。

两段之间会用标题分隔。规则区给的是「该遵守什么」，最近记忆区给的是「最近聊过什么」。想看某条记忆的完整内容，按输出里的路径直接读那个文件就好。

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

### 正文：user / assistant 多轮对话

front matter 之后是对话正文，由**多轮** `user:` 和 `assistant:` 交替组成，按对话发生顺序排列：

```markdown
user:
第一轮用户原话或意图摘要。

assistant:
第一轮回复要点。

user:
第二轮用户追问或补充。

assistant:
第二轮回复与达成的结论。
```

轮次多少不限，按实际对话展开就好。正文不必逐字记全，每一轮挑关键的写：用户问了什么、我回了什么、最后结论是什么。啰嗦的过程流水可以省略。

### 完整示例

```markdown
---
date: 2026-08-14 15:30
title: foo 模块用 X 方案替代 Y
keywords: foo, 架构决策, Y 已废弃
abstract: 讨论后拍板 foo 模块改用 X 方案，Y 方案不再维护；迁移分批做，不阻塞新需求。
---

user:
foo 模块最近总觉得别扭，是不是该重构一下？

assistant:
看了一下，主要痛点是 foo 还在依赖 Y 方案，而 Y 上个月就停止维护了。我建议直接切到 X 方案，改动面不大，大概三个文件。

user:
那老代码怎么办？新功能我都按 X 写？

assistant:
对，新功能一律按 X 方案写；老代码迁移可以分批做，不阻塞新需求。

user:
行，那就这么定。RULE.md 里补一条吧，免得后面忘了。

assistant:
好，我在 RULE.md 里补了一条约束：不再引入对 Y 的新依赖，新代码一律走 X。
```

### 避免重复与冗余

写新记忆前先做一次查重，别让 memory 目录里堆满重复条目：

1. **写前先查**：用 `apm read` 看最近记忆；最近 5 条里没有、但怀疑更早记过，就直接浏览 `docs/apm/memory/` 的文件名（带日期和主题标识）或读可疑文件确认。
2. **同主题更新而非新建**：如果已有同主题的记忆文件，优先**更新那个文件**——追加新的对话轮次、刷新 `date` 和 `abstract`——而不是新建一个重复的文件。
3. **一个主题一个文件**：同一天对同一件事的多次讨论，合并进同一个记忆文件，靠多轮 `user:` / `assistant:` 展开，不要每次讨论都单开一个文件。
4. **只记增量**：写记忆时如果发现新结论和旧记忆部分重叠，只补充新增加的部分，不复述已经记过的内容。`abstract` 重写为覆盖全貌的一句话，但正文只追加增量轮次。

---

## 记忆写入方式

**记忆文件由 agent 直接写到磁盘，不通过 APM 命令。** APM 不提供「write」命令，也没有 `--stdin` / `--text` 之类的入参。

写一条新记忆的步骤：

1. **查重**：`apm read` 看最近记忆，必要时浏览 `docs/apm/memory/` 目录，确认没有同主题的已有记忆；有则走「更新已有文件」（见「避免重复与冗余」）。
2. 在 `docs/apm/memory/` 下新建文件，命名 `yyyyMMdd-简短标识.md`。
3. 按上面「记忆文件格式」写好 front matter（`date` / `title` / `keywords` / `abstract`）和多轮 `user:` / `assistant:` 正文。
4. 用 agent 的写文件工具落盘。完成。

下一次 `apm read` 就会自动把它纳入最近记忆区，不需要任何额外的「index build」或「refresh」步骤。

`RULE.md` 也是同理：要加规则就直接编辑这个文件，存盘即生效。

---

## 典型场景

**会话初始化：** 进项目第一件事 `apm read`，拿到规则区（RULE.md 全文）和最近 5 条记忆摘要。接着按规则和最近记忆继续工作。

**主动回忆：** 想确认某件事以前聊过没有，先看 `apm read` 的最近记忆；时间更久的，按 `docs/apm/memory/` 下的文件名定位，再直接读文件。记忆文件的 front matter 里有 `keywords` 和 `abstract`，扫一眼就能判断是不是要找的那条。

**记录新记忆：** 一段对话有了值得留住的结论，先查一下同主题是否已有记忆——有就更新那个文件（追加轮次、刷新 date/abstract），没有再新建。会话快结束时或者关键节点记一条，别事无巨细都记，也别把同一件事反复记成多个文件。

**更新持久规则：** 出现了跨会话仍该遵守的约定，就编辑 `docs/apm/RULE.md` 加进去；没有新规则就别动它。

---

## 勿混淆的路径

| 路径 | 用途 |
|------|------|
| `docs/apm/RULE.md` | 持久规则，`apm read` 规则区原样输出 |
| `docs/apm/memory/*.md` | 对话记忆，`apm read` 取最近 5 条摘要 |
| `docs/`（项目根下其余 `.md`） | 项目文档，与 APM 无关，agent 需要时直接读 |
| 仓库内其他 `memory/` 目录 | 与 APM 无关，不要往里写 APM 记忆 |
| `.apm/`（旧版运行态目录） | 新架构已移除，不再使用；若遇到旧仓库残留，按需清理 |
