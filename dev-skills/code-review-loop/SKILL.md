---
name: code-review-loop
description: 对照 PRD/spec 与 diff 做代码评审 DAG，not-ready 时由子代理完善可执行 fix-spec（不改实现代码），循环直至 fix-spec-ready。含 scope/full/diff 模式。用户提及 CR loop、code review loop、产出修复 spec、评审后给执行 spec 时使用。
disable-model-invocation: true
---

# Code Review Loop

**评审 → 完善可执行 spec** skill：对照 **PRD + spec**（及 diff 范围）做质量评审，将 must-fix 收敛进一份 **fix-spec**；not-ready 时只改 fix-spec，再 CR。终点 **fix-spec-ready**（可交给 code-dev-loop / 实现任务执行）。

主代理 = 编排；子代理 = **review**（readonly）/ **spec-fix**（只改 fix-spec）。中文协作。

本 skill **不改实现代码、不跑 verify/cleanup、不宣称可合并**。

---

## 边界

| 本 skill | 不做 |
|----------|------|
| 代码评审 + 产出/完善 **fix-spec** | 改实现、跑测试/build、lint/format、宣称 merge-ready |
| 按评审维度拆 wave、循环收敛 | 业务 PRD/SPEC 首次撰写（属 generate / spec-check） |
| 独立 diff 单轮评审并落盘 fix-spec | 在本 skill 内执行 fix-spec |

与 **spec-check-loop** 对称：彼收敛「编码前文档」→ execute-ready；本 skill 收敛「评审后修复说明书」→ fix-spec-ready。

---

## 关键术语

### fix-spec

评审产出的**可执行修复说明书**（默认路径见「产出路径」）：把 must-fix 写成可派实现的步骤（文件、改法、验收/测试），供下游执行。**不是**业务迭代的完整 SPEC 重写，除非用户指定合并进既有 spec。

### fix-spec-ready

本 skill 终点。须同时满足：

- 末轮 **full**（或用户指定的等价全局评审）：已查维度内 **P0=P1=P2=0** 未写入 fix-spec 的开放项（全部已进 fix-spec 或已用户豁免）
- fix-spec 内每条 must-fix：**有改法、有触达文件、有验收/测试要点**；无「只批评、无改法」条目
- 无 open **spec_deviations**（相对业务 PRD/spec；用户确认收窄视同 fixed）
- **Fix-Spec Closure** 表已附
- `qa: manual_user` **不阻塞**（写入 C 类「合并后请你验收」，可进 fix-spec 附录）

不等于代码已修完或可合并；本 skill 只管「修复说明书可执行」。

### scope-ready

**scope** 模式中间结论：本 scope 已查维度 **P0=P1=P2=0** 均已落入 fix-spec（或本 scope 无发现）；无本 scope 相关 open **spec_deviations**。不含 D/E/F/H–J/K 全局结论——由 **review-full** 负责。

**scope 检查范围**：A（本 scope）+ B + C + **C-orch**（本 scope 文件）+ G（本 scope 测试）。

### 审查轮（review round）

一次完整的：**readonly 评审 → 主代理汇总 →（若未 ready）派遣 spec-fix 并同步等待 → 下一轮**。

### 编排收敛（C-orch）

对**同一条业务事务**，减少平行路径、多余转手、分散协调点；**不改变**对外可观察行为。

| 类型 | 说明 | 评审态度 |
|------|------|----------|
| **必要 hop** | 平台边界、序列化 DTO、性能合批、清晰分层 | 保留；须注释或 spec 写明 |
| **偶然 hop** | 平行装配入口、无事务语义透传、同语义多信号、死通道 | **收敛目标** → 写入 fix-spec |

与 **DRY**：DRY 反知识重复；C-orch 反路径重复。不适用（纯样式/单函数）标 **N/A**。

---

## 主代理禁止

- **自审**：review* 须 readonly 子代理
- **自改 fix-spec**：spec-fix 须 Task 子代理；主代理不得直接编辑闭合 must-fix
- **改实现 / 跑门禁**：不得在本 skill 内改代码、跑 verify/lint 代替下游执行
- **not-ready 时口头收尾**：不得只描述改法而不派 spec-fix；须更新 `spec_fix_plan` 并 `dag_version++`

