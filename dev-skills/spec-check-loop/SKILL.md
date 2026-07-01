---
name: spec-check-loop
description: 子代理循环审查 PRD/SPEC 并对照代码库提出修改意见，主代理修复文档后重复审查直至 execute-ready（可进入实现）。适用于 spec-generate 落盘后的文档收敛、编码前的 spec 质量闸门。用户提及 spec check loop、文档审查循环、execute ready、PRD/SPEC 复审时使用。
disable-model-invocation: true
---

# Spec Check Loop

## 目的

在 **编码或 code-dev-loop 实现之前**，通过「审查 → 修复文档 → 再审查」循环，使 PRD/SPEC 达到 **execute-ready**：

1. 子代理对照 **代码库** 审查 PRD/SPEC（非仅读文档）
2. 主代理根据必改项 **修复 PRD/SPEC**（可顺带小改 PRD 与 SPEC 不一致处）
3. 重复直至 **execute-ready**，或用户叫停 / 达到轮次上限

本 skill **只收敛文档**；不写实现代码、不跑实现向 DAG（那是 `code-dev-loop`）。

主代理与子代理的协作、说明、评审结论**一律使用中文**（路径、标识符、协议字段等除外）。

---

## 关键术语

### execute-ready

文档质量闸门：**可以开始按 SPEC 编码**。须同时满足：

- **无未闭合 P0**（矛盾、缺失契约、与代码硬冲突、验收不可测）
- P1 已修复 **或** 已写入 SPEC「已知限制 / 实现注」且用户未反对
- 审查子代理结论为 **Go**（或主代理汇总后认定等效 Go）

不等于 merge-ready；merge-ready 属于 `code-dev-loop` 完成后的代码评审（含 `code-review` 全局 CR 与收尾项）。

### 审查轮（review round）

一次完整的：**子代理 readonly 审查 → 主代理汇总 →（若未 ready）文档修复 → 下一轮**。

轮次从 1 开始；`dynamic` 记录当前轮次与上轮 must-fix 闭合情况。

### 同步等待

每轮审查须 **派发子代理并等待返回** 后再决定修复或结束；禁止未审即改、未改即宣称 ready。

---

## 与相关 skill 的分工

| Skill | 阶段 | 产出 |
|-------|------|------|
| `prd-generate` | 需求澄清 | `prd.md` |
| `spec-generate` | 首次方案 | `spec.md` |
| **`spec-check-loop`** | **编码前收敛** | PRD/SPEC 修订至 execute-ready |
| `code-dev-loop` | 按 spec 实现 | 代码 merge-ready |

推荐顺序：`prd-generate` → `spec-generate` → **`spec-check-loop`** → 用户确认 → `code-dev-loop`。

---

## 开始前检查

- 已知：`PRD path`、`SPEC path`（至少 SPEC；仅有 PRD 时先 `spec-generate` 或标明 SPEC 待写）
- 已知：仓库根路径、迭代名称（如 `agent-run-lifecycle-unify`）
- 若项目使用 APM：`apm read` + `apm kb search --q "<关键词>"`（workspace 不完整时仍可手工读 `.apm/kb/docs/...`）
- **不要求**工作区干净（本阶段只改文档）；若同时改代码则偏离本 skill

**准备完成**：更新 `dynamic`（PRD/SPEC 路径、轮次 0、状态「待首轮审查」）与 `persist`（迭代名、文档路径）。

---

## 总流程

```text
准备 → [审查子代理] → 汇总（P0/P1/P2 + Go/No-Go）
                          ↓
              execute-ready? ─是→ 通知用户确认 → 结束（可进入实现 / code-dev-loop）
                          ↓否
              主代理修复 PRD/SPEC → 轮次 +1 → 回到 [审查子代理]
```

**轮次上限**：默认 **5** 轮；仍 No-Go 时向用户汇报未闭合 P0 并请求拍板，勿自行宣布 ready。

---

## Step 1：派遣审查子代理（每轮必做）

每轮派遣 **一个** readonly 审查子代理（推荐 `generalPurpose` + `readonly: true`）。

### 审查子代理必须做的事

1. **通读** 当前 PRD、SPEC（含 YAML Front Matter、`dependency` 前置 PRD）
2. **对照代码库** 阅读关键路径（入口、将改模块、测试、事件契约）；禁止仅凭文档下结论
3. 若有上轮 must-fix：逐项标注 **已修复 / 仍开放 / 引入新问题**
4. 输出结构化审查报告（见下方模板）

### 审查子代理禁止

- 修改代码或 PRD/SPEC 文件
- 宣布 execute-ready（只给 Go/No-Go 建议，由主代理收敛）

### 审查 prompt 模板

