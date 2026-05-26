---
name: apm-usage
description: 指导 Agent 与开发者使用本仓库 APM CLI（init、read、role/persist/dynamic、kb 导入/索引/联想区、replace 局部更新）。在用户提到 apm、外置记忆、.apm、apm read、知识库、会话恢复或 Agent 初始化上下文时使用。
disable-model-invocation: true
---

# APM 使用指南

## 何时使用

- Agent **会话开始**：用 `apm read` 拉取角色、持久记忆、动态记忆与知识库联想。
- **任务进行中**：用 `apm dynamic write/replace` 更新当前进度；重要结论写入 `apm persist`。
- **沉淀文档**：将 Markdown 写入 `.apm/kb/docs/` 或 `apm kb import`，并重建索引。
- **改 APM 源码**：遵守下文开发约束，并跑 `npm test`。

## 前置条件

- 在**项目根目录**（含或将创建 `.apm/` 的目录）执行命令。
- 本地需已 `npm run build`；开发调试可用 `npx tsx src/index.ts <子命令>`。
- 二进制入口：`apm` 或 `npx apm`（`package.json` 的 `bin`）。

## 工作区布局（v2）

```
.apm/
  config.json
  status.json
  memory/
    role.md          # 角色
    persist.md       # 持久记忆
    dynamic.md       # 动态记忆（当前任务）
  kb/
    docs/            # 知识库 Markdown 树（迭代 PRD/SPEC 也放这里）
    dynamic/detail.md
    archive/         # dynamic 归档副本
    index/
      search.json.gz # 检索索引（gzip + MiniSearch JSON）
```

- **禁止**依赖旧布局（`.apm/role.md`、`.apm/persistence/`、`.apm/dynamic/`）；检测到旧树会报错并提示 `apm init`。
- 路径一律通过代码里的 `apmPaths()` 解析，**不要**硬编码 `.apm` 内部结构。

### Git 与版本控制

| 类型 | 是否提交 | 说明 |
|------|----------|------|
| `kb/index/**`（如 `search.json.gz`） | **否** | 检索索引为本地构建产物，可随时 `apm kb index rebuild` 再生 |
| 全部 `.md`（`kb/docs/`、`kb/dynamic/`、`kb/archive/`、`memory/*.md` 等） | **是** | PRD/SPEC、记忆、归档正文应纳入 Git，便于协作与评审 |
| `config.json`、`status.json` | 按团队约定 | 非索引文件；若需共享长度限制可提交 `config.json`，`status.json` 通常可忽略 |

项目根 `.gitignore` 建议**只忽略索引目录**，不要整棵忽略 `.apm/`：

```gitignore
# APM：索引可重建，不提交
.apm/kb/index/
```

克隆或拉取后若联想区提示索引缺失，在项目根执行一次 `apm kb index rebuild` 即可。

> **不要**使用 `.apm/` 或 `.apm/**` 整目录忽略——否则 `kb/docs` 下的 PRD/SPEC 无法被 Git 追踪。

## Agent 标准流程

```text
1. apm init                    # 首次或空目录（幂等）
2. apm read                    # 初始化上下文（必做）
3. … 执行任务 …
4. apm dynamic write --text …  # 全量更新进度；或 replace 做小范围修订
5. （可选）apm persist write/replace  # 长期规则/结论
6. （可选）kb 写入/导入 + apm kb index rebuild
```

### `apm read` 输出结构

按顺序输出（**无内容的区块会省略**）：

1. `# 角色` — `memory/role.md` 正文（已去 YAML front matter）
2. `# 持久记忆` — `memory/persist.md`
3. `# 动态记忆` — `memory/dynamic.md`
4. `# 联想区` — 基于 **role + persist + dynamic** 合并正文检索 `kb/`（**不含** `kb/index/`）

**联想区规则（摘要）：**

| 项 | 说明 |
|----|------|
| 详细区 | 最多 **5** 条：首行 `[匹配率%] kb相对路径 关键词(≤4)` + 最多 **3** 行 `行号\|正文` |
| 简略区 | 最多 **5** 条：仅首行，无匹配行 |
| 匹配率 | 当次 Top 结果内 BM25+ 分数归一化为 0–100 整数 |
| 无命中 | **不输出** `# 联想区` |
| 无索引 | 仍输出 `# 联想区` + 提示执行 `apm kb index rebuild` |

路径示例：`docs/Iterations/foo/spec.md`、`archive/dynamic-20260516-120000.md`。

## 常用命令

### 初始化与读取

```bash
apm init
apm read
```

### 记忆四段（结构相同）

适用：`role`、`persist`、`dynamic`、`kb dynamic`。

子命令：`show` | `write --text <正文>` | `replace --old <原文> --new <新文> [--all]`

> **已无 `edit` 子命令**（行号编辑已移除）；局部更新统一用 `replace`。

```bash
apm role show
apm role write --text "…"
apm role replace --old "旧片段" --new "新片段"
apm persist replace --old "…" --new "…"
apm dynamic write --text "…"
apm dynamic replace --old "下一步：…" --new "下一步：…"
apm kb dynamic replace --old "…" --new "…"
```

**`replace` 规则：**

