---
name: spec-generate
description: 输出技术规格（SPEC），整合技术设计与实现计划。先派遣多个子代理探索代码并汇总探索报告，再撰写 spec。用户提及写 spec、技术方案、实现计划时使用。
disable-model-invocation: true
---

# Spec 生成

## 边界

本 skill 产出面向实现的 **`spec.md`**。须基于真实代码上下文撰写，不允许只依据需求文本做「空中方案」。不再单独生成 `plan.md`，计划内容并入 spec。

**输入**：用户提供或指定的需求文档（常见为 `prd.md`，含 YAML Front Matter 与 `dependency` 前置 PRD；亦可是用户口述 + 路径）。

---

## 使用说明

1. 先读取需求文档：
   - 默认：`.apm/kb/docs/Iterations/<需求名称>/prd.md`（含 YAML Front Matter：`date`、`dependency`）；若 `dependency` 非空，一并读取所列前置 PRD
   - **非标准输入**（用户口述、自定义路径）：以用户指定路径为准；`dynamic` 记录 `requirement_path`（口述时可记「用户口述」或等价说明）
   - **APM 可用时**：`apm read` + `apm kb search --q "<需求名称/关键词>"` 检索相关历史方案与上下文（仅此步可由主代理直接做）
   - **无 APM 环境**：直读 `.apm/kb/docs/...` 或 `requirement_path`
   - **阶段完成**：更新 `dynamic`（需求名称、`requirement_path`、设计任务状态）与 `persist`（需求名称、`requirement_path`）
2. **在生成 `spec.md` 之前，必须先完成代码探索**（不可跳过）：
   - 主代理基于需求文档（`requirement_path`）列出可能涉及的文件、模块、接口，并用 `apm kb search`（或直读 kb）补充检索相关 spec、变更记录与模块文档（仅此步可由主代理直接做）
   - **深入阅读代码、确认实现与约束须派遣多个子代理**（见「探索阶段」），主代理 **不得** 自行深入读代码、扫目录替代子代理
   - 主代理根据各子代理返回的 **探索报告** 汇总：影响范围、兼容性风险、技术边界、关键模块映射
   - **阶段完成**：更新 `dynamic`（探索报告摘要、影响范围、待设计项）与 `persist`（现状约束、关键模块映射）
3. 完成代码探索后，再生成方案文档并写入知识库：
   - `.apm/kb/docs/Iterations/<需求名称>/spec.md`
   - **APM 可用时**写完后执行 `apm kb index rebuild`；无 APM 时直写文件即可
   - **阶段完成**：更新 `dynamic`（spec 路径、状态「待用户确认」）与 `persist`（spec 路径、核心方案要点、主要风险）
4. `spec.md` 须符合「文档格式规范」（YAML Front Matter + 正文），且必须基于真实代码上下文，不允许只依据需求文本做“空中方案”。
5. SPEC 正文必须面向实现，至少包含：
   - 总体实现思路与架构
   - 最终项目结构
   - 文件级改动点
   - 兼容性或迁移说明（如需要）
   - 分步骤实现计划
   - 测试策略与测试用例
   - 风险与回滚方案
6. 保证可执行性：每一步都应可落实、可验证。
7. **详细实现步骤**须逐步标注，便于后续按 spec 编码与验收：
   - 格式：`Step N — phase-<id> — blocking: yes|no — qa: auto|manual_user`
   - `phase-<id>` 中 `<id>` 用 kebab-case（如 `phase-auth-login`）
   - 测试用例 id 格式 `T-<模块缩写><序号>`（如 `T-W1`）；每条 T 须能映射到至少一个 Step
   - `blocking: yes`：该步骤为交付硬门槛
   - `qa: manual_user`：真机/录屏等，标注为合并后用户验收，**不**作为自动门禁阻塞项
