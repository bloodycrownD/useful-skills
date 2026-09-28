---
name: spec-check-loop
description: 子代理循环审查 PRD/SPEC 并对照代码库提出修改意见；not-ready 时由主代理直接修复文档。主代理=编排+doc-fix（派审查 Task、同步等待、汇总、直改 PRD/SPEC、更新 doc_fix_plan/dag_version、汇报）。适用于文档收敛、编码前的质量闸门。用户提及 spec check loop、文档审查循环、execute ready、PRD/SPEC 复审时使用。
disable-model-invocation: true
---

# Spec Check Loop

**文档审查编排** skill：通过「审查 → 主代理 doc-fix 直改 → 再审查」循环，使 PRD/SPEC 达到 **execute-ready**（文档可支撑按 spec 编码）。

主代理 = 编排 + **doc-fix**（直接修复 PRD/SPEC）；子代理 = **review**（readonly 审查）。中文协作。

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

一次完整的：**子代理 readonly 审查 → 主代理汇总 →（若未 ready）主代理直接 doc-fix 闭合 → 下一轮**。闭合后 `review_round +1`。

轮次从 1 开始；当前轮次与上轮 must-fix 闭合情况写在 **iteration-state**，记忆文件里只用一两句人话概括现状。

### 同步等待

每轮审查须 **派发审查子代理并等待返回** 后再汇总；not-ready 时主代理 **直接完成 doc-fix** 后再进入下轮审查。禁止未审即改、must-fix 未修完即宣称 ready。

### 子代理派遣规范

| 节点 | 工具 | subagent_type | readonly | 并行 |
|------|------|---------------|----------|------|
| **review**（judge） | Task | generalPurpose | **true** | 每轮 **一个**；大文档先并行 evidence 取证，见下 |
| **evidence**（可选，大文档） | Task | generalPurpose | **true** | 按章节/模块并行；**只取证不下结论** |

- **同步等待** 当前 wave 全部返回后再汇总、决策
- **失败**：重试一次 → 仍失败则主代理执行等效 readonly 审查，标注「手工审查」（见「失败处理」）

### 大文档两段式审查（evidence → judge）

文档过大时，单审查者读全量太慢；拆多个审查者又会丢全局视野（跨章节矛盾谁也看不见）、P0/P1 标定不一、must-fix 合并困难。改两段：

1. **evidence**（并行 readonly，1–4 个）：按章节/模块取证落盘证据包——文档写了什么、与代码库的对应事实、可疑点（含章节号与代码路径），**只取证不下结论**
2. **judge**（每轮仍 **一个** readonly 审查子代理）：读全部证据包，**回读原文与关键代码路径**后按「审查报告 Schema」输出 Go/No-Go

约束：judge **禁止仅凭证据包下结论**（对应「必须对照代码库阅读」）；轮间「已修复/仍开放」追踪由 judge 承担，证据包只提供事实底座。

### 证据包（evidence pack）

跨轮次重复读的**稳定背景**落盘复用，**生产者写、消费者读**：

- 生产者：第一个读该区域的子代理（evidence / 首轮审查）顺手落盘，路径写入 iteration-state；主代理只持路径，**不亲自写包**——否则读代码的成本只是转移到了最贵的主代理上下文上
- 路径：`docs/iterations/<name>/cache/<topic>.md`（过程产物，不入版本控制；用户另指定则从其指定）
- 失效：包头记轮次与所涉代码 sha；过期 → 仅作线索，事实须新鲜读
- 边界：只放稳定背景（模块地图、代码对应关系、既往 findings）；**被审的当前 PRD/SPEC 与代码必须新鲜读**

### doc-fix：主代理直接执行

doc-fix 改的是 PRD/SPEC **文档本身**，must-fix 又已由审查子代理带回证据与修改建议——主代理在汇总后直接改就好，**不派子代理**。与 code-review-loop 的 spec-fix 同理：单文件文档改动，既不需要上下文隔离、也不需要并行。

若个别 must-fix 措辞须重读大量代码查证，可先派 readonly 子代理查证，文档仍由主代理编辑。

---

## 审查轮重编排

文档审查循环在 **not-ready 时重编排**，而非固定「审→改→审」单线（与 code-dev-loop 的 fix 重编排对称）：

| 动作 | 何时 |
|------|------|
| **doc-fix 直改** | not-ready 时主代理按 `doc_fix_plan` 顺序直接修复，串行落盘，无并行编排 |
| **优先 P0** | 阻塞下游的 must-fix 排在 doc-fix 直改最前 |
| **合并审查** | 多文档同类问题 → 单审查子代理一轮覆盖 |
| **两段式拆分** | 文档过大 → 并行 evidence 取证 + 单 judge 裁决（见「大文档两段式审查」） |

```yaml
dag_version: 1
review_round: 2
prd_path: docs/iterations/<name>/prd.md
spec_path: docs/iterations/<name>/spec.md
open_must_fix: []
doc_fix_plan: [spec-§3, prd-验收, spec-测试]  # 主代理按序直改；完成后置 []
status: 待下轮审查  # 待首轮审查 | 待下轮审查 | 待用户确认 | execute-ready 已确认
```

上表落在 iteration-state / 对话 YAML，**不是**记忆文件正文。记忆写法见 `apm-usage`。

**禁止**：No-Go 后只改一处不更新 `doc_fix_plan` / `dag_version` / 不进入下轮审查。

