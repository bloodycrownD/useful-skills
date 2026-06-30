---
name: code-dev-loop
description: 基于 spec 的分阶段 DAG 编排：每阶段实现后立即 code-review 阶段 CR 并内循环修复，全部阶段完成后收尾 + 全局 CR，收敛至完完全全 merge-ready（含收尾、lint、无遗留）。评审子代理须遵循 code-review skill。
disable-model-invocation: true
---

# Code Dev Loop

## 目的

在 **spec 为唯一事实来源** 的前提下，通过「分阶段拆解 → 阶段内实现与 CR 内循环 → 收尾 → 全局 CR 外循环」完成实现：

1. 将 spec 拆解为 **阶段（phase）** 与阶段内 DAG 节点
2. **每个阶段**：实现 → 验证 → **阶段 CR**（`code-review`，`stage`）→ 若有 must-fix 则修复并重跑，直至 **phase-ready**
3. 全部阶段完成后：**收尾节点** → 全量验证 → **全局 CR**（`code-review`，`final`）→ 若有 must-fix 则修复并重跑，直至 **完完全全 merge-ready**

主代理负责编排与调度，**不得**跳过子代理自行完成实现类任务（资源故障见「失败处理」）。

主代理与子代理的协作、说明、评审结论**一律使用中文**（代码标识符、协议字段、第三方 API 名称等除外）。

---

## 关键术语

### phase（阶段）

spec 中一个可独立交付、独立评审的工作单元（如「模块 A」「API 层」「前端接线」）。每阶段有自己的 **阶段内循环**。

### 阶段内循环（inner loop）

单个 phase 内的闭环，直至 **phase-ready**：

```text
impl（实现）→ verify（验证）→ cr-stage（阶段 CR）─not ready─→ fix-stage → verify → cr-stage …
                                              └─ phase-ready ─→ 进入下一阶段
```

- 默认每阶段 **最多 5 轮** inner loop；仍 not phase-ready 时向用户汇报未闭合 P0/P1
- **phase-ready**：本阶段范围内 P0 = 0、P1 = 0（见 `code-review`）

### 全局外循环（outer loop）

全部 phase 与收尾完成后：

```text
cleanup（收尾）→ verify-final → cr-final（全局 CR）─not ready─→ fix-final → verify-final → cr-final …
                                                    └─ merge-ready ─→ 结束
```

- 默认 **最多 3 轮** outer loop
- **完完全全 merge-ready**：全量 P0/P1 = 0，且 `code-review` **E 节收尾项全部满足**（见下文）

### inline

子代理在**单次连续运行**内完成所派任务，不在子代理内部再拆多层子代理。

### 同步等待

每波次/每节点派发后 **须等待返回** 再推进；禁止「发了不等」。

---

## 与 code-review 的分工

| 执行者 | Skill | 职责 |
|--------|-------|------|
| 主代理 | **code-dev-loop** | 拆 phase/DAG、编排、派遣、内/外循环收敛 |
| 评审子代理 | **code-review** | readonly；spec + 风格 + DRY + 注释 + 测试 +（final 时）收尾项 |

**所有 CR 节点**（`cr-stage`、`cr-final`）须派遣 readonly 评审子代理，prompt 中 **显式要求遵循 `code-review` skill**，并使用该 skill 的输出格式与结论词（`phase-ready` / `merge-ready`）。

主代理 **不得** 用简易「对照 spec 看一眼」替代 `code-review`。

---

## 开始前检查（必须满足）

- 目标 worktree/branch 工作区干净（无未提交改动）
- 已知：`Spec path`（唯一事实来源）、`PRD path`（可选）、`Repo/worktree path`、当前分支名
- 执行 `apm read`，并用 `apm kb search --q "<需求名称/spec 关键词>"` 检索相关文档
- 分支安全闸：禁止在 `main` / `master` 直接开发；先切功能分支或独立 worktree（推荐 `feature/<topic>`、`fix/<topic>`）

