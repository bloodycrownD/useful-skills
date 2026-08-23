---
date: 2026-08-23 21:30
title: APM 从 CLI 退役为纯约定，skill 文档同步迁移新口径
keywords: apm, 退役, 纯约定, apm-usage, prd-generate, spec-generate, agile-dev, brain-storm, code-dev-loop, code-review-loop, spec-check-loop, 记忆语义
abstract: APM 从 npm CLI 整体退役为纯约定：外置记忆就是 docs/apm/（RULE.md + memory/），无命令无服务，apm-usage/SKILL.md 重写为唯一权威（含 bash/powershell 初始化片段与记忆语义）。八个 skill 分两批迁到新口径：第一批 prd-generate 与 spec-generate，第二批 agile-dev、brain-storm、code-dev-loop、code-review-loop、spec-check-loop，均把 apm read 改为按「快速开始」读规则与最近记忆、合并 APM 可用/无 APM 二分、清掉 dynamic/persist 三段式说法，换成「同主题记忆文件追加轮次并刷新 date/abstract，新跨会话规则才更新 RULE.md」，两批约 grep 自查无残留。
---

user:
帮我移除kb search以及联想区吧，感觉真没什么用处

assistant:
好呀，把 `apm search` 命令和 `apm read` 的联想区整体砍掉了。APM 现在只剩 `apm init` 和 `apm read` 两个命令，read 输出规则区（RULE.md 全文）+ 最近记忆区（最近 5 条摘要）。

具体改动：
- 删除 `src/cli/commands/search.ts`、`kb-search-service.ts`、`read-association-service.ts`、`kb-stopwords.ts`、`markdown-body.ts`；`walkMarkdown` 挪进 `memory-service.ts`（唯一消费者）
- 卸掉 `minisearch` 依赖
- 测试：删 search/read-association/markdown-body-parity 三个 spec，`help.spec` 改为断言不含 search，`read.spec` 的 T-R8 改为断言不再输出联想区，`no-apm-residue.spec` 的禁止符号表加入 MiniSearch/buildMemoryIndex/computeReadAssociation 等
- 全部 14 条测试通过，`npm link` 的全局 apm 已重建生效（`apm search` 报 unknown command）
- useful-skills 侧：`apm-usage` 全面改写（两命令、查重改为看最近记忆+翻 memory 目录、主动回忆改为按文件名定位直读文件），另外 7 个 skill（agile-dev/brain-storm/code-dev-loop/code-review-loop/prd-generate/spec-check-loop/spec-generate）清掉联想区说法和残留的 `apm index build` 引用
- 已同步 8 个 skill 到 `~/.agents/skills/`，diff 校验一致

user:
我有个问题哈，看起来记忆文件，作为 agent 你自己 grep 就很方便了？apm 还要什么需要提供的能力吗？

assistant:
结论：不需要再加能力，APM 终态就该这么薄。关键词检索 agent 自己的 grep 更强（所以砍 search 是对的），文件发现 ls 就够。`apm read` 剩余价值三条：一是省工具调用和上下文（一次调用返回压缩摘要，而不是把 5 个文件全文拉进上下文，即「按时间序的廉价索引」）；二是排序依据是 front matter 的 date 而非文件名约定，grep/ls 做不到文档元数据级排序；三是统一契约，8 个 skill 引用同一命令，坏文件有降级展示。真正的产品是 apm-usage 那份约定，CLI 只是执行器，往回加索引/检索是在跟 agent 自带工具竞争。唯一候选 `read --num N` 也 deemed 不值得加。

user:
我突然觉得当前 apm 的功能不需要这么复杂的 cli 应用，读取 RULE 完全可以用 agent 自己的 read 工具。按时间规范读取 memory 也是用 bash/powershell 命令就能实现？而且也不复杂？

