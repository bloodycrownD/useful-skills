---
name: spec-check-loop
description: 子代理循环审查 PRD/SPEC 并对照代码库提出修改意见；not-ready 时由子代理修复文档。主代理=编排（拆 wave、派 Task、同步等待、汇总、改 doc_fix_plan/dag_version、汇报）。适用于文档收敛、编码前的质量闸门。用户提及 spec check loop、文档审查循环、execute ready、PRD/SPEC 复审时使用。
disable-model-invocation: true
---

# Spec Check Loop

**文档审查编排** skill：通过「审查 → 子代理 doc-fix → 再审查」循环，使 PRD/SPEC 达到 **execute-ready**（文档可支撑按 spec 编码）。

主代理 = 编排；子代理 = **review**（readonly 审查）/ **doc-fix**（修复 PRD/SPEC）。中文协作。

本 skill **只收敛文档**；不写实现代码、不跑实现向 DAG。

---

## 边界

- **输入**：用户指定的 PRD / SPEC 路径（至少 SPEC；仅有 PRD 时须先补齐 spec 或标明 SPEC 待写）
- **产出**：修订后的 PRD/SPEC + execute-ready 结论
- **不做**：写代码、改实现、替代 spec 的首次撰写

---

## 关键术语

### execute-ready

文档质量闸门：**可以开始按 SPEC 编码**。须同时满足：

- **无未闭合 P0**（矛盾、缺失契约、与代码硬冲突、验收不可测）
- P1 已修复 **或** 已写入 SPEC「已知限制 / 实现注」且用户未反对
- 审查子代理结论为 **Go**（或主代理汇总后认定等效 Go）

不等于代码已 review-ready；本 skill 只管文档质量。

### 审查轮（review round）

一次完整的：**子代理 readonly 审查 → 主代理汇总 →（若未 ready）派遣 doc-fix 子代理并同步等待 → 下一轮**。

轮次从 1 开始；`dynamic` 记录当前轮次与上轮 must-fix 闭合情况。

### 同步等待

每轮审查须 **派发审查子代理并等待返回** 后再汇总；not-ready 时须 **派发 doc-fix 子代理并等待返回** 后再进入下轮审查。禁止未审即改、未等 fix 即宣称 ready。

### 子代理派遣规范

| 节点 | 工具 | subagent_type | readonly | 并行 |
|------|------|---------------|----------|------|
| **review** | Task | generalPurpose | **true** | 每轮 **一个**审查子代理；大文档可拆 scope 后单轮汇总 |
| **doc-fix** | Task | generalPurpose | **false** | 无同文件冲突可同 wave |

- **同步等待** 当前 wave 全部 doc-fix 返回后再汇总、进入下轮审查
- **同文件禁止** 并行 doc-fix（PRD 与 SPEC 视为不同文件）
- **失败**：重试一次 → 仍失败则主代理可手工等效 doc-fix，标注「手工 doc-fix」（见「失败处理」）

---

## 审查轮重编排

文档审查循环在 **not-ready 时重编排**，而非固定「审→改→审」单线（与 code-dev-loop 的 fix 重编排对称）：

| 动作 | 何时 |
|------|------|
| **并行化** | 无文件冲突的 doc-fix 同 wave |
| **合并** | 多 doc-fix 同一章节/模块 → 单 doc-fix 节点 |
| **拆分** | 节点过大或反复 fail → 串行子节点 |
| **合并审查** | 多文档同类问题 → 单审查子代理一轮覆盖 |
| **拆分审查** | 文档过大 → 分 scope 审查再汇总 |
| **优先 P0** | 阻塞下游的 must-fix 进下一 wave 首部 |

```yaml
dag_version: 1
review_round: 2
prd_path: .apm/kb/docs/Iterations/<name>/prd.md
spec_path: .apm/kb/docs/Iterations/<name>/spec.md
open_must_fix: []
doc_fix_plan: [[spec-§3], [prd-验收, spec-测试]]  # wave 计划；完成后置 []
status: 待下轮审查  # 待首轮审查 | 待下轮审查 | 待用户确认 | execute-ready 已确认
```

**禁止**：No-Go 后只改一处不更新 `doc_fix_plan` / `dag_version` / 不进入下轮审查。

无 APM 时：用 `docs/.iteration-state.yaml` 或对话内 YAML 块等价维护。

---

## 开始前检查

