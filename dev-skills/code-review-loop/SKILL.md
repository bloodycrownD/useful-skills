---
name: code-review-loop
description: 基于 PRD/spec 的评审 DAG：review / fix / verify / cleanup-check 波次并行，not-ready 时动态重编排，直至 review-ready。含 func 模式用于波次内功能小检。可独立 diff 评审。默认子代理执行。
disable-model-invocation: true
---

# Code Review Loop

**质量评审** skill：对照 **PRD + spec**（及用户提供的 diff 范围），拆解 **评审 DAG**（review / fix / verify / cleanup-check），波次并行、not-ready 时动态重编排。可独立用于 PR 评审，也可在用户指定的代码变更范围内运行。终点 **review-ready**。

主代理 = 编排；子代理 = review / fix / verify / cleanup-check。中文协作。

---

## 主代理禁止

- **自审**：review* 须 readonly 子代理，主代理不得代替 review 节点做评审
- **自改代码**：fix 须 Task 子代理在单次调用内连续执行，主代理不得直接编辑代码
- **自跑 verify / cleanup-check**：须 Task 子代理执行测试/build/lint/format，主代理不得自跑替代
- **not-ready 时口头 fix**：不得只描述修复方案而不派 fix 子代理；须更新 `wave_plan` 并 `dag_version++`

---

## 边界

| 本 skill | 不做 |
|----------|------|
| 按 spec 写代码、拆开发 DAG | impl / 开发编排 |
| 审查 PRD/SPEC 文档是否 execute-ready | 文档收敛（属 spec 审查类任务） |

---

## 关键术语

### review-ready

评审终点（用户语境下常等价于「可合并」）。须同时满足：

- 末轮 **full** 评审：P0=P1=P2=0，K 节 ✅
- **spec_deviations** 无 open
- **Review Closure** 表已附
- spec 中 `qa: manual_user` **不阻塞**（写入 C 类「合并后请你验收」）

### func-ready

**func** 模式结论，用于**波次内功能小检**。须同时满足：

- 本波次 spec 步骤矩阵 ✅、blocking 测试 id ✅、verify 证据 ✅
- 本波次 **must-fix：P0=P1=0**（func **不查 C 维**，不产生 P2）
- 无 open **spec_deviations**
- `qa: manual_user` **不阻塞** func-ready

### scope-ready

**scope** 模式结论，用于**分模块评审 DAG**。须同时满足：

- 本 scope 已查维度内 **P0=P1=P2=0**（见下「scope 检查范围」）
- 无 open **spec_deviations**（本 scope 相关）
- **不含** D/E/F/H–J/K 的全局结论——未查维度由 **cleanup-check**（K）与 **review-full**（D–K 全局）负责

**scope 检查范围**（仅此范围内的缺陷计入 scope-ready）：A（本 scope）+ B + C + **C-orch**（本 scope 文件）+ G（本 scope 测试）。

### func vs full（增量策略）

- **func**：只查 **A + G + spec_deviations**；must-fix 仅 **P0/P1**；**不产生 P2**
- **scope**：查 **A + B + C + C-orch + G**（本 scope 文件）；可产生 **P2**（C 维）
- **full**：查 **B–K 全维**（含 **C-orch**）

**full 模式 A/G 抽检**（须同时满足才可抽检，否则 full 须全查 A/G）：

1. Context Bundle 含 `func_context`：`func_node_ids`、`func_files`、`func_head_sha`
2. 对应 func 节点均已 **func-ready: yes**
3. 当前 `HEAD_SHA` = `func_head_sha`，且变更文件未超出 `func_files` 并集

不满足以上任一条 → review-full **须全查 A/G**。

### 编排收敛（C-orch）

**编排收敛（Orchestration Consolidation）**：对**同一条业务事务**，减少**平行路径、多余转手、分散的协调点**，使「谁发起、谁推进、谁收尾」可读、可测、可单点修改；**不改变**对外可观察行为。