assistant:
对账后承认：bash 完全可替代，且也是一次工具调用（推翻上轮「省工具调用」论点，那条只在 agent 挨个 read_file 时成立）。bash 管线：grep -m1 '^date:' 取日期 + sort -r + head -5 + sed -n '1,6p' 取 front matter 摘要；若信任文件名 yyyyMMdd 前缀约定，ls | sort -r | head -5 更简。CLI 仅剩差异：跨 shell 一致性（powershell 需另写一套）、坏文件降级容错、约定钉死一处。给出两条路：A. 退役 CLI，apm-usage 变纯约定 + 内嵌 bash/powershell 片段，8 个 skill 改引用；B. 保留现状当跨平台兜底（已建成、维护近零）。若 agent 环境以 Linux/Mac 为主建议 A；常在 Windows 跑 agent 则 B 有实际价值。待用户拍板。

user:
已拍板方案 A：APM 整体退役，改为纯约定。apm-usage/SKILL.md 已重写完成（快速开始含 bash/powershell 初始化片段，记忆语义定义 memory 文件与 RULE.md 分工）。现在把 prd-generate 和 spec-generate 两个 SKILL.md 改到新口径，只改这两个文件。

assistant:
好，两个 skill 文档已按新口径改写完毕。共七类替换：apm read 改为「按 apm-usage「快速开始」读规则与最近记忆摘要」；「APM 可用时/无 APM 环境」二分合并（未接入约定即直读 docs/ 或用户指定路径，作兑底保留在「环境与工具 fallback」）；dynamic/persist 三段式记忆说法全部换成「同主题 memory 文件追加轮次并刷新 date/abstract，新跨会话规则才逆 RULE.md」；「阶段记忆更新」小节整体重写，iteration-state 只用于编排状态；探索维度表的「APM 辅助」列改为初始化片段说法；子代理派遣规范表与执行检查清单同步改；「下列 apm 命令仅在 APM 可用时执行」整行删除。grep 自查两文件已无 apm read/apm init/apm search/apm index/dynamic/persist/APM 可用/无 APM 残留。

user:
剩下的五个 skill 文档也迁到新口径吧：agile-dev、brain-storm、code-dev-loop、code-review-loop、spec-check-loop，只改这五个。规则：apm read 引用改为按 apm-usage「快速开始」读规则与最近记忆摘要；「APM 可用/无 APM」二分合并；dynamic/persist 三段式说法换成新记忆语义；code-dev-loop 的 Context Bundle 禁止写入对象改为记忆文件（memory 目录与 RULE.md），括注「无 APM 时」删掉；spec-check-loop 的审查轮重编排里「不是 apm dynamic 正文」改为「不是记忆文件正文」，「无 APM 时」句删 dynamic 半句保留编排状态等价维护；检查清单条目同步；其它 apm 命令、.apm/、联想区、index build、archive 引用一并清理，完成后 grep 自查。

assistant:
五个文件改完，grep 自查 apm read/apm init/apm search/apm index/dynamic/persist/APM 可用/无 APM/archive/.apm/ 零命中。agile-dev：Step 1 与检查清单两处 apm read 改为「快速开始」读法。brain-storm：删「无 APM 时勿尝试 apm 命令 fallback」；「APM 可用时可额外 apm read」改为「已接入 apm-usage 约定（存在 docs/apm/）时按其快速开始读」；记忆小节的 dynamic/persist 两条改为记忆文件追加轮次 + RULE.md 仅新跨会话规则才更新。code-dev-loop：「不是 apm dynamic 正文」改「不是记忆文件正文」；Context Bundle 禁止写入对象改为记忆文件（memory 目录与 RULE.md），括注改「（或对话内 YAML）」；开始前/Step 1/Step 3/检查清单的记忆说法全部换新语义。code-review-loop：trivial 标注位置从 dynamic「现状」改为记忆文件；「无 APM 时：用用户指定路径…」删前缀保留等价维护编排状态；「默认（有 APM / 迭代结构时）」改「默认（有迭代结构时）」。spec-check-loop：「不是 apm dynamic 正文」改「不是记忆文件正文」；无 APM 句删 dynamic 半句，保留用 docs/.iteration-state.yaml 或对话内 YAML 维护编排状态；「若项目使用 APM：apm read」改按快速开始读；准备完成/修复完成/用户确认后/失败处理/检查清单全部换新语义，persist 写入对象改 RULE.md。
