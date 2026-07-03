---
name: code-review-loop
description: 基于 PRD/spec 的评审 DAG：review / fix / verify / cleanup-check 波次并行，not-ready 时动态重编排，直至 review-ready。含 func 模式用于波次内功能小检。可独立 diff 评审。
disable-model-invocation: true
---

# Code Review Loop

**质量评审** skill：对照 **PRD + spec**（及用户提供的 diff 范围），用 **动态 DAG** 循环至 **review-ready**。可独立用于 PR 评审，也可在用户指定的代码变更范围内运行。

主代理 = 编排；review = readonly 子代理；fix / verify / cleanup = 执行子代理。中文协作。

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

- 末轮 **full** 评审：P0=P1=0，K 节 ✅
- **spec_deviations** 无 open
- **Review Closure** 表已附
- spec 中 `qa: manual_user` **不阻塞**（写入 C 类「合并后请你验收」）

### func-ready

**func** 模式结论，用于**波次内功能小检**。须：本波次 spec 步骤矩阵 ✅、blocking 测试 id ✅、verify 证据 ✅、无 open spec_deviations。

### func vs full（增量策略）

- **func**：只查 **A（需求符合性-本波次范围）+ G（测试/verify）+ spec_deviations**
- **full**：查 **B–K 全维**；若上下文表明相关模块已通过 func 小检且 HEAD 未超出该范围，**A/G 可抽检**，重点查 B–K

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
| **fix** | Task | generalPurpose | false | 闭合 must-fix |
| **verify** | Task | generalPurpose 或 shell | false | 测试/build |
| **cleanup-check** | Task | generalPurpose 或 shell | false | K 节 lint/format/清调试 |

- 同 wave 无冲突可并行；**同步等待**
- 失败：重试一次 → not-ready 重编排或 blocked

---

## 动态 DAG

not-ready 后 **改图**，`dag_version++`：

| 动作 | 示例 |
|------|------|
| 并行 fix | 不同文件的 P1 → 同 wave |
| 合并 review | 多模块同类 → review-scope 一次 |
| 插入 verify | fix wave 之后 |
| cleanup-check | verify 之后、review-full 之前 |

```yaml
dag_version: 1
wave_plan: [[review-scope-a, review-scope-b], [fix-x, fix-y], [verify-all], [cleanup-check], [review-full]]  # → review-ready（终点结论，非 wave 节点）
node_status: { review-scope-a: done, fix-x: pending }
open_must_fix: []
```

**禁止**：fix 后不重跑 review；not-ready 只串行一个 fix 而不改 wave_plan。

---

## 节点类型与典型子图

| 类型 | 执行者 |
|------|--------|
| **review** / **review-scope** / **review-full** | readonly |
| **fix** | inline 子代理 |
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

**scope** 检查维度：A（本 scope）+ B/C（本 scope 文件）+ G（本 scope 测试）；不含全局 DRY/K（留 full）。

---

## 检查维度

### func

- **A** 本波次 spec 步骤矩阵、blocking 测试 id
- **G** verify 证据
- 无 open **spec_deviations**

### full / review-ready（B–K；A/G 见增量策略）

**A.** 需求符合性 — 变更点、步骤矩阵、测试矩阵 T-xx、PRD 验收、范围蔓延

**B.** 正确性 — 边界、错误处理、资源释放、类型契约

**C.** 质量 — 风格、**DRY**、命名、复杂度、职责、死代码

**D.** 安全 — 注入、鉴权、密钥、输入校验（适用时）

**E.** 性能 — 明显 N+1、热点 IO（适用时）

**F.** 中文注释 — 关键逻辑中文注释；公开 API 文档注释；英文说明→P0

**G.** 测试 — 行为测试、边界、失败路径；verify 已通过

**H–J.** 兼容性、可观测性、UI（适用则查，否则标注未涉及）

**K.** 收尾 — 无调试残留、无杂项改动、lint/format、工作区干净、spec 文档已更

**manual_user**：**不阻塞** review-ready → C 类「合并后请你验收」

### spec_deviations

open → 不得 func-ready / review-ready；fixed 或用户确认收窄 spec。