**不是**把 `A → … → Z` 一律压成 `A → Z`。须区分：

| 类型 | 说明 | 评审态度 |
|------|------|----------|
| **必要 hop** | 平台边界（IPC/沙箱）、序列化 DTO、性能合批、清晰的分层职责 | 保留；须在注释或 spec 写明「为何不能扁平」 |
| **偶然 hop** | 平行装配入口、无事务语义的 ref/Map 透传、同语义多信号混用、死通道、第 N 份 unwrap | **收敛目标** |

与 **DRY** 区分：DRY 反对**同一条知识**写多遍；C-orch 反对**同一件事走多条路径**、状态在函数间**抛来抛去**（路径重复 ≠ 代码行重复）。

**适用**：多文件协调、状态机、跨端入口、flush/pipeline、生命周期递减等改动。**不适用**时标注 **N/A**（纯样式、单函数算法、无跨函数事务）。

---

## 评审模式

| 模式 | 典型场景 | 结论 |
|------|----------|------|
| **func** | 波次内功能小检 | func-ready / not func-ready |
| **scope** | 分模块评审 DAG | scope-ready / not scope-ready |
| **full** | 末轮/global | review-ready / not review-ready |
| **diff** | 独立 PR，单轮 | 通过 / 需修改 / 阻塞 |

review* 节点 **须 readonly**；实现者 **不得**自审。

---

## 子代理派遣规范

| 节点 | 工具 | subagent_type | readonly | 说明 |
|------|------|---------------|----------|------|
| **review** / **review-scope** / **review-full** | Task | generalPurpose | **true** | 非自审 |
| **fix** | Task | generalPurpose | false | **必须**子代理；闭合 must-fix（含 P2） |
| **verify** | Task | generalPurpose 或 shell | false | **必须**子代理；测试/build |
| **cleanup-check** | Task | generalPurpose 或 shell | false | **必须**子代理；K 节 lint/format/清调试 |

- fix / verify / cleanup-check **必须** Task 子代理；主代理 **不得** inline 改代码、不得自跑 lint/test 替代子代理
- 同 wave 无冲突可并行；**同步等待** 当前 wave 全部返回后再汇总

---

## 动态 DAG

not-ready 后 **改图**，`dag_version++`：

| 动作 | 示例 |
|------|------|
| 并行 fix | 不同文件的 P1/P2 → 同 wave |
| 合并 review | 多模块同类 → review-scope 一次 |
| 插入 verify | fix wave 之后 |
| cleanup-check | verify 之后、review-full 之前 |
| cleanup 失败后重跑 | fix-cleanup → verify → cleanup-check → review-full |

```yaml
dag_version: 1
wave_plan: [[review-scope-a, review-scope-b], [fix-x, fix-y], [verify-all], [cleanup-check], [review-full]]
node_status: { review-scope-a: done, fix-x: pending }
must_fix: []   # 即 open_must_fix；项须含 id, P0|P1|P2, 文件, 维度, 改法, 来源 node_id
```

**cleanup-check 失败**：将其 must-fix 并入 `must_fix` → 插入 `fix-cleanup` wave → `verify` → 再跑 `cleanup-check` → 通过后方可 `review-full`。不得跳过 verify 直接重跑 cleanup。

**禁止**：fix 后不重跑 review；not-ready 只串行一个 fix 而不改 wave_plan。

---

## 节点类型与典型子图

| 类型 | 执行者 |
|------|--------|
| **review** / **review-scope** / **review-full** | readonly |
| **fix** | 子代理（Task 内连续执行） |
| **verify** | 子代理 |
| **cleanup-check** | 子代理 |

```text
review-scope-*（并行 wave）
  → fix-*（并行 wave）
  → verify
  → cleanup-check
  → review-full
  → review-ready（结论态，非波次节点）
```

