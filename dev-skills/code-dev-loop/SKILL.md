---
name: code-dev-loop
description: 基于 spec 的开发 DAG：impl / verify / cr-func 可波次并行；not-ready 时动态重编排。默认子代理执行。终点 dev-ready。
disable-model-invocation: true
---

# Code Dev Loop

**开发编排** skill：在 **spec 为唯一事实来源** 的前提下，拆解 **开发 DAG**（impl / verify / cr-func / fix），波次并行、not-ready 时动态重编排。终点 **dev-ready**（spec 范围内实现与功能小检完成）。

主代理 = 编排；子代理 = impl / verify / cr-func / fix。中文协作。

---

## 边界

| 本 skill | 不做 |
|----------|------|
| impl、verify、**cr-func**（波次内功能小检） | 全维代码评审、风格/DRY 全局 cleanup |
| 动态开发 DAG | merge-ready 宣称 |

---

## 关键术语

### dev-ready

开发 DAG 终点。须同时满足：

- 所有 impl / verify / cr-func 节点 `done`
- 各 cr-func 结论为 **func-ready**（见 §func-ready / §cr-func 检查项）
- spec 中 `blocking: yes` 的步骤已闭合；无 open **spec_deviations**
- HEAD 已提交或写入 Bundle-full

### func-ready

**cr-func** 节点结论。须：本波次 spec 步骤矩阵 ✅、blocking 测试 id ✅、verify 证据 ✅、无 open spec_deviations。以子代理返回的 **func-ready: yes/no** 为准（检查维度见 §cr-func 检查项）。

### 上游探索 ≠ 跳过 impl

无论探索结论来自何处，**都须真正落地 impl**（写代码），不能只探索不实现；至于是派子代理还是按 §轻量直改由主代理直接执行，看节点是否满足直改条件。探索摘要仅用于 Context Bundle，使 prompt 更清晰。

---

## 子代理派遣规范

| 节点 | 工具 | subagent_type | readonly | 并行 |
|------|------|---------------|----------|------|
| **impl** | Task | generalPurpose | false | 无同文件冲突可同 wave |
| **fix** | Task | generalPurpose | false | 不同文件可同 wave |
| **verify** | Task | generalPurpose 或 shell | false | 可同 wave |
| **cr-func** | Task | generalPurpose | **true** | 可同 wave |

- **同步等待** 当前 wave 全部返回后再汇总
- **失败**：重试一次 → 仍失败则 not-ready，主代理改 DAG 或标注 blocked
- **同文件禁止** 并行 impl/fix

### 轻量直改（主代理直接执行）

派遣子代理的核心价值是**上下文隔离**和**并行**。节点小而明确时两个价值都不沾，没必要付子代理往返的固定开销，主代理直接干掉。未标 executor 则默认 `executor: subagent`。

**适用节点**：impl、fix、verify。

**快路径**（命中即直改，不必细想）：改动集中在 **1–2 个文件**、位置已知（探索或 spec 已指明，无需再读一堆文件搞清现状）、验证输出短（单测 pass/fail 一行就够，而非完整构建日志）。

**主判据**是 **主代理直接执行不会消耗大量上下文**——快路径只是它的粗代理，拿不准时回到主判据衡量。特别注意**试错预期**：测试可能反复红、要边跑边改多轮的节点，即使文件少也应派遣，因为上下文膨胀发生在试错环节，不在编辑环节。

满足时在 `node_status` 标 `executor: main`，验证与证据留痕照常——省的是子代理往返，不是省质量。

**不适用**：cr-func（审查独立性须保留，且审查天生要读多个文件对照）。

---

## 动态 DAG

每轮 wave 结束或 **not-ready** 后，主代理 **改图** 并 `dag_version++`。

| 动作 | 何时 |
|------|------|
| **并行化** | 无文件冲突的 impl/fix 同 wave |
| **合并** | 多 fix 同一模块 → 单 fix 节点 |
| **拆分** | 节点过大或反复 fail → 串行子节点 |
| **插入 cr-func** | impl 波次后、下一模块前（**须在 DAG 内**） |
| **优先 P0** | 阻塞下游的 fix 进下一 wave 首部 |

```yaml
dag_version: 1
wave_plan: [[impl-a, impl-b], [verify-a, verify-b], [cr-func-ab]]
node_status:
  impl-a: { status: done, head_sha: abc123, executor: subagent }
  impl-b: { status: done, executor: main }   # 轻量直改
  cr-func-ab: { status: done, func_ready: yes }
open_must_fix: []
spec_deviations: []  # open 项阻塞 dev-ready
```