**准备完成**：更新 `dynamic`（spec/PRD 路径、分支、当前 phase、inner/outer 轮次）与 `persist`（需求名称、spec 路径、分支名）。

---

## 总流程

```text
准备 → 按 spec 拆 phase → 对每个 phase：
                              [ impl → verify → cr-stage ⇄ fix-stage ] 直至 phase-ready
         → cleanup → verify-final → cr-final ⇄ fix-final 直至 merge-ready → 结束
```

---

## Step 1：按 spec 拆解阶段（phase）

根据 spec 的「详细实现步骤」「变更点清单」等，拆为 **有序 phase**（phase-1, phase-2, …）。原则：

- 每 phase 对应 spec 中一段可闭环的实现目标
- phase 之间按依赖 **串行**（phase-2 依赖 phase-1 完成且 phase-ready）
- 同一 phase 内可拆多个 **无文件冲突** 的 impl 节点并行，但 **verify 与 cr-stage 须等该 phase 全部 impl 完成**

### 阶段内节点类型

| 节点 id 模式 | 类型 | 说明 |
|--------------|------|------|
| `p<N>-impl-*` | 实现 | 本 phase 按 spec 落地代码 |
| `p<N>-verify` | 验证 | 本 phase 改动的针对性测试 + 相关 build |
| `p<N>-cr-stage` | 阶段 CR | readonly + **code-review**（`stage`） |
| `p<N>-fix-stage-*` | 修复 | 闭合 cr-stage 的 must-fix；可多次，轮次后缀递增 |

### 全局节点（全部 phase 完成后）

| 节点 id | 类型 | 说明 |
|---------|------|------|
| `cleanup` | 收尾 | 清除调试代码、无关改动、补文档、跑 lint/format |
| `verify-final` | 全量验证 | 覆盖全改动范围的测试 + 项目约定全量 build |
| `cr-final` | 全局 CR | readonly + **code-review**（`final`） |
| `fix-final-*` | 修复 | 闭合 cr-final 的 must-fix |

### 拆解示例

```text
phase-1：模块 A
  p1-impl-a          实现模块 A
  p1-verify          验证（依赖 p1-impl-a）
  p1-cr-stage        阶段 CR（依赖 p1-verify）
  p1-fix-stage-1…    按需追加

phase-2：模块 B（依赖 phase-1 phase-ready）
  p2-impl-b
  p2-verify
  p2-cr-stage
  …

全局（依赖所有 phase phase-ready）
  cleanup → verify-final → cr-final → fix-final-* …
```

**拆解完成**：更新 `dynamic`（phase 列表、各 phase 节点摘要）与 `persist`（phase 与 spec 章节映射）。

---

## Step 2：阶段内循环（每个 phase）

对当前 phase 重复以下流程，直至 **phase-ready** 或达 inner loop 上限：

### 2.1 实现（impl）

- 向子代理派发 inline 实现任务（可并行多个无冲突 impl 节点）
- **同步等待**全部返回
- 输出不合格或验证未跑：重派或补 verify，不得进入 cr-stage

### 2.2 验证（verify）

- 运行 **本 phase 改动范围** 的最小测试集 + 相关 build
- 失败 → 派 fix-stage 或重跑 impl，**不得进入 cr-stage**

### 2.3 阶段 CR（cr-stage）

- 派遣 **readonly** 评审子代理，prompt 须包含：
  - `请 readonly 执行代码评审，并严格遵循 skill：code-review`
  - `评审范围：stage`
  - 本 phase 名称、BASE_SHA、HEAD_SHA、待评文件/模块、verify 结果摘要
- **同步等待**报告

### 2.4 阶段收敛

| 结论 | 动作 |
|------|------|
| **phase-ready** | 标记本 phase 完成，进入下一 phase |
| **not phase-ready** | 提取 must-fix → 派 `fix-stage`（inline，闭合 P0/P1）→ 回到 2.2 verify → 2.3 cr-stage |
| inner loop ≥ 5 且仍 not ready | 向用户汇报未闭合项，请求拍板 |

