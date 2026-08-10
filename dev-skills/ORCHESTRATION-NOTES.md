# 编排机制维护者笔记

> 本文件**给人看**，不给 agent 看。它不参与任何 skill 的运行时上下文，只帮助维护者理解四个 loop skill 共享的编排原语、各自的差异面，以及改动编排约定时需要同步哪些文件。

## 谁用了动态 DAG 编排

| skill | 节点类型 | 终点 | 产物 |
|-------|---------|------|------|
| `code-dev-loop` | impl / verify / cr-func / fix | dev-ready | 代码提交 |
| `code-review-loop` | review-scope / review-full / spec-fix | fix-spec-ready | cr-fix-spec 文档 |
| `spec-check-loop` | review / doc-fix | execute-ready | 修订后的 PRD/SPEC |
| `agile-dev` | 探索 / 实现（单线，非多轮 DAG） | 实现完成 + 留痕 | 代码 + prd.md/spec.md |

`agile-dev` 是单线探索→实现流程，不是多轮 DAG；但它也用了子代理派遣约定和 trivial 豁免，所以一并纳入管理。

## 共享的编排原语

以下约定在所有 loop 中语义一致。改其中任何一个，须同步改全部相关文件。

| 原语 | 约定 |
|------|------|
| **dag_version** | 初始 1；每次重编排（改图）后 `dag_version++` |
| **wave_plan** | 嵌套数组，外层是 wave 顺序，内层是可并行节点 |
| **node_status** | 每节点 `{ status: pending\|done\|failed, executor: subagent\|main, ... }`；trivial inline 标 `executor: main` |
| **同步等待** | 当前 wave 全部返回后才汇总；禁止未等即下一步 |
| **同文件禁止并行** | 写同一文件的节点不能同 wave |
| **震荡上限** | 同一 must-fix / fix 反复失败 ≥3 次 → blocked，请用户拍板 |
| **轮次上限** | spec-check-loop 默认 5 轮；code-review-loop 默认 5 轮；code-dev-loop 无固定轮次上限（靠震荡上限兜底） |
| **失败处理** | 子代理失败 → 重试一次 → 仍失败则主代理手工等效完成，标注「手工 xxx」 |
| **trivial 豁免** | 见下节 |

## trivial 豁免（2026-08 新增）

### 核心判据

派子代理的价值是**上下文隔离 + 并行**。当两者都不需要时——inline 做也不会消耗主代理大量上下文、又无可并发的兄弟节点——就不必派子代理，主代理自己干。

**主判据**：inline 执行不会消耗主代理大量上下文。

可观测信号（须同时满足）：

- 改动位置已知，无需深度探索
- 预估阅读量小（少数文件、改动集中）
- 验证输出短

**兜底启发式**：拿不准时，以「预估 ≤3 次工具调用（含读文件、搜索、跑测试等）」倾向主代理直接执行。调用次数只是粗略指标，本质还是看上下文消耗。

### 术语约定（2026-08 统一）

- **主代理直接执行**：trivial 豁免时的路径。早期草稿用「主代理 inline」，现已统一弃用，因为「inline」在这些 skill 里已有「子代理单次连续执行」的含义，重载会混。
- **子代理 inline**：子代理单次连续执行、不拆子代理的执行模式（原有术语，保留）。
- **trivial**：一个节点既不需要上下文隔离、又不需要并行时的状态。

### 各 skill 适用范围

| skill | 适用节点 | 不适用节点 | 理由 |
|-------|---------|-----------|------|
| `code-dev-loop` | impl、fix、verify | cr-func | cr-func 审查独立性须保留 |
| `code-review-loop` | spec-fix | review / review-scope / review-full | 审查要读多文件，独立性须保留 |
| `spec-check-loop` | doc-fix | review | 同上 |
| `agile-dev` | 实现（Step 3） | 探索（Step 2） | 探索的价值就是上下文隔离，豁免会适得其反 |

### 为什么不抽成共享 skill

三个原因：

1. `disable-model-invocation: true` 决定了运行时各 loop 不会主动 invoke 别的 skill，引用一个不在上下文里的 skill 是空操作
2. 各 loop 的节点语义、终止条件、重编排触发点全不同，共享部分只占骨架，差异面才是 prompt 精确性的关键
3. inline 复制回各 skill 与维持现状无实质区别，还多一层间接

所以选择：各 skill 各自维护完整自洽的 trivial 豁免条款，本文件负责记录共性和同步义务。

## 改动同步对照表

| 改了什么 | 要同步改的文件 |
|---------|--------------|
| trivial 豁免判据或术语 | `code-dev-loop`、`code-review-loop`、`spec-check-loop`、`agile-dev`、本文件 |
| trivial 定义 / 术语统一（主代理直接执行 vs 子代理 inline） | 同上（2026-08 已做一轮统一） |
| 编排状态字段（dag_version / wave_plan / node_status 语义） | `code-dev-loop`、`code-review-loop`、`spec-check-loop` |
| 震荡上限 / 轮次上限 | 对应的 loop skill |
| 子代理派遣规范（工具、readonly、并行） | 对应的 loop skill |
| 失败处理流程 | 所有 loop skill（约定一致） |

## 术语区分

| 术语 | 含义 | 出现在 |
|------|------|--------|
| `executor: main` | trivial 豁免，主代理直接执行该节点 | node_status |
| `executor: subagent` | 正常派遣子代理执行（默认，未标则默认此值） | node_status |
| 「手工 xxx」 | 子代理失败后主代理兜底，标注在 `dynamic`「现状」 | 失败处理 |
| 「trivial 直接执行」 | 经豁免判据认定后主代理直接做，标注在 `dynamic` 或汇报 | 各 skill 的豁免 section |

两条主代理直改路径需区分：「手工 xxx」是失败兜底（子代理挂了的应急），「trivial 直接执行」是效率优化（本来就不值得派子代理）。