末轮全局节点类型为 **review-full**；**review-ready** 为 DAG 终点结论，不单独派遣子代理。

**scope** 检查维度：A（本 scope）+ B + C + **C-orch** + G（本 scope 测试）；D/E/F/H–J/K 留 **review-full / cleanup-check**。

---

## 检查维度

### func

- **A** 本波次 spec 步骤矩阵、blocking 测试 id
- **G** verify 证据
- 无 open **spec_deviations**
- **must-fix：P0=P1=0**（不查 C，不产生 P2）

### full / review-ready（B–K；A/G 见增量策略）

**A.** 需求符合性 — 变更点、步骤矩阵、测试矩阵 T-xx、PRD 验收、范围蔓延

**B.** 正确性 — 边界、错误处理、资源释放、类型契约

**C.** 质量 — 风格、**DRY**、命名、复杂度、职责、死代码

| 检查项 | 严重度参考 |
|--------|------------|
| 死代码、明显 **DRY** 违反、职责越界、与项目惯例冲突的复杂度 | **P1** |
| 命名不顺、风格不一致、可读性小改、局部小 refactor（**改法明确**） | **P2** |
| 已认定有问题且改法明确 | **P1/P2**，写入 must-fix |
| 尚未认定有问题，或问题是否在 scope 内 | **不得写入 must-fix** → `open_questions` / `blocked` |

**C 维**：凡写入 **must-fix** 的项 = 已认定相对 PRD/spec/惯例**有问题**；须标 P1 或 P2 并附改法。禁止「只批评、不认定、不进 must-fix」。

**C-orch. 编排收敛**（多文件/跨端协调改动时适用，否则 **N/A**；见「关键术语 · 编排收敛」）

| 检查项 | 严重度参考 |
|--------|------------|
| **平行入口**：新增第 N 条装配/旁路且无 parity 测试 | P1；**升 P0**：已有红测证明行为分叉，或双写路径无 parity 覆盖 |
| **无意中间态**：新 ref/Map/队列仅透传、无明确事务阶段语义 | P1 |
| **信号混用**：同语义多源消费且无映射表或文档 | P1；**升 P0**：已观测双减/竞态/重复触发 |
| **假扁平化**：为减层数破坏平台边界（沙箱/IPC）、事务边界或 spec 明示的编排阶段 | **P0** |
| **收敛无证据**：合并编排链但无相关测试、T-xx 或 PRD 验收 id 覆盖 | P1 |

与 **A**：PRD/spec 若写明「单点装配」「编排单入口」等，未满足 → **A + C-orch** 同时记 must-fix。与 **G**：parity/集成测为编排收敛的**门禁证据**。

**D.** 安全 — 注入、鉴权、密钥、输入校验（适用时）

**E.** 性能 — 明显 N+1、热点 IO（适用时）

**F.** 中文注释 — 关键逻辑中文注释；公开 API 文档注释；英文说明→P0

**G.** 测试 — 行为测试、边界、失败路径；verify 已通过

**H–J.** 兼容性、可观测性、UI（适用则查，否则标注未涉及）

**K.** 收尾 — 无调试残留、无杂项改动、lint/format、工作区干净、spec 文档已更

**manual_user**：**不阻塞** func-ready / review-ready → C 类「合并后请你验收」

### spec_deviations

open → 不得 func-ready / scope-ready / review-ready。

**流转**：review 子代理标记 `open`（附描述）→ fix 或用户决策后标 `fixed`（附证据或用户确认原文）→ 下一轮 review 验证。用户确认「按现状收窄 spec」视同 `fixed`。

---

## 严重度

| 级别 | 含义 | fix |
|------|------|-----|
| **P0** | 阻塞：正确性、安全、spec 违背、假扁平化等 | 必须 |
| **P1** | 结构性质量：死代码、明显 DRY、C-orch、职责越界等 | 必须 |
| **P2** | **可明确优化的点**（风格、命名、小 refactor）；**不是**不确定或待商榷项 | 必须 |

