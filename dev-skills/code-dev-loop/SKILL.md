---
name: code-dev-loop
description: 基于 spec 的分阶段 DAG 编排。默认由子代理执行探索/实现/验证/CR（主代理上下文有限）；主代理只编排、维护 Context Bundle、汇总 must-fix。每 phase 内循环 CR，全局 merge-ready 须独立 readonly 评审。compact 模式仅作极少数例外。
disable-model-invocation: true
---

# Code Dev Loop

## 目的

在 **spec 为唯一事实来源** 的前提下，通过「分阶段拆解 → 阶段内实现与 CR 内循环 → 收尾 → 全局 CR 外循环」完成实现。

**为何默认派子代理**：主代理上下文有限；读多写广的探索、实现、验证、独立 CR 由子代理执行并**摘要回报**，主代理保留编排与决策，避免后期遗忘契约或自审通过。

主代理与子代理的协作、说明、评审结论**一律使用中文**（代码标识符、协议字段、第三方 API 名称等除外）。

---

## 主代理 vs 子代理（核心分工）

| 主代理（编排者） | 子代理（执行者） |
|------------------|------------------|
| 选 mode、拆 phase、维护 **Context Bundle** | explore / impl / verify / fix / cleanup |
| 派任务、同步等待、汇总 must-fix | readonly **code-review**（`stage` / `final`） |
| 更新 dynamic / persist | inline 完成所派节点并中文回报 |
| 向用户汇报、请求拍板 | **禁止**在 pipeline 中自评 merge-ready |

**硬规则**：

1. **默认**：impl / verify / fix / cleanup **须派子代理**
2. **cr-stage / cr-final 永远须 readonly 评审子代理** + `code-review` skill；主代理 **不得** 因「我实现的我很熟」而自审
3. **禁止**以省事为由跳过子代理直接改代码——除非符合 **compact** 且已在 dynamic 声明（见下）

---

## 执行模式（mode）

开始前选定 mode，写入 `dynamic`（含选择理由）：

| mode | impl / verify / fix | CR | 何时选用 |
|------|---------------------|-----|----------|
| **strict**（默认） | 全部子代理 | 子代理 readonly | 多 phase、广探索、大 diff、主对话上下文已胀 |
| **compact** | 主代理可 inline impl+verify+fix | **仍须**子代理 cr-stage / cr-final | **同时满足**：单 phase、强耦合链、≤5 核心文件、接口须一次对齐 |
| **review-only** | 已完成（补救） | 子代理 cr-final | 代码已存在，仅补独立终审 |

**路由**：

- 小 bug / 迭代内小 feature、先实现后留痕 → **`agile-dev`**，非本 skill
- 有 execute-ready spec、正式交付 → **本 skill**（默认 **strict**）

未在 dynamic 声明 mode 即跳过子代理 → **视为违规**。

---

## 关键术语

### phase（阶段）

spec 中一段可独立评审的工作单元。**允许仅 1 个 phase**（仍须完整 inner loop）。

### 阶段内循环（inner loop）

```text
impl → verify → cr-stage（子代理 + code-review）─not ready─→ fix → verify → cr-stage …
                                              └─ phase-ready ─→ 下一 phase 或全局收尾
```

- 每 phase 默认 **最多 5 轮** inner loop

### 全局外循环（outer loop）

```text
cleanup → verify-final → cr-final（子代理 + code-review）─not ready─→ fix-final → …
                                                    └─ merge-ready ─→ 结束
```

- 默认 **最多 3 轮** outer loop

### Context Bundle（派子代理前必建）

主代理维护的**窄上下文包**，每次派子代理时**整包粘贴**，避免重贴全文 spec / 整段对话：

```text
【Context Bundle】
- 迭代 / 分支 / Spec 路径
- Spec 摘要（≤15 行：目标、变更点、验收要点）
- 已拍板决策（≤7 条）
- 当前 phase 与节点 id
- 本任务文件清单 + 接口/类型契约（强耦合链须写清）
- 不变量与禁止改动
- 上轮 must-fix 闭合状态
- 相关 HEAD_SHA
```

Bundle 更新时机：phase 切换、cr 产出 must-fix 后、fix 完成后。

### inline / 同步等待

子代理单次连续完成所派任务；派发后 **须等待返回** 再推进。

---

## 与 code-review 的分工