8. 落盘后请求用户确认 `spec.md`；**确认后**：更新 `dynamic`（状态「已完成」、`spec_draft_confirmed: yes`）与 `persist`（已确认的方案要点）。
9. 探索报告摘要可写入 **Context Bundle**（见下），供后续实现参考；**探索不构成跳过 impl 子代理的依据**。

## 探索阶段（生成 spec 之前必做）

**探索编排**（并行 fan-out + 可选补派）：**不是** loop skill 的动态 DAG；主代理不得因补派而改代码或跳过汇总写 spec。

探索由 **主代理编排、子代理执行**；主代理依据探索报告撰写 SPEC，不替代子代理读代码。

### 主代理职责

1. 读完需求文档后，将技术探索面拆为 **2–4 个互不重叠的子任务**（例如：核心模块实现、API/事件契约、测试与构建约束、关联历史 spec/变更）
2. **并行** 派遣 **2–4** 个 readonly 探索子代理（`Task`，`subagent_type: explore`，`readonly: true`），**同步等待** 全部返回
3. 汇总各报告为影响分析与「现状约束」，再撰写 `spec.md`
4. 报告有缺口、与 PRD 矛盾或涉及未知模块时，补派聚焦探索（仍由子代理执行）；**补派上限 2 轮**

### 子代理派遣规范

| 项 | 值 |
|----|-----|
| 工具 | `Task`，`subagent_type: explore` |
| readonly | **true** |
| 并行 | 2–4 个，同步等待 |
| 失败 | 重试一次 → 主代理手工 readonly 探索，标注「手工探索」，dynamic 记录 |

### 主代理禁止

- 自行深入阅读源码、遍历目录、分析调用链与实现细节（应写入子代理 prompt）
- 未等探索子代理返回即撰写 spec 或宣称「已了解现状」

### 探索子代理须返回（探索报告）

每条报告须为中文，结构如下：

```text
1）探索范围：本任务覆盖的模块/路径
2）现状实现摘要：关键类/函数/入口、数据流、依赖关系
3）与 PRD 相关的约束：兼容性、配置、测试、构建、既有契约
4）变更影响面：建议修改的文件/模块列表
5）风险与疑点：未读清处、与 PRD 可能冲突处
6）关键证据：文件路径、符号名、测试路径（列表即可）
```

### 探索维度参考

| 探索项 | 目的 |
|--------|------|
| PRD 涉及的核心模块 | 确认当前实现、扩展点、不可动边界 |
| API / 事件 / 配置契约 | 避免 spec 与既有接口冲突 |
| 测试与 CI 约束 | 保证方案可验证、可落地 |
| 历史 spec / 变更记录 | 对齐命名、复用既有决策 |

### 探索子代理 prompt 模板

```text
【语言要求】
- 全程使用中文；路径、符号、命令可保留原文

请以 readonly 模式探索下列范围，为 SPEC 撰写提供技术现状与影响分析。

仓库：<REPO_PATH>
需求文档路径：<REQUIREMENT_PATH>（标准 PRD 或用户指定路径）
探索范围：<SCOPE，如「agent 生命周期相关模块与 hook 注册」>
请先列出将阅读的文件/目录，再按「探索报告」结构返回。
禁止修改任何文件。
```

探索报告摘要写入 `dynamic` / `persist`；`spec.md` 中的变更点与实现步骤须能追溯到探索报告中的证据。

## Context Bundle（可选，供后续实现参考）

```yaml
iteration_name: ...
requirement_path: ...
spec_path: ...
explore_summary: ...
impact_files: [...]
constraints: [...]
blocking_steps: [...]
```

---

## 阶段记忆更新

每阶段结束后执行（多行正文用 heredoc 管道：`cat <<'EOF' | apm dynamic write --stdin`；persist 同理；`replace` 更新过时条目；短句可用 `--text`）：