上表落在 Context Bundle / iteration-state，**不是**记忆文件正文。

**禁止**：not-ready 时只「再派一个 fix」而不更新 `wave_plan` / `dag_version`。

---

## 节点类型

| 类型 | 执行者 | 说明 |
|------|--------|------|
| **impl** | 子代理（轻量时主代理直改，§轻量直改） | 强耦合链 → **单 impl 节点**；无冲突可多 impl **同 wave** |
| **verify** | 子代理（轻量时主代理直改） | 依赖上游 impl/fix；同 wave 可并行 |
| **cr-func** | readonly 子代理 | 波次内 **功能小检**（§cr-func 检查项）；波次即其检查范围，每波次一个 |
| **fix** | 子代理（轻量时主代理直改） | 闭合 verify/cr-func 的 must-fix |

典型子图：

```text
wave-0: [impl-core]
wave-1: [verify-core]
wave-2: [cr-func-core] ─not ready─→ wave-3: [fix-core] → wave-4: [verify-core, cr-func-core]
wave-5: [impl-ui, impl-bridge]
wave-6: [verify-ui, verify-bridge]
wave-7: [cr-func-ui-bridge]
```

**fix 后**：须重跑 **verify + cr-func**（作为新 wave 写入 DAG，不可跳过）。

---

## Context Bundle

DAG 与编排状态（`dag_version`、`wave_plan`、`node_status`、open must-fix 等）写在 **Context Bundle** 或 `docs/.iteration-state.yaml`（或对话内 YAML），**禁止**写入记忆文件（memory 目录与 RULE.md）。记忆写法见 `apm-record`「记忆语义」。

### full（首轮 / dev-ready 产出）

```yaml
spec_path: ...
prd_path: ...
spec_summary: ...
explore_summary: ...    # 若上下文中有
files_touched_plan: [...]
contracts: [...]        # API/事件/配置
evidence_packs: [...]   # 稳定背景证据包路径（见「证据包」）
head_sha: ...
blocking_steps: [...]
dag_version: 1
wave_plan: [[impl-a, impl-b], [verify-a], [cr-func-ab]]
node_status:
  impl-a: { status: done, head_sha: abc123, executor: subagent }
open_must_fix: []
spec_deviations: []
```

### delta（fix / 后续 wave）

```yaml
must_fix: [{ id, severity, file, desc }]
files_changed: [...]
prev_cr_func: { node_id, func_ready, open_items }
head_sha: ...
dag_version: ...
wave_plan: [...]
node_status: { ... }
```

### 证据包（evidence pack）

跨波次重复读的**稳定背景**（模块地图、调用链、约定、既往 findings、diff 统计）落盘复用，**生产者写、消费者读**：

- 生产者：第一个读该区域的子代理（上游探索 / 首轮 review / 首个 impl）顺手落盘，路径写入 Bundle 的 `evidence_packs`；主代理只持路径，**不亲自写包**——否则读代码的成本只是转移到了最贵的主代理上下文上
- 路径：`docs/iterations/<name>/cache/<topic>.md`（过程产物，不入版本控制；用户另指定则从其指定）
- 失效：包头记 `head_sha` / 生成节点 / 范围；后续节点发现 sha 不一致 → 视为 stale，仅作线索，事实须新鲜读
- 边界：只放稳定背景；**spec 与被改代码必须新鲜读**，不得以包替代

---

## 开始前

- 工作区干净；非 main/master
- 用户已确认 spec；已知 Spec / PRD path（记忆文件或 Bundle 中可沿用）
- 按 `apm-recall`「快速开始」读规则与最近记忆摘要
- 读 spec Step：`phase-*`、`blocking: yes/no`
- **记忆**：按 `apm-record`「记忆语义」记录进展：同主题记忆文件追加对话轮次并刷新 `date`/`abstract`；仅当出现新的跨会话规则时才更新 `RULE.md`

---

## cr-func 检查项

- **A** 本波次 spec 步骤矩阵、blocking 测试 id
- **G** verify 证据
- 无 open **spec_deviations**

---

## 总流程

```text
准备 → Bundle-full → 初始 DAG（含 cr-func 边）
  → loop:
      取就绪 wave → 轻量节点主代理直接执行，余者并行派子代理 → 同步等待 → 更新 node_status
      → 全节点 done 且 cr-func 均 func-ready? ─是→ dev-ready
      → not-ready? ─→ 重编排（dag_version++）→ 继续
```

同一 must-fix 震荡 ≥3 次 → **blocked**，请用户拍板。