- 已知：`PRD path`、`SPEC path`（至少 SPEC；仅有 PRD 时须先补齐 spec 或标明 SPEC 待写）
- 已知：仓库根路径、迭代名称（如 `agent-run-lifecycle-unify`）
- 若项目使用 APM：`apm read` + `apm kb search --q "<关键词>"`（workspace 不完整时仍可手工读 `.apm/kb/docs/...`）
- **不要求**工作区干净（本阶段只改文档）；若同时改代码则偏离本 skill

**准备完成**：更新 `dynamic`（PRD/SPEC 路径、轮次 0、状态「待首轮审查」）与 `persist`（迭代名、文档路径）。

---

## 总流程

```text
准备 → [审查子代理] → 汇总（P0/P1/P2 + Go/No-Go）
                          ↓
              execute-ready? ─是→ 通知用户确认 → 结束
                          ↓否
              拆 wave → [doc-fix 子代理] → 同步等待 → 轮次 +1 → 回到 [审查子代理]
```

**轮次上限**：默认 **5** 轮；仍 No-Go 时向用户汇报未闭合 P0 并请求拍板，勿自行宣布 ready。

**主代理职责**（仅此）：拆 wave、派 Task、同步等待、汇总、改 `doc_fix_plan` / `dag_version`、向用户汇报。

**主代理禁止**：直接编辑 PRD/SPEC 闭合 must-fix；未等 doc-fix 子代理返回即进入下轮审查。

---

## Step 1：派遣审查子代理（每轮必做）

每轮派遣 **一个** readonly 审查子代理（`generalPurpose` + `readonly: true`）。

### 审查子代理必须做的事

1. **通读** 当前 PRD、SPEC（含 YAML Front Matter、`dependency` 前置 PRD）
2. **对照代码库** 阅读关键路径（入口、将改模块、测试、事件契约）；禁止仅凭文档下结论
3. 若有上轮 must-fix：逐项标注 **已修复 / 仍开放 / 引入新问题**
4. 输出结构化审查报告（见「审查报告 Schema」）

### 审查子代理禁止

- 修改代码或 PRD/SPEC 文件
- 宣布 execute-ready（只给 Go/No-Go 建议，由主代理收敛）

### 审查 prompt 模板（派遣用）

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
1）相对上轮：已修复项 / 仍开放项 / 新引入问题（第 2 轮起须附「已修复项摘要表」）
2）剩余问题（P0/P1/P2），每条含：严重度、问题、证据（文档章节 + 代码路径）、修改建议
3）结论：Go（execute-ready）或 No-Go，并一句话理由

P0 定义：矛盾、缺失 API/验收、与现有代码冲突、实施必打架的歧义。
```

### 审查报告 Schema（主代理解析用）

子代理须按下列结构返回（与派遣 prompt 一致）：

```text
1）相对上轮：已修复项 / 仍开放项 / 新引入问题（第 2 轮起须附「已修复项摘要表」）
2）剩余问题（P0/P1/P2），每条含：严重度、问题、证据、修改建议
3）结论：Go | No-Go + 一句话理由
```

---

## Step 2：主代理汇总与决策

收到审查子代理报告后：

1. **去重合并** 与子代理结论；主代理可补一条自己发现的 P0（须注明证据）
2. **判定本轮状态**：
   - **execute-ready**：无 P0；P1 已处理或已文档化；子代理 Go 或主代理等效认定
   - **not ready**：存在未闭合 P0，或子代理 No-Go
3. **向用户简短汇报**（一轮一次）：轮次、P0 数量、是否 ready；**not ready** 时列出 P0 标题

**主代理禁止**：亲自编辑 PRD/SPEC 闭合 must-fix（须进入 Step 3 派 doc-fix 子代理）。

not ready 时：根据 must-fix 拆 wave、写入 `doc_fix_plan`，`dag_version++`，进入 Step 3。

---

## Step 3：派遣文档 fix 子代理（仅 not ready 时）

not ready 时，主代理 **必须** 派遣 doc-fix 子代理修复 PRD/SPEC，**不得**主代理直改。

### 派遣规则

1. 按 `doc_fix_plan` 取当前 wave；无冲突的 doc-fix **并行**派发
2. **同步等待** 当前 wave 全部返回后再汇总
3. 同一 must-fix 重派 **≤3 次**；仍失败 → **blocked**，请用户拍板
4. 全部 doc-fix 完成后 `doc_fix_plan` 置空，状态「待下轮审查」，轮次 +1

### 主代理禁止

- 直接编辑 PRD/SPEC 闭合 must-fix
- 未等 doc-fix 子代理返回即进入下轮审查

### doc-fix 子代理 prompt 模板（派遣用）

```text
【语言要求】
- 全程使用中文；路径、符号、命令可保留原文