- **P0/P1/P2 均须闭合**才 scope-ready / review-ready（**func-ready** 仅要求 P0=P1=0，不产生 P2）
- **must-fix = 已认定有问题**；认定了就必须标 P0/P1/P2 并附改法，**不得标 blocked**
- P2 须写清**具体改法**（改什么、改成什么）；写不出改法 → 尚未认定有问题，不得进 must-fix

### must-fix vs open_questions / blocked

**原则**：CR **查出来且写入 must-fix** 的 = 原方案（相对 PRD/spec/惯例）**有问题**，闭环内**必须修**。不存在「有问题但不知道该不该改」的 must-fix。

| 输出 | 含义 | 是否修 |
|------|------|--------|
| **must-fix**（P0/P1/P2） | 已认定有问题 + 有改法（P2 亦须写清改法） | 必须 fix |
| **open_questions** | 观察/疑点，**尚未认定**有问题；或问题在 scope 外 | 不进 fix，列给用户 |
| **blocked** | 调查未完成、spec 空白待补、fix 往返 ≥3 次仍合不拢 | 暂停 ready，请用户 |

**术语消歧**：DAG **`blocked`**（流程卡死）≠ diff 结论「**阻塞**」（未闭合 P0 / open spec_deviations）。

**open_questions 典型**（不是「原方案有问题」，故不进 must-fix）：

- 邻接文件类似写法，**不在本 PR diff**，是否扩 scope 一并改？
- spec/PRD 未覆盖的行为，当前实现**尚不能说错**，需补 spec 再评？
- 需更多上下文才能判断是否为问题（只读调查未完成）

**曾易混淆、现规则**：

| 场景 | 正确处理 |
|------|----------|
| 命名丑、风格不统一 | 有问题 → **P2** + 改法 → must-fix |
| 两种写法都能用，但项目**有惯例** | 违背惯例 = 有问题 → **P2** |
| 两种写法都能用，**且无惯例**，当前实现可接受 | **不写入 must-fix**；可记 open_questions「是否建立惯例」 |
| 确定是 bug / 违背 spec | **P0/P1** → must-fix |
| 改了会动行为且 spec 未规定 | 先记 **spec_deviations** 或 open_questions；**不要**假装成 P2 |

**简记**：**认定有问题 → must-fix；没认定 → 别进 must-fix。**

---

## 总流程

```text
准备 → 读 PRD/spec/变更范围 → 初始评审 DAG
  → loop:
      取就绪 wave → 并行派子代理 → 同步等待 → 更新 node_status
      → review-ready? ─是→ Review Closure 表 → 结束
      → not-ready? ─→ 重编排（dag_version++）→ 继续
```

同一 must-fix 震荡 ≥3 次 → **blocked**，请用户拍板。

**diff**：`diff` 评审（只读）→ **cleanup-check（必跑）** → 汇总三态；无 review-full 循环。K 节 ✅ 以 cleanup-check 结果为准。需改则由用户决定是否进入 fix loop。

---

## Step 1：初始拆解

- 按模块/检查域拆 review-scope 或 review-full
- 预置 fix/verify/cleanup 占位
- 小 diff + 用户明示 → 可 diff 单轮或精简为 review-full + cleanup-check

---

## Step 2：波次执行

1. 当前 wave **全部**节点 **并行**派发 Task 子代理
2. **同步等待** 全部返回后再汇总
3. 并行 **review-scope-\*** 返回后：合并 `must_fix`（见下）
4. 任一 fail / review* not ready → **not-ready**

**并行 scope 合并规则**：

- id 格式：`{scope-id}/{维度}-{序号}`（例 `scope-a/C-02`）
- 同 id 去重；同文件同维度多条 → 保留**最高严重度**（P0 > P1 > P2）
- 同级冲突 → 合并为一次 `review-scope` 复检或标 `blocked`