---

## 评审模式

| 模式 | 典型场景 | 结论 |
|------|----------|------|
| **scope** | 分模块评审 DAG | scope-ready / not |
| **full** | 末轮/global | 贡献 fix-spec-ready / not |
| **diff** | 独立 PR，可单轮 | 通过（无修复项）/ 需产出 fix-spec / 阻塞 |

review* **须 readonly**。波次内功能小检（func）属 **code-dev-loop**，本 skill 不覆盖。

---

## 子代理派遣规范

| 节点 | 工具 | subagent_type | readonly | 说明 |
|------|------|---------------|----------|------|
| **review** / **review-scope** / **review-full** | Task | generalPurpose | **true** | 非自审 |
| **spec-fix** | Task | generalPurpose | false | **只改 fix-spec**（及用户指定的文档路径） |

- 同 wave 无冲突可并行；**同步等待** 当前 wave 全部返回后再汇总
- **同文件禁止** 并行 spec-fix
- **失败**：重试一次 → 仍失败则主代理可手工等效 spec-fix，标注「手工 spec-fix」

---

## 动态 DAG

not-ready 后 **改图**，`dag_version++`：

| 动作 | 示例 |
|------|------|
| 并行 review-scope | 不同模块同 wave |
| 合并 review | 多模块同类 → 一次 review-scope |
| 并行 / 合并 spec-fix | 无同文件冲突的章节同 wave；同章合并 |
| 优先 P0 | 阻塞条目进下一 wave 首部 |
| 插入 review-full | 各 scope-ready 且 fix-spec 草案齐后 |

```yaml
dag_version: 1
review_round: 2
wave_plan: [[review-scope-a, review-scope-b], [spec-fix-p0], [review-full]]
node_status: { review-scope-a: done, spec-fix-p0: pending }
must_fix: []   # open 项；须含 id, P0|P1|P2, 文件, 维度, 改法, 来源 node_id
fix_spec_path: .apm/kb/docs/Iterations/<name>/cr-fix-spec.md
spec_fix_plan: [[fix-spec-§P0], [fix-spec-§P1]]
status: 待下轮审查  # 待首轮审查 | 待下轮审查 | 待用户确认 | fix-spec-ready 已确认
```

**禁止**：not-ready 后只改一处不更新 `spec_fix_plan` / `dag_version`；宣称 ready 却未落盘 fix-spec（有 must-fix 时）。

无 APM 时：用用户指定路径或对话内约定路径等价维护。

---

## 节点类型与典型子图

```text
review-scope-*（并行）
  → spec-fix-*（把 must-fix 写入/完善 fix-spec）
  → review-scope-* 或 review-full（校验 fix-spec 覆盖与可执行性）
  → … 循环 …
  → fix-spec-ready（结论态，非波次节点）
```

末轮全局节点为 **review-full**；**fix-spec-ready** 为终点结论，不单独派遣子代理。

---

## 检查维度

### full / 贡献 fix-spec-ready（B–K；scope 见上）

**A.** 需求符合性 — 变更点、步骤矩阵、测试矩阵 T-xx、PRD 验收、范围蔓延

**B.** 正确性 — 边界、错误处理、资源释放、类型契约

**C.** 质量 — 风格、**DRY**、命名、复杂度、职责、死代码

| 检查项 | 严重度参考 |
|--------|------------|
| 死代码、明显 DRY、职责越界、与惯例冲突的复杂度 | **P1** |
| 命名不顺、风格不一致、可读性小改、局部小 refactor（**改法明确**） | **P2** |
| 已认定有问题且改法明确 | **P1/P2** → must-fix → 写入 fix-spec |
| 尚未认定，或是否在 scope 内不明 | **不得**进 must-fix → `open_questions` / `blocked` |

**C 维**：写入 must-fix = 已认定相对 PRD/spec/惯例**有问题**；须 P1/P2 + 改法。禁止「只批评、不认定」。

**C-orch.** 编排收敛（多文件/跨端协调时适用，否则 **N/A**）