```text
【语言要求】
- 全程使用中文；路径、符号、命令可保留原文

请以 readonly 模式审查下列迭代文档是否达到 execute-ready（可开始编码）。

仓库：<REPO_PATH>
PRD：<PRD_PATH>
SPEC：<SPEC_PATH>
审查轮次：第 <N> 轮
上轮 must-fix（若有）：
- <...>

必须对照代码库阅读相关实现（列出你将阅读的文件）。

请按以下结构返回：
1）相对上轮：已修复项 / 仍开放项 / 新引入问题
2）剩余问题（P0/P1/P2），每条含：严重度、问题、证据（文档章节 + 代码路径）、修改建议
3）首轮已修复项摘要表（第 2 轮起）
4）结论：Go（execute-ready）或 No-Go，并一句话理由

P0 定义：矛盾、缺失 API/验收、与现有代码冲突、实施必打架的歧义。
```

---

## Step 2：主代理汇总与决策

收到子代理报告后：

1. **去重合并** 与子代理结论；主代理可补一条自己发现的 P0（须注明证据）
2. **判定本轮状态**：
   - **execute-ready**：无 P0；P1 已处理或已文档化；子代理 Go 或主代理等效认定
   - **not ready**：存在未闭合 P0，或子代理 No-Go
3. **向用户简短汇报**（一轮一次）：轮次、P0 数量、是否 ready；**not ready** 时列出 P0 标题

---

## Step 3：修复文档（仅 not ready 时）

主代理 **直接编辑** PRD/SPEC（文档修复不强制子代理；范围大时可派 `generalPurpose` 非 readonly 专司改文档，但仍须下一轮审查）。

修复原则：

- **只改文档** 闭合 must-fix；不顺手改实现代码
- P0 必须在本轮修复中闭合
- P1：优先修；来不及则写入 SPEC「风险与实现注」并标为已知限制
- 修复后 **同步 PRD 与 SPEC**（验收、命名、契约一致）
- 大改契约时检查 `dependency` 前置 PRD 是否需同步一句

**修复完成**：更新 `dynamic`（轮次、上轮 must-fix 闭合列表、状态「待下轮审查」）与 `persist`（已拍板契约要点、仍开放 P1）。

---

## Step 4：循环终止与用户确认

### 自动终止（execute-ready）

主代理用中文通知用户：

- 审查共 N 轮
- 已闭合的 P0 摘要
- PRD/SPEC 路径
- **请确认是否开始实现**（或进入 `code-dev-loop`）

用户确认前：**不开始编码**（与 `spec-generate` 一致）。

### 用户中途指令

- 「先实现 P0 文档项」→ 仅修文档，继续循环
- 「接受某 P1 风险开工」→ 写入 SPEC 后可为 ready，须用户显式说
- 「停止循环」→ 汇报当前 No-Go 项后结束

**用户确认 execute-ready 后**：更新 `dynamic`（状态「spec 已确认，可进入实现」）与 `persist`（已确认要点）。

---

## 严重度与闭合标准

| 级别 | 含义 | execute-ready 要求 |
|------|------|-------------------|
| **P0** | 不做必返工：矛盾、双端职责冲突、缺 hook/API、验收不可测、与代码硬冲突 | **必须 0 条** |
| **P1** | 应修：测试缺口、接线矩阵不全、边界未写 | 修完或写入 SPEC 实现注 |
| **P2** | 可选：父 PRD 同步、toast 等非强制验收 | 可遗留 |

---

## 阶段记忆更新（APM）

| 阶段 | dynamic | persist |
|------|---------|---------|
| 准备 | PRD/SPEC 路径、轮次 0 | 迭代名、文档路径 |
| 每轮审查后 | 轮次、P0/P1 计数、Go/No-Go | must-fix 摘要 |
| 文档修复后 | 状态「待下轮审查」、已闭合 P0 | 已拍板契约 |
| execute-ready | 状态「待用户确认」 | 核心方案要点 |
| 用户确认 | 状态「可进入实现」 | 已确认要点 |

---

## 失败处理

**子代理失败**（超时、资源）：等待后重试一次；仍失败则主代理执行等效 readonly 审查（读文档+代码），标注「手工审查」，下轮尽量恢复子代理。

**多轮震荡**（同一 P0 反复出现）：停止自动循环，请用户拍板二选一写进 SPEC。

---

## 执行检查清单

- [ ] 已读 PRD + SPEC + dependency 前置
- [ ] 每轮已派 readonly 审查子代理并 **同步等待**
- [ ] 审查含代码库对照，非空泛文档互审
- [ ] P0 闭合后才可宣称 execute-ready
- [ ] 未在用户确认前开始编码
- [ ] 已更新 dynamic / persist（若 APM 可用）

---

## 与 code-dev-loop 的衔接

execute-ready 且用户确认后，将 **同一份 SPEC 路径** 与 **Context Bundle 素材**（spec 摘要、已拍板决策）交给 `code-dev-loop`（默认 **strict** mode）：

```text
Spec：<SPEC_PATH>
PRD：<PRD_PATH>
分支：fix/<topic> 或 feature/<topic>
```

spec-check-loop **不替代** 实现阶段的评审；实现后须 `code-dev-loop` 分阶段 CR + `code-review` 全局 CR，达到完完全全 merge-ready。