---

## Step 1：初始拆解

1. 按 spec 步骤/模块拆 **impl + verify + cr-func**（依赖边：impl → verify → cr-func）
2. 拓扑排序得 **wave-0…n**；`blocking: yes` 步骤须落在对应 cr-func 范围
3. 将节点表、边、`wave_plan`、`dag_version: 1` 写入 **Context Bundle / iteration-state**（勿写记忆文件）
4. 按 `apm-record`「记忆语义」记录进展（如「已拆 DAG，进入 wave-0」）：同主题记忆文件追加对话轮次并刷新 `date`/`abstract`；无新跨会话规则则不动 `RULE.md`

---

## Step 2：波次执行

1. 当前 wave 节点分类：轻量者主代理直接执行（标 `executor: main`，判据见 §轻量直改），其余 **并行**派子代理
2. **同步等待**
3. 任一 fail / cr-func not func-ready → **not-ready**

### cr-func prompt

```text
【语言要求】全程中文

readonly 功能小检。节点：<node_id>
仓库：<REPO_PATH>
Spec：<SPEC_PATH>
PRD：<PRD_PATH>
本波次范围文件：<FILES>
verify 摘要：<VERIFY_SUMMARY>
Context Bundle：<YAML>

检查：A（本波次步骤矩阵+blocking 测试 id）、G（verify 证据）、spec_deviations

返回：1）矩阵对照 2）must-fix 3）spec_deviations 4）func-ready: yes|no + 理由
```

### impl / fix / verify prompt 模板

```text
【语言要求】
- 全程使用中文：任务说明、过程、结论、返回报告
- 代码注释与公开 API 文档注释须中文；commit message 中文
- 标识符、协议字段、第三方 API 名保持英文

节点：<node_id>，类型：<impl | fix | verify>
仓库/worktree：<PATH>
分支：<BRANCH>
Spec / PRD：<路径>
Context Bundle：
<粘贴 full 或 delta YAML>

任务：<本节点具体目标，含 spec 步骤 id>
约束：
- impl/fix：单次 inline 连续执行；按逻辑块提交；不 revert 无关改动
- verify：运行与改动相关的测试/build；记录命令与结果

请用中文按下列结构返回：
1）已完成项与剩余缺口
2）提交记录（sha + message，无则写「无提交」）
3）验证命令及通过/失败
4）阻塞项（如有）
```

---

## Step 3：动态重编排（not-ready）

1. 收集 open must-fix、failed 节点
2. 按「动态 DAG」表改图：新增 fix、调整 wave、必要时重插 cr-func
3. `dag_version++`，写回 Bundle / iteration-state → 回到 Step 2
4. 按 `apm-record`「记忆语义」用一两句人话记录现状（卡点 + 下一步）；勿把新 DAG 全文贴进记忆

---

## dev-ready

### 检查清单

- [ ] DAG 内 impl/verify/cr-func 均 `done`
- [ ] 各 cr-func **func-ready: yes**
- [ ] blocking spec 步骤闭合；spec_deviations 无 open
- [ ] HEAD 写入 Bundle-full

**禁止**在未完成上述项时宣称 merge-ready。

### 完成产出（Bundle-full，供用户或后续任务取用）

```yaml
spec_path: ...
prd_path: ...
branch: ...
base_sha: ...
head_sha: ...
bundle_full: { ... }
dev_dag_matrix: { node_id: done, ... }
spec_deviations: []
```

---

## 对用户结束语

| 状态 | 含义 |
|------|------|
| **dev-ready** | 开发 DAG 完成，spec 范围内实现已闭合 |
| **blocked** | 需用户决策 |
| **in-progress** | 汇报 wave / dag_version，不称完成 |

---

## 执行检查清单

- [ ] 用户已确认 spec（记忆文件或 Bundle 可沿用）
- [ ] 初始 DAG **含 cr-func 节点与边**，且状态在 Bundle / iteration-state（非记忆文件）
- [ ] 无冲突节点已尝试 **wave 并行**
- [ ] 轻量节点已由主代理直接执行并标 `executor: main`（验证仍照常执行）
- [ ] not-ready 已 **重编排**（dag_version 递增）
- [ ] fix 后已重跑 verify + cr-func
- [ ] 阶段推进已按 `apm-record` 记忆语义更新记忆
- [ ] dev-ready 时未自称 merge-ready

---

## 备注

- 偏离 spec：dev-ready 前须 fix 或用户确认收窄并记入 spec_deviations
- 同文件禁止并行 impl