| 检查项 | 严重度 |
|--------|--------|
| 平行入口且无 parity 测试 | P1；升 P0：红测证明分叉或双写无 parity |
| 无意中间态（仅透传的 ref/Map/队列） | P1 |
| 信号混用无映射 | P1；升 P0：双减/竞态/重复触发 |
| 假扁平化破坏平台/事务/spec 阶段 | **P0** |
| 收敛无证据（无测/T-xx/验收覆盖） | P1 |

与 **A**：spec 写明单点装配等未满足 → A + C-orch 同记。与 **G**：parity/集成测为门禁证据（写入 fix-spec 验收栏）。

**D.** 安全 — 注入、鉴权、密钥、校验（适用时）

**E.** 性能 — 明显 N+1、热点 IO（适用时）

**F.** 中文注释 — 关键逻辑中文注释；公开 API 文档注释；英文说明→P0

**G.** 测试 — 行为/边界/失败路径；缺口写入 fix-spec 验收，**本 skill 不跑测**

**H–J.** 兼容性、可观测性、UI（适用则查，否则未涉及）

**K.** 收尾建议 — 调试残留、杂项改动、lint/format、文档同步：**作为 fix-spec 步骤写入**，由下游执行时闭合；本 skill **不**自跑 lint/format

**manual_user**：不阻塞 → C 类附录

### spec_deviations

open → 不得 scope-ready / fix-spec-ready。

**流转**：review 标 `open` → 用户决策或写入 fix-spec「按现状收窄」经用户确认 → `fixed` → 下轮 review 验证。

---

## 严重度

| 级别 | 含义 | 进 fix-spec |
|------|------|-------------|
| **P0** | 阻塞：正确性、安全、spec 违背、假扁平化等 | 必须 |
| **P1** | 结构性质量：死代码、DRY、C-orch、职责越界等 | 必须 |
| **P2** | 可明确优化（风格、命名、小 refactor）；**不是**待商榷 | 必须 |

- **must-fix = 已认定有问题**；须标级别 + 改法，**不得标 blocked**
- P2 须写清改法；写不出 → 未认定，不得进 must-fix

### must-fix vs open_questions / blocked

| 输出 | 含义 | 处置 |
|------|------|------|
| **must-fix** | 已认定 + 有改法 | 写入 fix-spec，下游必须执行 |
| **open_questions** | 未认定 / scope 外 | 列给用户；可进 fix-spec「待拍板」附录，不阻塞 ready（除非用户要求） |
| **blocked** | 调查未完成、spec 空白、同一项震荡 ≥3 次 | 暂停，请用户 |

**简记**：**认定有问题 → must-fix → fix-spec；没认定 → 别进 must-fix。**

| 场景 | 处理 |
|------|------|
| 命名丑、风格不统一 | P2 + 改法 → fix-spec |
| 违背项目惯例 | P2 → fix-spec |
| 两可且无惯例、可接受 | 不进 must-fix；可 open_questions |
| bug / 违背 spec | P0/P1 → fix-spec |
| 改行为但 spec 未规定 | spec_deviations 或 open_questions；勿假装 P2 |

---

## 总流程

```text
准备 → 读 PRD/spec/diff → 初始评审 DAG → 确定 fix_spec_path
  → loop:
      取就绪 wave → 并行派子代理 → 同步等待 → 更新 node_status
      → fix-spec-ready? ─是→ Fix-Spec Closure → 请用户确认 → 结束
      → not-ready? ─→ 拆 spec-fix wave（dag_version++）→ 继续
```

**轮次上限**：默认 **5**；仍 not-ready 时汇报未闭合项，请用户拍板。

同一 must-fix 震荡 ≥3 次 → **blocked**。

**diff**：单轮 readonly 评审 →（有 must-fix 则）派 spec-fix 落盘 → 汇总三态；可不进入多轮 scope DAG。

---

## Step 1：初始拆解

- 按模块/检查域拆 review-scope；末轮预置 review-full
- 确定 `fix_spec_path`（见「产出路径」）；首轮前可建空壳或由首次 spec-fix 创建
- 小 diff + 用户明示 → **diff** 单轮 + 一次 spec-fix

---

## Step 2：波次执行（审查）