| 阶段 | dynamic（当前任务） | persist（跨会话结论） |
|------|---------------------|---------------------|
| 需求读取 | 需求名称、`requirement_path`、设计任务状态 | `requirement_path` |
| 探索完成 | 探索报告摘要、影响范围、技术约束、待设计方案 | 现状约束、关键模块与接口映射 |
| spec 落盘 | spec 路径、待确认项 | spec 路径、核心方案要点、主要风险 |
| 用户确认 | 状态「已完成」、`spec_draft_confirmed: yes` | 已确认的方案要点 |

- `dynamic`：全量覆盖当前任务进度与下一步。
- `persist`：仅写入已确认、可长期复用的结论；避免堆砌过程细节。
- **无 APM 环境**：`dynamic` / `persist` 可写入 `docs/.iteration-state.yaml`（或用户指定路径），或在对话内用 YAML 块等价维护；字段含义与上表一致。

## 环境与工具 fallback

- **无 APM 环境**：知识库直读 `.apm/kb/docs/...` 或 `requirement_path`；spec 落盘直写等价路径；`dynamic` / `persist` 见「阶段记忆更新」
- 下列 **`apm` 命令仅在 APM 可用时执行**；不可用时以读/写文件与 `docs/.iteration-state.yaml`（或对话内状态）为准

## 执行检查清单（每次都要走完）

- [ ] 已读取需求文档（默认 `prd.md` 或 `requirement_path`）；标准 PRD 时已校验 Front Matter 并读取 `dependency` 所列前置 PRD（如有）
- [ ] **APM 可用时**已 `apm read` / `apm kb search`（不可用时已直读 kb 或 `requirement_path`）
- [ ] 需求读取后已更新 `dynamic` 与 `persist`（含 `requirement_path`）
- [ ] 已派遣多个 readonly 探索子代理并 **同步等待** 全部探索报告
- [ ] 主代理已汇总探索报告（影响范围、现状约束、关键模块映射；非主代理直接读代码）
- [ ] 探索后已更新 `dynamic` 与 `persist`
- [ ] 已在方案中体现现状约束与影响分析
- [ ] 已生成 `.apm/kb/docs/Iterations/<需求名称>/spec.md`（含 YAML Front Matter：`date`）；**APM 可用时**已 `apm kb index rebuild`
- [ ] spec 落盘后已更新 `dynamic` 与 `persist`
- [ ] 已请求用户确认 `spec.md`
- [ ] 用户确认后已更新 `dynamic` 与 `persist`

## 文档格式规范

### PRD（读取时校验）

标准 `prd.md` 与标准 PRD Front Matter 一致，须含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `date` | string | `YYYY-MM-DD` |
| `dependency` | string \| string[] | 前置 PRD 路径（相对 `kb/docs/`）；无则 `[]` |

非标准需求文档（用户口述、自定义路径）可无 Front Matter；主代理须在 `spec.md` 正文（如「需求来源」）注明来源与关键约束。

### SPEC（本 skill 产出）

`spec.md` 必须以 **YAML Front Matter** 开头，紧跟 Markdown 正文。

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `date` | string | 是 | 文档日期，ISO 8601：`YYYY-MM-DD`（默认取落盘当日） |

## 文档模板

```markdown
---
date: YYYY-MM-DD
---

# <需求名称> 技术规格（SPEC）

## 设计目标

（非标准 `prd.md` 输入时，在此或「需求来源」小节注明 `requirement_path` 与来源）

## 总体方案

## 最终项目结构

## 变更点清单

## 详细实现步骤

（每步一行，须含 phase / blocking / qa 标注；`phase-<id>` 用 kebab-case）

- Step 1 — phase-auth-login — blocking: yes — qa: auto：<描述>
- Step 2 — phase-auth-login — blocking: yes — qa: auto：<描述>
- Step 5 — phase-auth-login — blocking: no — qa: manual_user：Android 录屏验收（合并后用户执行）

## 测试策略
### 测试用例

（用例须带 id，如 `T-W1`；`T-<模块缩写><序号>` 须能映射到 Step，供验收矩阵对照）

- T-W1 — blocking: yes — …

## 风险与回滚方案
```