**fix vs verify 测试边界**：

- **fix**：与改动直接相关的**冒烟/局部**测试（确认改动不显然坏）
- **verify**：本波次**正式门禁**（相关 suite/build 按项目惯例完整执行）
- fix 已绿**不免除** verify；verify 失败 → 新 must-fix → fix → verify

各节点 prompt 要点：

- **review***：readonly + 本 skill + 模式 + PRD/spec 路径
- **fix**：闭合 must-fix（含 P2；须引用来源 review 节点 id / must-fix id）
- **verify**：相关测试/build
- **cleanup-check**：K 节

---

## 失败处理

- 子代理失败：**重试一次**
- 仍失败 → **not-ready**，主代理重编排 DAG 或标注 **blocked**
- 同一 must-fix 震荡 ≥3 次 → **blocked**，请用户拍板

---

## Step 3：重编排

收集 `must_fix`（含 P2）→ 并行 fix → verify →（若上轮 cleanup 失败或 fix 触达 K 相关项）cleanup-check → 合并 review → `dag_version++`

cleanup-check 失败 → `fix-cleanup` → verify → cleanup-check → 通过后 review-full。

---

## Context Bundle 规范

主代理向子代理传递的 YAML，**至少**含：

```yaml
repo_path: <绝对路径>
base_sha: <sha>
head_sha: <sha>
mode: func | scope | full | diff
node_id: <review-scope-a | review-full | ...>
files: [<本节点范围文件>]
spec_path: <路径或「未提供」>
prd_path: <路径或「未提供」>
prior_conclusion: <上轮结论，首轮填 none>
must_fix: []          # 当前 open 项
func_context:         # full 模式 A/G 抽检时必填；否则省略
  func_node_ids: []
  func_files: []
  func_head_sha: <sha>
```

fix 子代理用 **delta**  subset：相对上轮新增的 `must_fix`、变更文件、`head_sha`。

---

## Prompt 模板

### func

```text
【语言要求】全程中文

readonly 评审。skill：code-review-loop。模式：func
仓库：<REPO_PATH>
Spec：<SPEC_PATH>
PRD：<PRD_PATH>
本波次范围文件：<FILES>
verify 摘要：<VERIFY_SUMMARY>
Context Bundle：<YAML>

检查：A（本波次步骤矩阵+blocking 测试 id）、G（verify 证据）、spec_deviations
注意：func 不查 C 维，must-fix 仅 P0|P1，不产生 P2。

请用中文返回：
1）步骤/测试矩阵对照（pass/fail + 证据）
2）must-fix 列表（id, P0|P1, 文件, 描述）
3）spec_deviations（open/fixed/none）
4）结论：func-ready: yes|no + 一句话理由（须 P0=P1=0、无 open spec_deviations）
```

### scope

```text
【语言要求】全程中文

readonly 评审。skill：code-review-loop。模式：scope
仓库：<REPO_PATH>
Spec / PRD：<路径>
BASE_SHA / HEAD_SHA：<sha>
节点 id：<review-scope-x>
检查维度：A + B + C + C-orch + G（本 scope 文件）；不含 D/E/F/H–J/K
C-orch：本 scope 多文件协调改动时必查；纯局部改动标 N/A 并简述理由。
Context Bundle：<YAML>

请用中文返回：
1）相对上轮：已修复 / 仍开放（第 2 轮起）
2）must-fix（id, P0|P1|P2, 文件, 描述, 维度, 改法；含 C-orch 时注明检查项）
   - 写入 must-fix = 已认定有问题；C 维须 P1 或 P2 + 具体改法
3）open_questions（如有）— 未认定有问题 / scope 外；**不得**混入 must-fix
4）spec_deviations
5）未涉及维度：D/E/F/H–J/K=本 scope 未查（由 full/cleanup 负责）
6）结论：scope-ready: yes|no + 理由（须本 scope 已查维度 P0=P1=P2=0）
   - **禁止**输出 review-ready；scope 节点不得宣布全局可合并
```

