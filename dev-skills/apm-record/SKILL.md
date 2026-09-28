---
name: apm-record
description: APM（Agent Persistence Memory）记忆写入侧：把对话结论落盘为 docs/apm/memory/ 下的多轮对话记忆（yyyyMMdd-name.md：front matter + user/assistant 正文），维护 docs/apm/RULE.md 持久规则；写前查重、同主题追加轮次而非新建文件。没有 CLI、没有服务，agent 用写文件工具直接落盘。在记录对话记忆、更新 RULE.md，或其它 skill 提到「记忆语义」「按 apm-record 落盘」时使用。目录结构、记忆文件格式与写路径以本文件为唯一权威，其它 skill 不得另立字段表或目录约定；读侧（会话初始化、主动回忆）见 apm-recall。
disable-model-invocation: true
---

# APM 记忆记录（Record）

APM 的记忆就是 `docs/apm/` 下的 markdown 文件，跟着项目一起进版本控制。本文件负责**怎么写**：写什么进哪一层、什么格式、怎么查重。读侧（会话初始化、主动回忆）见 `apm-recall`。

## 记忆语义

记忆分两层，都落在 `docs/apm/` 里。

### RULE.md：持久规则（类 AGENTS.md）

`docs/apm/RULE.md` 装的是**换会话仍该遵守**的内容，写法和 `AGENTS.md` 一样，偏约定、边界、已拍板的决策。适合写：

- 项目术语、模块边界、命名约定
- 用户拍板且跨任务仍有效的决策（「X 模块不负责 Y」）
- 协作/实现约束（「blocking 步骤必须有对应测试 id」）

**不要**把「本会话做到哪一步」「某次验证过了没」写进 RULE.md——那是具体对话记忆的范畴。路径类信息如果只是服务当前任务，写成一条记忆就好；只有当某条路径已经是团队约定时，才值得进 RULE.md。

RULE.md 由 agent 直接维护：有新规则就整理进去，没有就别动它。

### memory 文件：带时间的对话记忆

`docs/apm/memory/` 下的每个 `.md` 文件是一条**对话记忆**，文件名形如 `yyyyMMdd-简短标识.md`。它记录的是某次具体的交流——用户问了什么、我回了什么、当时的关键结论是什么。文件靠 front matter 的 `date` 排序，会话初始化（见 `apm-recall`「快速开始」）取最近 5 条做摘要。

记忆文件适合写：

- 某次讨论的关键问答与结论
- 用户透露的偏好、踩过的坑、当时的决策理由
- 当前任务的阶段性进展（作为事后回顾，而不是工作流状态）

不要把 DAG 节点表、wave 计划、验证日志这类**工作流状态**塞进记忆文件，那些东西属于 Context Bundle 或其它编排文件。

---

## 目录结构与初始化

```
docs/
  apm/
    RULE.md          # 持久规则，agent 直接维护
    memory/          # 对话记忆目录
      yyyyMMdd-name.md
      .gitkeep       # 保证空目录能进版本控制
```

新项目第一次接入时，agent 直接创建骨架（幂等，已存在不覆盖）：

```bash
mkdir -p docs/apm/memory
touch docs/apm/RULE.md docs/apm/memory/.gitkeep
```

- `docs/apm/RULE.md`：持久规则，会话初始化时原样读它。
- `docs/apm/memory/*.md`：每条记忆一个文件，初始化片段取最近 5 条摘要。
- 没有 `.apm/` 目录，也没有 role/persist/dynamic 之分；持久规则统一在 `RULE.md`，对话记忆统一在 `memory/`。
- 约定只覆盖 `docs/apm/`；项目里其它 `.md` 文档与 APM 无关，agent 需要时直接按路径读取。

> 历史：APM 曾是一个 npm CLI（`apm init` / `apm read` / `apm search`），2026-08 退役。旧仓库里的 `.apm/` 目录已废弃，遇到残留按需清理。

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
| `date` | 这条记忆的时间，格式 `yyyy-MM-dd HH:mm`（24 小时制）。`apm-recall` 的初始化片段按它降序排最近记忆 |
| `title` | 一句话标题，出现在最近记忆摘要里 |
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

1. **写前先查**：按 `apm-recall`「快速开始」看最近记忆；最近 5 条里没有、但怀疑更早记过，就直接浏览 `docs/apm/memory/` 的文件名（带日期和主题标识）或读可疑文件确认。
2. **同主题更新而非新建**：如果已有同主题的记忆文件，优先**更新那个文件**——追加新的对话轮次、刷新 `date` 和 `abstract`——而不是新建一个重复的文件。
3. **一个主题一个文件**：同一天对同一件事的多次讨论，合并进同一个记忆文件，靠多轮 `user:` / `assistant:` 展开，不要每次讨论都单开一个文件。
4. **只记增量**：写记忆时如果发现新结论和旧记忆部分重叠，只补充新增加的部分，不复述已经记过的内容。`abstract` 重写为覆盖全貌的一句话，但正文只追加增量轮次。

---

## 记忆写入方式

**记忆文件由 agent 用写文件工具直接落盘**，没有任何命令、也不需要任何构建步骤。

写一条新记忆的步骤：

1. **查重**：看最近记忆摘要，必要时浏览 `docs/apm/memory/` 目录，确认没有同主题的已有记忆；有则走「更新已有文件」（见「避免重复与冗余」）。
2. 在 `docs/apm/memory/` 下新建文件，命名 `yyyyMMdd-简短标识.md`。
3. 按上面「记忆文件格式」写好 front matter（`date` / `title` / `keywords` / `abstract`）和多轮 `user:` / `assistant:` 正文。
4. 落盘即完成，下一次会话初始化自动纳入最近记忆。

`RULE.md` 也是同理：要加规则就直接编辑这个文件，存盘即生效。

---

## 典型场景（写侧）

**记录新记忆：** 一段对话有了值得留住的结论，先查一下同主题是否已有记忆——有就更新那个文件（追加轮次、刷新 date/abstract），没有再新建。会话快结束时或者关键节点记一条，别事无巨细都记，也别把同一件事反复记成多个文件。

**更新持久规则：** 出现了跨会话仍该遵守的约定，就编辑 `docs/apm/RULE.md` 加进去；没有新规则就别动它。

（会话初始化与主动回忆是读侧场景，见 `apm-recall`。）

---

## 勿混淆的路径

| 路径 | 用途 |
|------|------|
| `docs/apm/RULE.md` | 持久规则，agent 直接维护 |
| `docs/apm/memory/*.md` | 对话记忆，每条一个文件 |
| `docs/`（项目根下其余 `.md`） | 项目文档，与本约定无关，勿往里写 APM 内容 |
| 仓库内其他 `memory/` 目录 | 与本约定无关，不要往里写 APM 记忆 |
| `.apm/`（旧版运行态目录） | 已废弃，勿写入 |
