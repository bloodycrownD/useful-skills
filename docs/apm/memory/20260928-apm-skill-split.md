---
date: 2026-09-28 14:17
title: apm-usage 拆分为 apm-recall（读侧）与 apm-record（写侧）
keywords: apm-usage, apm-recall, apm-record, skill 拆分, 记忆约定, 交叉引用
abstract: apm-usage 按读写职责拆成两个 skill：apm-recall 拿走「快速开始」（会话初始化、主动回忆、读的边界），apm-record 拿走「记忆语义」「记忆文件格式」「查重」「目录初始化」并保留唯一权威声明。AGENTS.md 与六个 skill 的交叉引用换名（章节名保留故不断链），仓库与 .agents/skills 部署两处同步删除旧 apm-usage。
---

user:
帮我把 apm-usage 这个 skill 拆分吧，我发现恢复记忆与记录记忆是不相同的领域，它们合并到一个 skill 其实不方便使用。你觉得呢？

assistant:
同意拆：读和写的触发时机完全不同——恢复发生在会话开始和需要回忆时，记录发生在请求落盘和阶段推进时；合并的 skill 每次触发把两半一起载入、description 要同时覆盖两堆关键词，触发精度差。拆分时防了两个坑：一是格式漂移，让 apm-record 独占目录结构与记忆文件格式的唯一权威、apm-recall 只拥有读路径，两边互相声明对方是另一半；二是引用断裂，全家桶大量「apm-usage『快速开始』/『记忆语义』」的引用靠保留章节名、只换 skill 名来保住（快速开始归 apm-recall、记忆语义归 apm-record）。共改七个文件（AGENTS.md + agile-dev/brain-storm/code-dev-loop/prd-generate/spec-check-loop/spec-generate），脚本批量替换后 grep 自查仅历史记忆文件残留旧名（属正常历史记录）。仓库删除 dev-skills/apm-usage，新增 apm-recall 与 apm-record；部署目录同步两增一删。改动未提交，待用户确认后可随下次提交入库。

user:
提交 push 了吗？同步到根目录吗？

assistant:
拆分当轮没有提交（用户未要求），这一轮补做：全部改动（七个改引用文件 + apm-usage 删除 + apm-recall/apm-record 新增 + 本记忆文件）提交到 main 并 push 到 origin。部署目录在拆分当轮已同步（两增一删），本轮复验哈希一致。