用 `docs/.iteration-state.yaml` 或对话内 YAML 块等价维护**编排状态**。

---

## 开始前检查

- 已知：`PRD path`、`SPEC path`（至少 SPEC；仅有 PRD 时须先补齐 spec 或标明 SPEC 待写）
- 已知：仓库根路径、迭代名称（如 `agent-run-lifecycle-unify`）
- 已按 `apm-usage`「快速开始」读规则与最近记忆（workspace 不完整时仍可手工读项目根 `docs/...`）
- **不要求**工作区干净（本阶段只改文档）；若同时改代码则偏离本 skill

**准备完成**：按 `apm-usage`「记忆语义」记录进展（同主题记忆文件追加轮次并刷新 `date`/`abstract`，一句话概括「文档审查循环待首轮审查」）。编排状态写入 iteration-state，勿塞进记忆文件。

---

## 总流程

```text
准备 → [审查子代理] → 汇总（P0/P1/P2 + Go/No-Go）
                          ↓
              execute-ready? ─是→ 通知用户确认 → 结束
                          ↓否
              主代理 doc-fix 直改 PRD/SPEC → 轮次 +1 → 回到 [审查子代理]
```

**轮次上限**：默认 **5** 轮；仍 No-Go 时向用户汇报未闭合 P0 并请求拍板，勿自行宣布 ready。

**主代理职责**：派审查 Task、同步等待、汇总、doc-fix 直改 PRD/SPEC、更新 `doc_fix_plan` / `dag_version`、向用户汇报。

**主代理禁止**：自审（审查须 readonly 子代理）；must-fix 未修完即宣称 execute-ready；改实现代码。

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

not ready 时：根据 must-fix 更新 `doc_fix_plan`（P0 靠前），`dag_version++`，进入 Step 3 由主代理直接修复。

---

## Step 3：doc-fix（主代理直接修复；仅 not ready 时）

not ready 时，主代理按 `doc_fix_plan` **直接编辑 PRD/SPEC** 闭合本轮 must-fix——单文件文档改动，不派子代理（理由见「子代理派遣规范 · doc-fix：主代理直接执行」）。

### 执行规则

1. 按 `doc_fix_plan` 顺序修复，P0 靠前
2. 同一 must-fix 反复修 **≥3 轮**仍未闭合 → **blocked**，请用户拍板
3. 全部修复后 `doc_fix_plan` 置空，状态「待下轮审查」，轮次 +1

### doc-fix 原则（主代理须遵守）

- **只改文档** 闭合 must-fix；不顺手改实现代码
- P0 必须在本轮修复中闭合
- P1：优先修；来不及则写入 SPEC「风险与实现注」并标为已知限制
- **修复后同步 PRD 与 SPEC**（验收、命名、契约一致）
- 大改契约时检查 `dependency` 前置 PRD 是否需同步一句

**修复完成**：主代理更新 iteration-state（轮次、闭合列表）；按 `apm-usage`「记忆语义」更新同主题记忆文件（追加轮次并刷新 `date`/`abstract`）。已拍板且跨任务仍有效的契约可写入 `RULE.md`。

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

**用户确认 execute-ready 后**：按 `apm-usage`「记忆语义」更新记忆（现状=execute-ready 已确认）；已确认要点若跨任务仍有效可写入 `RULE.md`。

---

## 严重度与闭合标准

| 级别 | 含义 | execute-ready 要求 |
|------|------|-------------------|
| **P0** | 不做必返工：矛盾、双端职责冲突、缺 hook/API、验收不可测、与代码硬冲突 | **必须 0 条** |
| **P1** | 应修：测试缺口、接线矩阵不全、边界未写 | 修完或写入 SPEC 实现注 |
| **P2** | 可选：父 PRD 同步、toast 等非强制验收 | 可遗留 |

---

## 阶段记忆更新（APM）

遵守 **`apm-usage`「记忆语义」**。轮次、`doc_fix_plan`、must-fix 清单属编排状态，写 iteration-state；记忆文件只用一两句人话描述当前任务进展。勿自造字段表。

---

## 失败处理

**审查子代理失败**（超时、资源）：等待后重试一次；仍失败则主代理执行等效 readonly 审查（读文档+代码），标注「手工审查」，下轮尽量恢复子代理。

**多轮震荡**（同一 P0 反复出现）：停止自动循环，请用户拍板二选一写进 SPEC。

---

## 执行检查清单

- [ ] 已读 PRD + SPEC + dependency 前置
- [ ] 每轮已派 readonly 审查子代理并 **同步等待**
- [ ] 审查含代码库对照，非空泛文档互审
- [ ] not-ready 时主代理已直接 doc-fix 闭合 must-fix（只改文档，未动实现代码）
- [ ] P0 闭合后才可宣称 execute-ready
- [ ] 未在用户确认前开始编码
- [ ] 已按 `apm-usage` 记忆语义更新记忆

---

## 完成产出（execute-ready 后）

用户确认后，在 **Context Bundle / iteration-state** 保留可供后续使用的素材（用户或后续任务自行取用，本 skill 不强制指定下游步骤）；记忆文件用一两句指向这些路径即可，勿把整段 YAML 贴进记忆：

```yaml
spec_path: ...
prd_path: ...
execute_ready_confirmed: yes
spec_confirmed: yes          # 可选，与 code-dev-loop 等实现对齐时用
explore_summary: ...          # 若上下文中有
blocking_steps: [...]
```