| 节点 | 执行者 | Skill |
|------|--------|-------|
| cr-stage / cr-final | **readonly 评审子代理**（mandatory） | **code-review** |
| impl / verify / fix / cleanup | 子代理（compact 下 fix 亦可子代理） | 本 skill |

prompt 须含：`请 readonly 执行代码评审，并严格遵循 skill：code-review` + **Context Bundle**。

---

## 开始前检查

- 工作区干净；不在 `main` / `master` 上开发
- 已知 Spec（唯一事实来源）、PRD（可选）、repo/worktree、分支名
- `apm read` + `apm kb search`
- 选定 **mode** 并写入 dynamic

**准备完成**：更新 `dynamic`（mode、spec/PRD、分支、phase、轮次）与 `persist`（需求名称、spec 路径、分支）。

---

## 总流程

```text
准备 → 建 Context Bundle → 按 spec 拆 phase（可 1 个）
  → 每 phase：[ impl → verify → cr-stage ⇄ fix ] 直至 phase-ready
  → cleanup → verify-final → cr-final ⇄ fix-final 直至 merge-ready → 结束
```

---

## Step 1：拆 phase 与耦合启发式

根据 spec「详细实现步骤」「变更点清单」拆 **有序 phase**。除 **compact** 外，**至少 1 个 phase、至多按 spec 自然段拆**，不强行凑 3 phase。

### 耦合与 impl 编排（默认 strict）

| 情况 | 编排 |
|------|------|
| 模块间无共享文件、接口已冻结 | 同 phase 内多个 **impl 子代理可并行** |
| 共享类型/bridge/接口跨 **≥3 文件** 须同步改 | **单 impl 子代理** 包整条耦合链，**禁止**并行 impl |
| 同一文件多改动点 | **串行**，禁止多子代理并发写同一文件 |
| 需广读 codebase 再改少量文件 | 先派 **explore 只读子代理**，再 impl |

**compact 例外**：强耦合链可由 **主代理一次 inline impl** 完成整条链，但 verify 仍建议子代理；**cr-stage 不得省略**。

### 节点类型

| 节点 | 执行者 | 说明 |
|------|--------|------|
| `p<N>-explore` | 子代理 readonly | 按需：改动前广探索 |
| `p<N>-impl` | 子代理 inline（compact：主代理） | 本 phase 实现 |
| `p<N>-verify` | 子代理 | 本 phase 最小测试 + build |
| `p<N>-cr-stage` | **评审子代理 readonly** | **code-review** `stage` |
| `p<N>-fix-*` | 子代理（compact：主代理可 fix） | 闭合 must-fix |
| `cleanup` / `verify-final` | 子代理 | 收尾与全量验证 |
| `cr-final` | **评审子代理 readonly** | **code-review** `final` |
| `fix-final-*` | 子代理 | 闭合 final must-fix |

### 示例：强耦合链（推荐）

```text
phase-1（单 phase 即可）
  p1-explore（可选）
  p1-impl          单个子代理：A→B→C 整条链
  p1-verify
  p1-cr-stage      readonly 评审子代理
  p1-fix-1…        按需

全局：cleanup → verify-final → cr-final → fix-final…
```

---

## Step 2：阶段内循环

### 2.1 实现（impl）

- **strict**：派 impl 子代理，prompt 含 **Context Bundle** + 语言要求
- **compact**：主代理 inline 前须确认 dynamic 已记录 compact 理由；仍建议 verify 派子代理
- **同步等待**；不合格不得进入 cr-stage

### 2.2 验证（verify）

- 子代理跑本 phase 最小测试集 + 相关 build
- 失败 → fix → verify，**不得** cr-stage

### 2.3 阶段 CR（cr-stage）— 不可跳过、不可自审

- **必须** readonly 评审子代理 + **code-review** `stage`
- prompt：Bundle + BASE/HEAD_SHA + 待评文件 + verify 摘要
- 实现者与评审者 **不得** 同一执行体（compact 下 impl 是主代理，则 cr **必须** 子代理）

### 2.4 收敛

| 结论 | 动作 |
|------|------|
| **phase-ready** | 下一 phase 或进入全局收尾 |
| **not phase-ready** | must-fix → fix（子代理或 compact 主代理）→ verify → cr-stage |
| inner ≥ 5 | 向用户汇报未闭合 P0/P1 |

---

## Step 3：收尾与全局外循环

### 3.1 cleanup（子代理，不可跳过）