**phase 完成**：更新 `dynamic`（phase 状态 phase-ready、HEAD_SHA）与 `persist`（本 phase 实现要点）。

---

## Step 3：收尾与全局外循环

全部 phase **phase-ready** 后：

### 3.1 收尾（cleanup）

**不得跳过。** 子代理 inline 完成：

- 清除 `console.log`、调试代码、临时代码、注释掉的废弃块
- revert 无关改动与误提交文件
- 补全 spec 要求的文档/CHANGELOG（若有）
- 运行项目 lint / format 并修复告警（项目有则必须）
- 确认工作区仅含本需求相关改动

### 3.2 全量验证（verify-final）

- 覆盖 **全部改动范围** 的测试 + 根目录 `npm run build`（及项目约定命令，如 `npm --prefix web run build`）
- 失败 → fix-final，不得进入 cr-final

### 3.3 全局 CR（cr-final）

- 派遣 readonly 评审子代理，遵循 **code-review**，`评审范围：final`
- 须检查：全量 spec 符合性、代码质量、DRY、注释、测试，以及 **E 节全部收尾项**

### 3.4 全局收敛

| 结论 | 动作 |
|------|------|
| **merge-ready** | 见「完完全全 merge-ready」清单，更新 dynamic/persist，结束 |
| **not merge-ready** | must-fix → fix-final → verify-final → cr-final |
| outer loop ≥ 3 且仍 not ready | 向用户汇报，请求拍板 |

---

## 完完全全 merge-ready

**低于此标准不得结束 skill。** 须同时满足：

1. 所有 phase 均已 **phase-ready**
2. **cleanup** 节点已完成（非口头宣称）
3. **verify-final** 全量测试与 build 已通过
4. **cr-final** 结论为 **merge-ready**（P0 = 0，P1 = 0）
5. `code-review` **E 节收尾项** 逐项 ✅（调试已清、无无关改动、文档已更、lint 已过、工作区干净）
6. **无**「合并后再做」「后续 PR 再修」类遗留 must-fix

通过后主代理用中文给出：完成摘要、主要提交、集成选项（PR / merge / cleanup worktree）。

---

## Step 4：任务编排规则

1. **校验 DAG**：无环；phase 顺序与 spec 一致
2. **并发**：仅同 phase 内、无共享文件冲突的 impl 可并行
3. **串行**：verify / cr-stage / fix-stage 须按依赖顺序；phase 之间串行
4. **输入绑定**：每节点 prompt 含 spec 路径、repo、分支、上游产出、当前 must-fix
5. **记录状态**：`dynamic` 维护当前 phase、inner/outer 轮次、节点状态、HEAD_SHA

---

## 派遣约束

每条实现/修复/验证/收尾类 prompt **开头**须含「语言要求」块：

```text
【语言要求】
- 全程使用中文：任务说明、执行过程、结论、返回报告
- 代码注释与公开 API 文档注释（JSDoc/TSDoc、JavaDoc、Python docstring 等）必须使用中文
- commit message 使用中文
- 仅代码标识符、协议字段名、第三方 API 等保持英文
```

### 子代理输出契约

**实现 / 修复 / 收尾类**须返回：

1. 已完成项与剩余缺口
2. 提交记录（sha + message）
3. 验证命令及通过/失败
4. 阻塞项（如有）

**验证类**须返回：

1. 执行的命令及通过/失败
2. 失败详情
3. 是否可进入下游 CR（是/否及原因）

**CR 类**：遵循 **code-review** 输出格式，不得简化。

---

## 开发规范：中文注释与文档注释

实现/修复类节点须遵守；评审按 **code-review** 执行。

| 层级 | 要求 |
|------|------|
| 行内/块注释 | 中文；解释「为什么」 |
| 模块/文件头 | 中文；职责边界、不变量 |
| 公开 API | 中文文档注释（JSDoc/JavaDoc/docstring 等） |