1. 当前 wave **全部**并行派 Task
2. **同步等待** 后汇总
3. 并行 scope：合并 `must_fix`（id：`{scope-id}/{维度}-{序号}`；同 id 去重；同文件同维保留最高严重度）
4. 任一 fail / not ready → **not-ready** → Step 3

审查子代理须：通读相关代码与 PRD/spec；若已有 fix-spec，对照是否**覆盖且可执行**；上轮 must-fix 标 已写入 / 仍缺 / 新问题。

审查子代理禁止：改代码或 fix-spec；宣布 fix-spec-ready（只给建议，主代理收敛）。

---

## Step 3：spec-fix（仅 not-ready）

主代理按 `spec_fix_plan` 派 **spec-fix** 子代理，**只改 fix-spec**（及用户明确允许的文档），把本轮 must-fix 写成可执行条目。

完成后：`spec_fix_plan` 置空、状态「待下轮审查」、`review_round++`，回到 Step 2。

---

## Step 4：终止与用户确认

### 自动终止（fix-spec-ready）

通知用户：轮次、P0/P1/P2 条数、`fix_spec_path`、Fix-Spec Closure；**请确认是否按该 fix-spec 开工执行**。

用户确认前：**不开始改代码**（除非用户显式接受风险或另开实现任务）。

### 无修复项

末轮评审 P0=P1=P2=0 且无 open spec_deviations：fix-spec 可为空壳或注明「无必须修复项」；仍附 Closure，**不**宣称代码可合并。

### 用户中途指令

- 「某 P2 不修」→ 移出 must-fix，写入 fix-spec「已豁免」+ 用户原话
- 「停止循环」→ 汇报当前缺口与已有 fix-spec 后结束
- 「直接修代码」→ 提示本 skill 终点是 fix-spec；确认后可另开 code-dev-loop / 实现任务

---

## 产出路径

默认（有 APM / 迭代结构时）：

```text
.apm/kb/docs/Iterations/<需求名称>/cr-fix-spec.md
```

用户另指定则从其指定。落盘后若 APM 可用：`apm kb index rebuild`。

### fix-spec 文档结构（须遵守）

```markdown
# CR Fix Spec: <简短名称>

## 元信息
- repo / base_sha / head_sha（或 PR）
- prd_path / spec_path
- review_round / dag_version
- 状态：draft | fix-spec-ready

## Must-fix（按 P0 → P1 → P2）

### <id> [P0|P1|P2] <标题>
- 维度：A|B|C|C-orch|...
- 文件：
- 问题：
- 改法：（具体到可执行）
- 验收/测试：
- 来源：review node / round

## Spec deviations
- open / fixed / none

## Open questions / 待拍板
## 已豁免（用户确认不修）
## 合并后 QA（manual_user）
## K 节建议（下游执行时闭合）
```

---

## Context Bundle

```yaml
repo_path: <绝对路径>
base_sha: <sha>
head_sha: <sha>
mode: scope | full | diff
node_id: <review-scope-a | review-full | ...>
files: [<本节点范围文件>]
spec_path: <业务 spec 或「未提供」>
prd_path: <或「未提供」>
fix_spec_path: <路径>
prior_conclusion: <上轮结论，首轮 none>
must_fix: []
```

spec-fix 用 **delta**：本轮新增/仍开放 must-fix、目标章节、`fix_spec_path`。

---

## Prompt 模板

### scope

```text
【语言要求】全程中文

readonly 评审。skill：code-review-loop。模式：scope
仓库：<REPO_PATH>
业务 Spec / PRD：<路径>
fix-spec：<FIX_SPEC_PATH>（无则写「尚未创建」）
BASE_SHA / HEAD_SHA：<sha>
节点 id：<review-scope-x>
检查维度：A + B + C + C-orch + G（本 scope）；不含 D/E/F/H–J/K
C-orch：多文件协调必查；否则 N/A + 理由
Context Bundle：<YAML>

请用中文返回：
1）相对上轮：已写入 fix-spec / 仍缺 / 新问题（第 2 轮起）
2）must-fix（id, P0|P1|P2, 文件, 描述, 维度, 改法）— 供写入 fix-spec
3）open_questions — 未认定；不得混入 must-fix
4）spec_deviations
5）fix-spec 可执行性（若已有）：缺口 / 歧义 / 缺验收
6）结论：scope-ready: yes|no（禁止输出 fix-spec-ready）
```