清调试代码、revert 杂项、补文档、lint/format、工作区干净。

### 3.2 verify-final（子代理）

全改动范围测试 + 项目约定全量 build。

### 3.3 cr-final（readonly 评审子代理，不可跳过）

**code-review** `final`：全量 spec + 质量 + **K 节收尾项**。

### 3.4 完完全全 merge-ready

1. 各 phase **phase-ready**
2. cleanup / verify-final 已完成（子代理回报为据）
3. **cr-final** = merge-ready（P0/P1 = 0）
4. **K 节收尾项** 全部 ✅
5. 无「合并后再做」类遗留

---

## Step 4：编排规则

1. DAG 无环；phase 串行
2. 默认 **子代理执行**；compact 须在 dynamic 留痕
3. 每派子代理 **必附 Context Bundle**
4. **CR 永远子代理 readonly**
5. `dynamic` 记录：mode、phase、轮次、节点状态、HEAD_SHA、must-fix

---

## 派遣约束

### 语言要求（每条执行类 prompt 必含）

```text
【语言要求】
- 全程使用中文：任务说明、执行过程、结论、返回报告
- 代码注释与公开 API 文档注释必须使用中文
- commit message 使用中文
- 仅代码标识符、协议字段名、第三方 API 等保持英文
```

### 子代理输出契约

**impl / fix / cleanup**：已完成项、提交（sha+message）、验证结果、阻塞项  
**verify**：命令及结果、是否可进入 CR  
**CR**：完整 **code-review** 格式，不得简化

---

## 开发规范：中文注释

见 **code-review** F 节；阶段 CR 即拦截 P1。

---

## 失败处理

**子代理调用失败**（如 `resource_exhausted`）：

1. 停顿后 **重试该节点一次**
2. 仍失败：主代理 **readonly** 按 **code-review** 维度整理 must-fix，标注 `manual-review`；**不得** 主代理代为 impl 除非 **compact** 且已声明
3. 下轮优先恢复子代理

**禁止**：以失败处理为由，在 **strict** 模式下长期由主代理替代 impl/verify。

**实现偏离 spec**：先更新 spec，再改 DAG。

---

## Prompt 模板

### 通用前缀（执行类）

```text
【语言要求】
（见上）

【Context Bundle】
（粘贴当前 Bundle）

请以 inline 模式完成下列节点。
仓库/worktree：<PATH>
分支：<BRANCH>
节点：<NODE_ID> — <描述>
本轮 must-fix：
- ...
```

### impl 子代理（strict）

```text
（通用前缀）

任务：按 Spec 实现本 phase 变更。
Spec 路径：<SPEC_PATH>
本 phase 文件与契约：（从 Bundle）
约束：单次 inline；按逻辑块提交；跑针对性测试；中文注释与公开 API 文档。

请用中文返回：1）已完成项 2）提交 3）验证结果 4）阻塞项
```

### cr-stage / cr-final

使用 **code-review** skill 中的「code-dev-loop 阶段 / 合并前 CR」模板 + **Context Bundle**。

---

## 阶段记忆更新

| 阶段 | dynamic | persist |
|------|---------|---------|
| 准备 | mode、spec、分支、phase 列表 | 需求名称、spec 路径 |
| 每 phase ready | phase 状态、HEAD_SHA、must-fix | 各 phase 要点 |
| merge-ready | 验证与 CR 结论、集成选项 | 完成摘要、主要提交 |

---

## 执行检查清单

- [ ] 已选 mode 并写入 dynamic（compact 须写理由）
- [ ] 已建 Context Bundle，每次派子代理已附带
- [ ] strict：impl / verify / fix / cleanup 均为子代理
- [ ] cr-stage / cr-final 均为 readonly 子代理 + **code-review**（**即使 compact 也不得自审**）
- [ ] 强耦合链用 **单 impl 子代理** 或 compact 主代理，未错误并行
- [ ] 各 phase phase-ready 后才进入下一阶段
- [ ] merge-ready 含 K 节收尾项，无遗留
- [ ] 已更新 dynamic / persist

---

## 备注

- 文档默认：`.apm/kb/docs/Iterations/<需求名称>/`（`prd.md`、`spec.md`）
- 主对话上下文 **>50% 饱和** 时，**禁止** 继续主代理 impl，须改 **strict** 并派子代理
- 未附 Bundle、CR 非 readonly 子代理、compact 未声明 → **不合规派遣**