### full

```text
【语言要求】全程中文

readonly 评审。skill：code-review-loop。模式：full
仓库：<REPO_PATH>
Spec / PRD：<路径>
BASE_SHA / HEAD_SHA：<sha>
节点 id：review-full
检查维度：B–K 全维（含 C-orch）；A/G 增量见 skill「func vs full」
C-orch：多文件协调改动时必查；纯局部改动标 N/A 并简述理由。
Context Bundle：<YAML>（含 func_context 时方可抽检 A/G）

请用中文返回：
1）相对上轮：已修复 / 仍开放（第 2 轮起）
2）must-fix（id, P0|P1|P2, 文件, 描述, 维度, 改法；含 C-orch 时注明检查项）
3）open_questions（如有）
4）spec_deviations
5）未涉及维度：H/I/J=未涉及（适用则查，列出 pass/fail）
6）A/G 策略：全查 | 抽检（列 func_node_ids + 抽检证据）
7）结论：review-ready: yes|no + 理由（须 P0=P1=P2=0；K 节以末轮 cleanup-check 为准）
   - review-ready: yes 时须附 Review Closure 表字段摘要
```

### fix

```text
【语言要求】
- 全程使用中文：任务说明、过程、结论、返回报告
- 代码注释与公开 API 文档注释须中文；commit message 中文
- 标识符、协议字段、第三方 API 名保持英文

节点：fix-<id>，类型：fix
来源 review：<review_node_id>
仓库/worktree：<PATH>
分支：<BRANCH>
Spec / PRD：<路径>
Context Bundle（delta）：
<粘贴 delta YAML>
must-fix：<列表>

任务：闭合 must-fix 列表（含 P2 优化项）
约束：
- 在 Task 子代理内连续执行（非主代理编辑）；按逻辑块提交；不 revert 无关改动
- 跑与改动直接相关的冒烟/局部测试；记录命令与结果（正式门禁由 verify 负责）

请用中文按下列结构返回：
1）已完成项与剩余缺口
2）提交记录（sha + message，无则写「无提交」）
3）验证命令及通过/失败
4）阻塞项（如有）
```

### verify

```text
【语言要求】全程中文

节点：verify-<id>。来源 fix：<fix_node_id>（无则写「scope 初评后」）
仓库：<PATH>。变更文件：<FILES>

任务：运行与改动相关的测试/build（按项目惯例）；记录命令与结果。

请用中文返回：
1）执行的命令与 exit code
2）通过/失败摘要
3）失败时的关键日志片段
4）阻塞项（如有）
```

### cleanup-check

```text
【语言要求】全程中文

节点：cleanup-check。仓库：<PATH>

任务：执行 K 节收尾检查（须**实际运行**项目 lint/format 命令，并检查 git 工作区）。

K 节五项（逐项 ✅/❌ + 证据）：
1）无调试残留 — console.log、debugger、临时代码、注释掉的死代码块
2）无杂项改动 — 无关文件、误提交、超出 spec/diff 范围的改动
3）lint/format 通过 — 运行项目标准命令（如 npm run lint、npm run format、eslint --fix、prettier -w）；记录命令与 exit code
4）工作区干净 — `git status` 无意外未提交文件；若有预期残留须说明
5）spec/PRD 已更 — 代码-文档契约变更时对应文档已同步（无则标注 N/A）

请用中文返回：
1）五项清单及命令输出摘要
2）已自动修复项（如有）
3）must-fix（id, P0|P1, 描述）— 未通过项
   - 调试残留/杂项改动/工作区不干净 → P1
   - lint/format 命令失败 → P1（cleanup 可自动 fix 的 format 问题修后再报）
4）结论：K-ready: yes|no
```

### diff