英文说明性注释、缺公开 API 文档 → **P1 must-fix**（阶段 CR 即须拦截）。

---

## 阶段记忆更新

| 阶段 | dynamic | persist |
|------|---------|---------|
| 准备完成 | spec/PRD、分支、phase 列表 | 需求名称、spec 路径、分支 |
| 每 phase phase-ready | 当前 phase、HEAD_SHA、must-fix 闭合情况 | 各 phase 实现要点 |
| cleanup / verify-final 完成 | 收尾与全量验证结果 | — |
| merge-ready | 状态、集成选项 | 完成摘要、主要提交 |

---

## 验证规则（禁止口头通过）

- 阶段 verify：本 phase 最小测试集 + 相关 build
- verify-final：全改动范围测试 + 项目约定全量 build
- CR 节点：必须派子代理 + **code-review**，主代理不得自评通过
- 无 cleanup 记录不得 cr-final

---

## 失败处理

**子代理调用失败**：停顿后重试一次；仍失败则主代理 readonly 手工对照 **code-review** 维度整理 must-fix，标注「手工评审」，下轮恢复子代理。

**节点输出不合格**：不进入下游；从 verify 或 fix 重跑。

**实现偏离 spec**：**先更新 spec** 再改 DAG 与实现。

---

## Prompt 模板

### 实现 / 修复类节点

```text
【语言要求】
（见上文块）

请以 inline 模式完成下列任务。

仓库/worktree：<PATH>
分支：<BRANCH>
Spec：<SPEC_PATH>
阶段：<PHASE_NAME>
节点：<NODE_ID> — <描述>
上游产出：<...>
本轮 must-fix（来自 cr-stage / cr-final）：
- ...

约束：
- 单次运行内完成；按逻辑块提交
- 运行相关针对性测试/构建
- 不 revert 无关改动
- 中文注释与公开 API 文档注释

请用中文返回：1）已完成项 2）提交 3）验证结果 4）阻塞项
```

### 验证类节点

```text
【语言要求】
（见上文块）

请对下列范围运行验证。

仓库/worktree：<PATH>
Spec：<SPEC_PATH>
范围：<phase 名 | 全量>
涉及文件/模块：<...>

请用中文返回：1）命令及结果 2）失败详情 3）是否可进入 CR
```

### 收尾类节点（cleanup）

```text
【语言要求】
（见上文块）

请 inline 完成本需求收尾（为 merge-ready 做准备）。

仓库/worktree：<PATH>
分支：<BRANCH>
Spec：<SPEC_PATH>
全量改动范围：<文件/模块列表>

必须完成：
- 清除调试代码与临时代码
- revert 无关改动
- 补全 spec 要求的文档更新
- 运行 lint/format 并修复（项目有则必须）
- 工作区干净

请用中文返回：1）收尾项逐项说明 2）提交 3）lint/format 结果 4）阻塞项
```

### 阶段 / 全局 CR 节点

**直接使用 `code-review` skill 中的「评审子代理 Prompt 模板」**，不得省略 skill 引用与输出格式。

---

## 执行检查清单

- [ ] 已按 spec 拆 phase，且每 phase 含 impl → verify → cr-stage 内循环
- [ ] cr-stage / cr-final 均派 readonly 子代理并遵循 **code-review**
- [ ] 每 phase phase-ready（P0/P1 = 0）后才进入下一 phase
- [ ] 全部 phase 完成后已执行 **cleanup** 与 **verify-final**
- [ ] cr-final 为 merge-ready，且 E 节收尾项全部满足
- [ ] 无「合并后再收尾」类遗留
- [ ] 已更新 dynamic / persist

---

## 备注

- 文档默认位于 `.apm/kb/docs/Iterations/<需求名称>/`（`prd.md`、`spec.md`）
- 同文件多 impl 默认串行或分 phase，避免并发冲突
- 子 agent prompt 未含「语言要求」或未在 CR 时引用 **code-review** → 视为不合规派遣