### full

```text
【语言要求】全程中文

readonly 评审。skill：code-review-loop。模式：full
仓库：<REPO_PATH>
业务 Spec / PRD：<路径>
fix-spec：<FIX_SPEC_PATH>
BASE_SHA / HEAD_SHA：<sha>
节点 id：review-full
检查维度：B–K 全维（含 C-orch）；A/G 按 diff/范围全查或说明抽检依据
Context Bundle：<YAML>

请用中文返回：
1）相对上轮：已写入 / 仍缺 / 新问题
2）must-fix（含改法，供 fix-spec）
3）open_questions
4）spec_deviations
5）fix-spec 覆盖度与可执行性（每条是否有文件+改法+验收）
6）未涉及维度：H/I/J=未涉及（适用则查）
7）结论：建议 fix-spec-ready: yes|no + 理由（由主代理最终判定）
```

### spec-fix

```text
【语言要求】全程中文

节点：spec-fix-<id>。只改文档，不改实现代码。
仓库：<PATH>
fix-spec 路径：<FIX_SPEC_PATH>（不存在则创建，结构见 skill「fix-spec 文档结构」）
业务 Spec / PRD：<路径>（只读参考；除非用户允许，勿改业务 spec）
本 wave 范围：<章节/严重度>

must-fix（须写入或修订进 fix-spec）：
- <条目>

约束：
- 只编辑 fix-spec（及用户明确允许的文档）
- 每条含：id、严重度、维度、文件、问题、改法、验收/测试、来源
- P0 必须写入；P1/P2 全部写入除非用户已豁免
- 同步元信息中的 round / sha；状态保持 draft 直至主代理宣布 ready

请用中文返回：
1）修改的文件与章节
2）各 must-fix：已写入 / 仍开放 / 需拍板
3）阻塞项（如有）
```

### diff

```text
【语言要求】全程中文

readonly 评审。skill：code-review-loop。模式：diff（独立 PR，偏单轮）
仓库：<REPO_PATH>
业务 Spec / PRD：<路径或「未提供」>
Diff：BASE/HEAD 或 PR #<n>；文件：<FILES>
检查：B–K（含 C-orch）；A 对照 PRD/spec（若有）与 diff 变更点
K 节：只给出应写入 fix-spec 的收尾步骤，不跑 lint

请用中文返回：
1）diff 摘要
2）must-fix（含改法）
3）open_questions / spec_deviations
4）结论：**通过** | **需产出 fix-spec** | **阻塞**
   - 通过：P0=P1=P2=0，无 open spec_deviations
   - 需产出 fix-spec：存在 must-fix（含 P2）
   - 阻塞：未闭合 P0，或 open spec_deviations，或调查未完成
```

**diff 收尾**（主代理）：需产出 / 阻塞且用户未叫停 → 派 **spec-fix** 落盘；通过则 Closure 注明无修复项。

---

## Fix-Spec Closure

```markdown
| 项 | 状态 |
| fix-spec-ready | yes/no |
| fix_spec_path | ... |
| dag_version / review_round | N / N |
| P0 / P1 / P2（已写入 fix-spec） | n / n / n |
| 未写入的开放 must-fix | 0 |
| spec_deviations | none / open: [...] |
| C-orch | ✅/N/A |
| C 类合并后 QA | （可不空，不阻塞） |
```

---

## 执行检查清单

- [ ] 已读 PRD + 业务 spec + diff 范围；已定 `fix_spec_path`
- [ ] review* 已派 readonly 子代理；主代理未自审
- [ ] not-ready 已派 spec-fix（非主代理直改；未改实现代码）
- [ ] Context Bundle 已拼装；`dag_version` / `spec_fix_plan` 已更新
- [ ] must-fix 均有改法并已进 fix-spec（或用户豁免）
- [ ] spec_deviations 已闭合或用户确认
- [ ] 未在本 skill 内跑 verify/cleanup 或宣称可合并
- [ ] Fix-Spec Closure 已附；用户确认前未开工改代码
- [ ] 多文件协调已查 C-orch；不适用已标 N/A
- [ ] manual_user 未误标阻塞
