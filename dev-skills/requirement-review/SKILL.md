---
name: requirement-review
description: 用于在实现前与用户澄清新需求并输出 PRD。先探索项目了解现状，再基于上下文澄清业务与产品需求，不产出技术实现 spec。
disable-model-invocation: true
---

# 需求评审

## 使用说明

1. **在首次 `AskQuestion` 之前，必须先探索项目**（不可跳过）：
   - 执行 `apm read`，并配合 `apm kb search --q "<关键词>"` 检索相关 PRD、历史迭代与文档
   - 基于用户描述，列出可能涉及的文件、模块、接口、页面/路由
   - 阅读关键实现，了解现有功能、用户流程、数据边界与命名习惯
   - 形成简短「现状摘要」：已有能力、缺口、可能影响范围（仅业务视角，不写技术方案）
   - **阶段完成**：更新 `dynamic`（现状摘要、探索结论）与 `persist`（已确认的项目术语、既有能力边界）
2. 完成探索后，使用 `AskQuestion` 与用户澄清需求细节：
   - 问题须基于探索结论，引用具体模块/流程/界面，避免空泛提问
   - 每次只问一个聚焦问题，直到关键信息明确：
     - 目标与业务价值
     - 目标用户与使用场景
     - 范围（包含 / 不包含）
     - 成功指标（可量化）
     - 验收标准
   - **阶段完成**：更新 `dynamic`（已澄清项、待确认项、下一步）与 `persist`（用户已确认的关键决策）
3. 澄清完成后，生成 PRD 并写入 APM 知识库：
   - `.apm/kb/docs/Iterations/<需求名称>/prd.md`
   - 可用 `apm kb write --path Iterations/<需求名称>/prd.md --text "…"`，或直接写文件
   - 写完后执行 `apm kb index rebuild`（便于 `apm read` 联想检索）
   - **阶段完成**：更新 `dynamic`（PRD 路径、状态「待用户确认」或「已完成」）与 `persist`（需求名称、PRD 路径、核心范围与验收要点）
4. `prd.md` 须符合「文档格式规范」（YAML Front Matter + 正文），默认输出轻量 PRD，正文至少包含：
   - 背景（含与现状的关系）
   - 目标（含成功指标）
   - 用户与场景
   - 范围（包含 / 不包含）
   - 核心需求（3-7 条）
   - 验收标准
5. 验收标准必须可测试、可判定，优先使用清单或 Given / When / Then 表达。
6. 明确告知生成路径，并请用户进行最终确认。
7. 明确边界：本 skill 只输出 PRD，不展开技术方案、接口设计、数据库结构、任务拆分等技术 spec 内容。
8. 如用户明确需要，再按需追加「扩展章节」（非默认）：
   - 约束与依赖
   - 非功能需求（仅业务/体验层面）
   - 风险与待确认项
   - 里程碑（更偏 plan，可选）

## 探索指引（Ask 之前必做）

优先使用 `Task`（`subagent_type: explore`）或等效方式快速摸清上下文，再进入提问。

| 探索项 | 目的 | APM 辅助 |
|--------|------|----------|
| 相关页面/路由/入口 | 确认用户从哪进入、现有流程是什么 | `apm kb search --q "<功能名/路由>"` |
| 核心模块与服务 | 理解业务边界、已有能力与缺口 | `apm kb search --q "<模块名>"` |
| 对外接口/事件/配置 | 判断需求是否涉及上下游或第三方 | 代码探索 + kb search |
| 命名与术语 | 与用户描述对齐，减少歧义 | `apm read` 联想区 |
| 已有 PRD/文档 | 避免重复定义、识别增量需求 | `apm kb search --q "Iterations/<需求>"` |

探索产出不要求单独写入文件，但须在后续提问与 PRD「背景」中体现。

## 阶段记忆更新

每阶段结束后执行（`apm dynamic write --text "…"`；`apm persist write --text "…"` 或 `replace` 更新过时条目）：

| 阶段 | dynamic（当前任务） | persist（跨会话结论） |
|------|---------------------|---------------------|
| 探索完成 | 现状摘要、涉及模块、待澄清问题 | 项目术语、既有能力边界 |
| 澄清完成 | 已确认目标/范围/指标、剩余缺口 | 用户已拍板的关键决策 |
| PRD 落盘 | PRD 路径、阶段状态、下一步（如 design-proposal） | 需求名称、PRD 路径、核心范围与验收要点 |

- `dynamic`：全量覆盖当前任务进度与下一步。
- `persist`：仅写入已确认、可长期复用的结论；避免堆砌过程细节。

## 执行检查清单（每次都要走完）

- [ ] 已执行 `apm read` 与 `apm kb search`，并完成代码探索、形成现状摘要
- [ ] 探索后已更新 `dynamic` 与 `persist`
- [ ] 已基于探索结论使用 `AskQuestion` 澄清关键信息
- [ ] 澄清后已更新 `dynamic` 与 `persist`
- [ ] 已生成 `.apm/kb/docs/Iterations/<需求名称>/prd.md`（含 YAML Front Matter：`date`、`dependency`）并已 `apm kb index rebuild`
- [ ] PRD 落盘后已更新 `dynamic` 与 `persist`
- [ ] 已明确 PRD 路径并请用户最终确认
- [ ] 未输出技术 spec（接口设计、库表、任务拆分等）

## 文档格式规范（PRD）

`prd.md` 必须以 **YAML Front Matter** 开头，紧跟 Markdown 正文。Front Matter 与正文之间、正文标题层级须符合下列约定。

### Front Matter 字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `date` | string | 是 | 文档日期，ISO 8601：`YYYY-MM-DD`（默认取落盘当日） |
| `dependency` | string \| string[] | 是 | 依赖的前置 PRD；路径相对 `kb/docs/`。无依赖时写 `[]` 或 `""` |

`dependency` 示例：

- 无前置：`dependency: []`
- 单个前置：`dependency: Iterations/<前置需求>/prd.md`
- 多个前置：`dependency: [Iterations/<需求A>/prd.md, Iterations/<需求B>/prd.md]`
- feature 级变更：`dependency: Iterations/<需求名称>/prd.md`（指向父级 PRD）

探索阶段若发现前置需求，须在澄清时与用户确认 `dependency` 取值后再落盘。

## 默认模板（轻量）

```markdown
---
date: YYYY-MM-DD
dependency: []
---

# <需求名称> PRD

## 背景

## 目标（含成功指标）

## 用户与场景

## 范围
### 包含范围
### 不包含范围

## 核心需求（3-7 条）

## 验收标准
```

## 扩展章节（按需添加，不默认）

```markdown
## 约束与依赖

## 非功能需求（业务/体验）

## 风险与待确认项

## 里程碑（可选，偏 plan）
```
