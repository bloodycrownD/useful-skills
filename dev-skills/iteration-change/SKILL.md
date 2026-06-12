---
name: iteration-change
description: 用于处理已有迭代内的需求变更，仅在 features 子路径下生成变更级 PRD 文档时使用。
disable-model-invocation: true
---

# 迭代内变更

## 使用说明

1. 将每次迭代内变更视为该需求下的 feature 级变更。
2. **在首次 `AskQuestion` 之前，必须先探索上下文**（不可跳过）：
   - 读取父级 `.apm/kb/docs/Iterations/<需求名称>/prd.md`（及已有 `spec.md` 若存在）
   - 执行 `apm read` 与 `apm kb search --q "Iterations/<需求名称>"` 检索相关文档与历史变更
   - 基于变更描述，探索可能影响的文件、模块、接口
   - 形成「变更前现状」摘要：原范围、当前实现、疑似影响面
   - **阶段完成**：更新 `dynamic`（变更名称、现状摘要、待澄清项）与 `persist`（父级 PRD 路径、原范围要点）
3. **必须使用 `AskQuestion` 与用户澄清变更内容**（至少覆盖以下四点）：
   - 变更动机与原因
   - 与原始范围相比发生了什么变化
   - 影响的模块与接口
   - 更新后的验收标准
   - **阶段完成**：更新 `dynamic`（已澄清的变更点、剩余缺口）与 `persist`（用户已确认的变更决策）
4. **必须生成变更级需求文档 `prd.md`**（APM 知识库）：
   - `.apm/kb/docs/Iterations/<需求名称>/features/<变更名称>/prd.md`
   - 写完后执行 `apm kb index rebuild`
   - **阶段完成**：更新 `dynamic`（feature PRD 路径、状态「待用户确认」）与 `persist`（变更名称、路径、范围变更与验收要点）
5. 本技能在当前流程中仅覆盖 `prd.md` 的产出与确认。
6. 若用户提出技术方案需求，可提示使用 `design-proposal` 生成 `spec.md`。
7. 完成 `prd.md` 后，**必须请求用户确认该文档**，然后结束本技能流程；**用户确认后**：更新 `dynamic`（状态「变更 PRD 已确认」）与 `persist`。
8. 文档内容质量标准与主需求一致：
   - 范围清晰
   - 需求表达明确、可实现
   - 测试用例明确

## 阶段记忆更新

每阶段结束后执行（`apm dynamic write --text "…"`；`apm persist write --text "…"` 或 `replace` 更新过时条目）：

| 阶段 | dynamic（当前任务） | persist（跨会话结论） |
|------|---------------------|---------------------|
| 探索完成 | 变更前现状、影响面、待澄清问题 | 父级 PRD 路径、原范围要点 |
| 澄清完成 | 变更动机、范围差异、影响模块 | 用户已确认的变更决策 |
| feature PRD 落盘 | feature PRD 路径、待确认状态 | 变更名称、路径、验收要点 |
| 用户确认 | 状态「变更 PRD 已确认」、下一步 | 已确认的变更范围与验收标准 |

## 执行检查清单（每次都要走完）

- [ ] 已读取父级 PRD，并已 `apm read` / `apm kb search`、完成代码探索
- [ ] 探索后已更新 `dynamic` 与 `persist`
- [ ] 已完成变更澄清（4 个维度）
- [ ] 澄清后已更新 `dynamic` 与 `persist`
- [ ] 已生成 `features/<变更名称>/prd.md` 并已 `apm kb index rebuild`
- [ ] PRD 落盘后已更新 `dynamic` 与 `persist`
- [ ] 已请求并等待用户确认 `prd.md`
- [ ] 用户确认后已更新 `dynamic` 与 `persist`
- [ ] 已在确认后结束本技能流程

## 必要产物

- `.apm/kb/docs/Iterations/<需求名称>/features/<变更名称>/prd.md`

## 文档模板

```markdown
# <变更名称> PRD

## 背景与变更动机

## 范围变更说明（相对原需求）

## 影响模块与接口

## 验收标准

## 测试用例
```