```text
【语言要求】全程中文

readonly 评审。skill：code-review-loop。模式：diff（独立 PR，**单轮**）
仓库：<REPO_PATH>
Spec / PRD：<路径，无则写「未提供」>
Diff 范围：
- BASE_SHA / HEAD_SHA：<sha>（或 PR #<n> / compare URL）
- 变更文件：<FILES 或 git diff --name-only 输出>

检查维度：B–K 全维（含 **C-orch**，多文件协调改动时必查），范围**限定于 diff**；A 对照 PRD/spec（若提供）与 diff 内变更点。

与 review-full 区别：diff 单轮、只评 diff 范围、结论**三态**（非 scope-ready/review-ready 循环）。
**流程**：本 prompt 评审后，主代理**必跑** cleanup-check；K 节 ✅ 以 cleanup-check 为准。

请用中文返回：
1）diff 范围摘要（文件数、主要模块）
2）must-fix（id, P0|P1|P2, 文件, 描述, 维度, 改法）
3）open_questions（如有）
4）spec_deviations（open/fixed/none；无 spec 则 none）
5）结论（评审子代理）：**通过** | **需修改** | **阻塞** + 一句话理由
   - 通过：P0=P1=P2=0，无 open spec_deviations（**K 节待 cleanup-check 确认**）
   - 需修改：存在未闭合 must-fix（含 P2）
   - 阻塞：存在未闭合 **P0**，或 open spec_deviations，或调查未完成（DAG `blocked`）
   - 注：diff「阻塞」≠ DAG `blocked`（后者指 fix 往返≥3 次等流程卡死）
```

**diff 最终结论**（主代理在 cleanup-check 后汇总）：

- **通过**：评审 P0=P1=P2=0 + cleanup K-ready: yes + 无 open spec_deviations
- **需修改**：评审或 cleanup 有未闭合 must-fix
- **阻塞**：未闭合 P0 或 open spec_deviations

---

## review-ready 与 Review Closure

```markdown
| 项 | 状态 |
| review-ready | yes/no |
| dag_version | N |
| P0 / P1 / P2 | 0 / 0 / 0 |
| K 节 | ✅/❌ |
| spec_deviations | none / open: [...] |
| C-orch | ✅/N/A（多文件协调改动时须 ✅） |
| C 类合并后 QA | （可不空，不阻塞） |
```

---

## 独立使用（diff）

「review 这个 PR」→ **diff** 模式：输入 PRD/spec（若有）+ **PR diff 范围**；流程为 **diff 评审 → cleanup-check（必跑）** → 主代理汇总三态。**通过**须评审与 cleanup 均达标。

---

## 执行检查清单

### 通用

- [ ] 已读 PRD + spec（或用户指定的评审范围）
- [ ] review* 已派 readonly 子代理，主代理未自审
- [ ] fix / verify / cleanup-check 均已派 Task 子代理
- [ ] 主代理未自改代码、未自跑 lint/test 替代子代理
- [ ] Context Bundle 已按规范拼装
- [ ] not-ready 已重编排 DAG（`wave_plan` 更新 + `dag_version++`）
- [ ] fix 后已重跑 review；cleanup 失败已走 fix-cleanup → verify → cleanup-check
- [ ] P2 已派 fix 闭合；must-fix 项均已认定有问题（未认定项在 open_questions）
- [ ] spec_deviations 已闭合或用户已确认
- [ ] Review Closure 表已附（review-ready 时）
- [ ] `dag_version` 已记录

### func / scope / full / diff

- [ ] **func**：P0=P1=0，未产生 P2
- [ ] **scope**：结论仅 scope-ready，未误报 review-ready；并行 scope 已合并 must_fix
- [ ] **full**：A/G 抽检条件已满足或已全查；K 以 cleanup-check 为准
- [ ] **diff**：cleanup-check 已必跑；最终三态已含 K 结果
- [ ] manual_user 未误标阻塞
- [ ] 多文件协调改动已查 **C-orch**；不适用处已标 N/A