---

## 严重度

P0/P1 必须 0 才 ready；P2 可列建议。

---

## 总流程

```text
准备 → 读 PRD/spec/变更范围 → 初始评审 DAG
  → loop: wave 并行 → 同步等待 → 汇总
      → review-ready? ─是→ Review Closure 表 → 结束
      → not-ready? ─→ 重编排 dag_version++ → 继续
```

**diff**：单轮 **diff** 评审（见 diff prompt，非 review-full 循环）+ cleanup-check（可选），无 loop；结论三态；需改则用户决定是否进入 fix loop。

---

## Step 1：初始拆解

- 按模块/检查域拆 review-scope 或 review-full
- 预置 fix/verify/cleanup 占位
- 小 diff + 用户明示 → 可 diff 单轮或精简为 review-full + cleanup-check

---

## Step 2：波次执行

- review*：readonly + 本 skill + 模式 + PRD/spec 路径
- fix：闭合 must-fix（须引用来源 review 节点 id / must-fix id）
- verify：相关测试/build
- cleanup-check：K 节

---

## Step 3：重编排

收集 must-fix → 并行 fix → 合并 review → dag_version++

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

请用中文返回：
1）步骤/测试矩阵对照（pass/fail + 证据）
2）must-fix 列表（id, P0|P1, 文件, 描述）
3）spec_deviations（open/fixed/none）
4）结论：func-ready: yes|no + 一句话理由
```

### scope / full

```text
【语言要求】全程中文

readonly 评审。skill：code-review-loop。模式：<scope|full>
仓库：<REPO_PATH>
Spec / PRD：<路径>
BASE_SHA / HEAD_SHA：<sha>
节点 id：<review-scope-x | review-full>
检查维度：<scope: A+B+C+G | full: B–K，A/G 增量见 skill>

请用中文返回：
1）相对上轮：已修复 / 仍开放（第 2 轮起）
2）must-fix（id, P0|P1|P2, 文件, 描述, 维度）
3）spec_deviations
4）结论：scope-ready|review-ready: yes|no + 理由
```

### fix

```text
【语言要求】全程中文

节点：fix-<id>。来源 review：<review_node_id>
仓库：<PATH>。Bundle-delta：<YAML>
must-fix：<列表>

inline 闭合 must-fix；跑相关测试；中文 commit。

返回：1）已闭合项 2）提交 sha+message 3）验证结果 4）未闭合项
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

检查维度：B–K 全维，范围**限定于 diff**；A 对照 PRD/spec（若提供）与 diff 内变更点。

与 review-full 区别：diff 单轮、只评 diff 范围、结论**三态**（非 scope-ready/review-ready 循环）。

请用中文返回：
1）diff 范围摘要（文件数、主要模块）
2）must-fix（id, P0|P1|P2, 文件, 描述, 维度）
3）spec_deviations（open/fixed/none；无 spec 则 none）
4）结论：**通过** | **需修改** | **阻塞** + 一句话理由
   - 通过：P0=P1=0，无 open spec_deviations，K 节 ✅
   - 需修改：存在 P1 或可自动修复项
   - 阻塞：存在 P0 或需用户决策的 open spec_deviations
```

---

## review-ready 与 Review Closure

```markdown
| 项 | 状态 |
| review-ready | yes/no |
| dag_version | N |
| P0 / P1 | 0 / 0 |
| K 节 | ✅/❌ |
| spec_deviations | none / open: [...] |
| C 类合并后 QA | （可不空，不阻塞） |
```

---

## 独立使用（diff）

「review 这个 PR」→ **diff** 模式（见 diff prompt）：输入 PRD/spec（若有）+ **PR diff 范围**（BASE/HEAD 或 compare URL）；单轮、非 review-full 循环；输出 **通过 / 需修改 / 阻塞** 三态。

---

## 执行检查清单

- [ ] 已读 PRD + spec（或用户指定的评审范围）
- [ ] review readonly，非自审
- [ ] not-ready 已重编排 DAG
- [ ] fix 后已重跑 review
- [ ] manual_user 未误标阻塞
