---
date: 2026-08-22 21:30
title: APM 移除 search 命令与联想区，收敛为 init/read 两命令
keywords: apm, search, 联想区, minisearch, 命令收敛
abstract: 用户觉得 search 和联想区没什么用处，要求整体移除。APM 只剩 init/read 两个命令，read 输出规则区+最近记忆区；minisearch 依赖与检索相关源码/测试全部删除，8 个 skill 文档同步去掉旧说法并清理残留的 apm index build 引用，已同步到 ~/.agents/skills。
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
（后续注意）skill 文档里还有「按 apm-usage 刷新 dynamic/persist」的旧说法，dynamic/persist 已经不存在了，后续要改成「更新 memory 文件 / RULE.md」的新约定。

user:
我有个问题哈，看起来记忆文件，作为 agent 你自己 grep 就很方便了？apm 还要什么需要提供的能力吗？

assistant:
结论：不需要再加能力，APM 终态就该这么薄。关键词检索 agent 自己的 grep 更强（所以砍 search 是对的），文件发现 ls 就够。`apm read` 剩余价值三条：一是省工具调用和上下文（一次调用返回压缩摘要，而不是把 5 个文件全文拉进上下文，即「按时间序的廉价索引」）；二是排序依据是 front matter 的 date 而非文件名约定，grep/ls 做不到文档元数据级排序；三是统一契约，8 个 skill 引用同一命令，坏文件有降级展示。真正的产品是 apm-usage 那份约定，CLI 只是执行器，往回加索引/检索是在跟 agent 自带工具竞争。唯一候选 `read --num N` 也 deemed 不值得加。