| 情况 | 行为 |
|------|------|
| `--old` 在正文中 0 次 | 失败，**不改文件**（`--old text not found`） |
| `--old` 为空 | 失败（`--old must not be empty`） |
| 默认（无 `--all`） | 只替换**第一次**出现 |
| `--all` | 替换**全部**出现 |
| `--new` 可为 `""` | 表示删除 `--old` 片段 |
| 匹配方式 | 精确子串（与 `show`/`read` 正文一致，不做 trim/正则） |
| 替换后超长/过短 | 与 `write` 相同，受 `config` 的 `min`/`max` 约束，失败且不改文件 |

- **局部修改**：从 `show` / `read` **原样复制**子串作为 `--old`；多处相同文本需全改时加 `--all`。
- **全量覆盖**：仍用 `write`。
- Section 文件带 YAML front matter（`createdAt` / `updatedAt`），**不要**在 `read` 输出里手写 front matter。

### 动态记忆归档

```bash
apm dynamic archive   # 将 memory/dynamic.md 全文复制到 kb/archive/（带时间戳文件名）
apm dynamic clear     # 清空 dynamic 正文模板；不删除 archive 已有文件
```

### 知识库

```bash
apm kb import --from <目录>    # 复制目录下全部 .md 到 kb/docs/，并自动 rebuild 索引
apm kb write --path <相对路径> --text "<内容>"   # 写入 kb/docs/（路径相对 docs/，须 .md）
apm kb search --q "<查询>"     # BM25+ 检索，默认 Top 5
apm kb index rebuild           # 扫描 kb/ 下除 index/ 外全部 .md 并写 search.json.gz
apm kb dynamic show|write|replace   # 对应 kb/dynamic/detail.md
```

**迭代文档落盘（PRD/SPEC）：**

```text
.apm/kb/docs/Iterations/<需求名称>/prd.md
.apm/kb/docs/Iterations/<需求名称>/spec.md
```

或：`apm kb write --path Iterations/<名称>/prd.md --text "…"`，然后 `apm kb index rebuild`。

**索引注意：**

- 升级或联想异常时，先执行 **`apm kb index rebuild`**（索引内路径相对 `kb/`，如 `docs/foo.md`）。
- `kb write` / `dynamic archive` **不会**自动重建索引；大批量变更后应手动 `rebuild`。
- 索引文件仅落盘在 `kb/index/`，**不提交 Git**；协作者拉代码后自行 `rebuild`。

### 配置

```bash
apm config show
apm config set --section role|persist|dynamicDetail|kbDynamicDetail --min <n> --max <n>
```

## 典型场景

### 初始化知识库并联调 read

```bash
npm run build
apm init
apm kb import --from <含 .md 的目录>   # 若有现成文档树
apm kb index rebuild                   # import 已 rebuild 时可省略
apm read
```

### 会话恢复（Agent）

1. `apm read` 获取角色 + 规则 + 当前任务 + 相关文档联想。
2. 根据「动态记忆」的「下一步」继续执行。
3. 小改进度用 `apm dynamic replace`；整段重写用 `apm dynamic write`；稳定知识写入 `apm persist`。

### 测试/CI 前

```bash
npm run build
npm test
```

| 领域 | 测试文件 |
|------|----------|
| 布局 / init / kb / front matter | `tests/layout.spec.ts` |
| read 联想区 | `tests/read-association.spec.ts`（`T-READ-ASSOC-*`） |
| section replace | `tests/replace.spec.ts`（`T-REP-*`） |
| 共享 harness | `tests/helpers/cli-harness.ts` |

## 修改 APM 代码时的约束

1. **路径**：只用 `apmPaths()` / `resolveKbDocPath` / `resolveKbIndexedPath`。
2. **写入**：`atomicWrite` + `withGlobalLock` + `serialWrite`。
3. **检索**：索引与联想共用 `kb-index-service`（`kbTokenize`、MiniSearch）；联想关键词展示走 `kb-stopwords`（与索引分词分离）。
4. **局部替换**：`applySubstringReplace`（`src/core/substring-replace.ts`）→ `replaceSection` → `writeSection`。
5. **测试**：改动 CLI 行为须更新对应 `tests/*.spec.ts`（见上表）。
6. **迭代文档**：新 PRD/SPEC 写入 `.apm/kb/docs/Iterations/<名称>/`，写完执行 `apm kb index rebuild`。

## 故障排查

| 现象 | 处理 |
|------|------|
| `Old .apm layout detected` | 备份后删除旧树，`apm init` |
| `Incomplete .apm workspace` | `apm init` |
| `Knowledge index missing` | `apm kb index rebuild` |
| 联想区无结果 | 确认记忆正文与 kb 文档有共同检索词；执行 rebuild |
| `replace` 报 not found | `--old` 与磁盘正文不完全一致；先 `show` 再复制 |
| section 长度报错 | `apm config set` 调大 `max` 或缩短正文 / 缩小替换范围 |

## 延伸阅读（知识库内）

执行 `apm read` 后可在联想区命中，或直接在本地打开：

- `.apm/kb/docs/Iterations/apm-read-association-area/prd.md` / `spec.md`
- `.apm/kb/docs/Iterations/apm-kb-memory-layout-init/spec.md`
- `.apm/kb/docs/Iterations/apm-replace-remove-edit/prd.md` / `spec.md`
