---
name: design-proposal
description: 用于在已有 PRD 的前提下输出技术规格（SPEC），整合技术设计与实现计划，作为编码前唯一技术文档。
disable-model-invocation: true
---

# 方案设计

## 使用说明

1. 先读取需求文档（APM 知识库）：
   - `.apm/kb/docs/Iterations/<需求名称>/prd.md`（须含 YAML Front Matter：`date`、`dependency`）
   - 若 `dependency` 非空，一并读取所列前置 PRD
   - 配合 `apm read` 与 `apm kb search --q "<需求名称/关键词>"` 检索相关历史方案与上下文
   - **阶段完成**：更新 `dynamic`（当前需求、PRD 路径、设计任务状态）与 `persist`（需求名称、PRD 路径）
2. **在生成 `spec.md` 之前，必须先探索相关项目代码**（不可跳过）：
   - 基于 `prd.md` 列出可能涉及的文件、模块、接口
   - 用 `apm kb search` 补充检索相关 spec、变更记录与模块文档
   - 阅读关键文件并确认当前实现与约束
   - 明确影响范围、兼容性风险、技术边界
   - **阶段完成**：更新 `dynamic`（探索结论、影响范围、待设计项）与 `persist`（现状约束、关键模块映射）
3. 完成代码探索后，再生成方案文档并写入知识库：
   - `.apm/kb/docs/Iterations/<需求名称>/spec.md`
   - 写完后执行 `apm kb index rebuild`
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
7. 编码前先请求用户确认 `spec.md`；**用户确认后**：更新 `dynamic`（状态「spec 已确认，可进入实现」）与 `persist`（已确认的方案要点）。
8. 明确边界：不再单独生成 `plan.md`，计划内容并入 `spec.md`。

## 阶段记忆更新

每阶段结束后执行（多行正文用 `apm dynamic write --stdin`；`apm persist write --stdin` 或 `replace` 更新过时条目；短句可用 `--text`）：

| 阶段 | dynamic（当前任务） | persist（跨会话结论） |
|------|---------------------|---------------------|
| PRD 读取 | 需求名称、PRD 路径、设计任务状态 | PRD 路径 |
| 探索完成 | 影响范围、技术约束、待设计方案 | 现状约束、关键模块与接口映射 |
| spec 落盘 | spec 路径、待确认项 | spec 路径、核心方案要点、主要风险 |
| 用户确认 | 状态「可进入实现」、分支/worktree 建议 | 已确认的方案要点 |

## 执行检查清单（每次都要走完）

- [ ] 已读取 `.apm/kb/docs/Iterations/<需求名称>/prd.md`（含 Front Matter），并已读取 `dependency` 所列前置 PRD（如有）
- [ ] 已 `apm read` / `apm kb search`
- [ ] PRD 读取后已更新 `dynamic` 与 `persist`
- [ ] 已完成相关代码探索（文件/模块/接口）
- [ ] 探索后已更新 `dynamic` 与 `persist`
- [ ] 已在方案中体现现状约束与影响分析
- [ ] 已生成 `.apm/kb/docs/Iterations/<需求名称>/spec.md`（含 YAML Front Matter：`date`）并已 `apm kb index rebuild`
- [ ] spec 落盘后已更新 `dynamic` 与 `persist`
- [ ] 已请求用户确认 `spec.md` 后再进入编码
- [ ] 用户确认后已更新 `dynamic` 与 `persist`

## 文档格式规范

### PRD（读取时校验）

与 `requirement-review` 产出一致，须含 Front Matter：

| 字段 | 类型 | 说明 |
|------|------|------|
| `date` | string | `YYYY-MM-DD` |
| `dependency` | string \| string[] | 前置 PRD 路径（相对 `kb/docs/`）；无则 `[]` |

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

## 总体方案

## 最终项目结构

## 变更点清单

## 详细实现步骤

## 测试策略
### 测试用例

## 风险与回滚方案
```