请以非 readonly 模式修复下列 PRD/SPEC 文档（只改文档，不改实现代码）。

仓库：<REPO_PATH>
PRD：<PRD_PATH>
SPEC：<SPEC_PATH>
审查轮次：第 <N> 轮
本轮 fix 范围：<WAVE_SCOPE>（如 prd-验收、spec-§3）

must-fix 清单（须在本 wave 内闭合）：
- <P0/P1 条目 + 证据 + 修改建议>

约束：
- 只改 PRD/SPEC，闭合分配给你的 must-fix
- 修复后同步 PRD 与 SPEC（验收、命名、契约一致）
- 大改契约时检查 dependency 前置 PRD 是否需同步一句

请用中文返回：
1）已修改文件与章节
2）各 must-fix 闭合情况（已修复 / 仍开放 / 需主代理拍板）
3）与另一文档的同步说明（如有）
4）阻塞项（如有）
```

### doc-fix 原则（子代理须遵守）

- **只改文档** 闭合 must-fix；不顺手改实现代码
- P0 必须在本轮修复中闭合
- P1：优先修；来不及则写入 SPEC「风险与实现注」并标为已知限制
- 修复后 **同步 PRD 与 SPEC**（验收、命名、契约一致）
- 大改契约时检查 `dependency` 前置 PRD 是否需同步一句
- 若修改了 kb 内 PRD/SPEC 且 APM 可用：执行 `apm kb index rebuild`

**修复完成**：主代理更新 `dynamic`（轮次、上轮 must-fix 闭合列表、状态「待下轮审查」）与 `persist`（已拍板契约要点、仍开放 P1）。

---

## Step 4：循环终止与用户确认

### 自动终止（execute-ready）

主代理用中文通知用户：

- 审查共 N 轮
- 已闭合的 P0 摘要
- PRD/SPEC 路径
- **请确认文档是否满足 execute-ready**（是否按当前 spec 开工）

用户确认前：**不开始编码**（若用户同时要求实现，须先完成 execute-ready 或用户显式接受风险）。

### 用户中途指令

- 「先实现 P0 文档项」→ 仅修文档，继续循环
- 「接受某 P1 风险开工」→ 写入 SPEC 后可为 ready，须用户显式说
- 「停止循环」→ 汇报当前 No-Go 项后结束

**用户确认 execute-ready 后**：更新 `dynamic`（状态「execute-ready 已确认」、`execute_ready_confirmed: yes`）与 `persist`（已确认要点）。

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
| 用户确认 | 状态「execute-ready 已确认」、`execute_ready_confirmed: yes` | 已确认要点 |

---

## 失败处理

**审查子代理失败**（超时、资源）：等待后重试一次；仍失败则主代理执行等效 readonly 审查（读文档+代码），标注「手工审查」，下轮尽量恢复子代理。

**doc-fix 子代理失败**（超时、资源）：

1. 短暂停顿后重试一次
2. 仍失败：主代理可手工完成 doc-fix，但须同样闭合 must-fix 并记录于 dynamic，标注「手工 doc-fix」
3. 资源恢复后优先回到子代理流程

**多轮震荡**（同一 P0 反复出现）：停止自动循环，请用户拍板二选一写进 SPEC。

---

## 执行检查清单

- [ ] 已读 PRD + SPEC + dependency 前置
- [ ] 每轮已派 readonly 审查子代理并 **同步等待**
- [ ] 审查含代码库对照，非空泛文档互审
- [ ] not-ready 时已派 doc-fix 子代理并 **同步等待**（非主代理直改）
- [ ] P0 闭合后才可宣称 execute-ready
- [ ] 未在用户确认前开始编码
- [ ] 已更新 dynamic / persist（若 APM 可用）

---

## 完成产出（execute-ready 后）

用户确认后，在 dynamic 中保留可供后续使用的素材（用户或后续任务自行取用，本 skill 不强制指定下游步骤）：

```yaml
spec_path: ...
prd_path: ...
execute_ready_confirmed: yes
spec_confirmed: yes          # 可选，与 code-dev-loop 等实现对齐时用
explore_summary: ...          # 若上下文中有
blocking_steps: [...]
```
