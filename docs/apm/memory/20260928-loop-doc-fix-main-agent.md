---
date: 2026-09-28 14:01
title: loop skill 文档修正改主代理直改 + 子代理派遣策略的三个延伸讨论
keywords: code-review-loop, spec-check-loop, code-dev-loop, spec-fix, doc-fix, 主代理直改, 子代理, 审查并发, 上下文传递
abstract: 两个 loop skill 的 spec-fix/doc-fix 已由「默认派子代理 + trivial 豁免」反转为「主代理直接执行」；随后讨论了三个延伸方向——单文件豁免推广到 DAG、审查阶段拆「并行取证 + 单人裁决」提并发、临时文件传上下文——结论均已给出，待用户拍板是否落地。
---

user:
code-review-loop 和 spec-check-loop 这两个 loop 中，允许 spec 文件修正由主代理来进行，因为只是单文件更改，没必要用子代理。

assistant:
好的，把两个 skill 的默认行为整个反转了：code-review-loop 的 spec-fix 和 spec-check-loop 的 doc-fix 改为汇总后主代理直接编辑文档落盘。连带清理了旧机制的配套内容——trivial 豁免章节、spec-fix/doc-fix 的并行 wave 与同文件互斥约束、派遣用 prompt 模板、子代理失败兜底（「手工 spec-fix」「手工 doc-fix」标注）都删掉了，DAG 示例里的 spec-fix 节点也改成主代理在 wave 间直改。review 审查环节照旧必须派 readonly 子代理，保住审查独立性；个别 must-fix 需要读大量代码查证时，可以派 readonly 子代理先查证、文档仍由主代理编辑。改完后两个文件已同步到 .agents/skills 部署目录。

user:
其实我理解 DAG 派遣本身就应该对单文件（一两个文件）更改这种情况豁免，小文件更改很快，用子智能体传上下文反而麻烦，之前的豁免太严格了。另外审查阶段好像不容易并发派遣：dev 的 cr-func、review 和 check 的审查阶段常见都只派一个 agent，不像开发/探索阶段那样并发。为什么？把审查拆成「可并发的探索 + 基于探索结果的审查」两个小阶段能提并发吗？还有一个思路：上下文传递靠子智能体重复读文件，能不能让主代理把大部分上下文写进临时文件让子智能体读，提高效率？这些想法有道理吗？

assistant:
三个想法方向都成立，但各有一处要修正。一、单文件豁免：真正的成本变量是「主代理要为节点读多少东西、试错多少轮」而非文件数；文档类节点按文件数判断基本成立，代码类节点 1-2 个文件也可能要读十个文件才能动一行，建议文件数只做快路径辅助、上下文消耗为主判据；code-dev-loop 目前还是「默认子代理 + trivial 豁免」，可考虑对齐成「默认直改、超阈值才派遣」。二、审查难并发的根因：探索/开发是覆盖型任务可正交切分、结果可加法合并，审查是收敛型判断任务——并行切分会丢全局视野（跨模块矛盾看不见）、破坏严重度标定一致性、增加 must-fix 去重合并成本，且审查对象通常比探索对象小、fan-out 固定开销占比高，spec-check 每轮单审查者还承担轮间 must-fix 状态追踪。两段式提案（并行取证 + 单人裁决）有效但限于大对象，且裁决阶段必须保留按 file:line 回读代码的义务（不能只审摘要），否则违反「对照代码库」铁律；code-review-loop 的 scope 并行 + review-full 收口已是该结构的雏形。三、临时文件传上下文：成本模型要修正——子代理重读文件耗的是子代理上下文和墙钟，不耗主代理上下文；若主代理为写包自己去读代码，成本只是转移到最贵的主代理上。正确形态是「生产者写包、消费者读包」：第一个读该区域的子代理把稳定事实（模块地图、调用链、既往 findings）落盘成 evidence pack，后续轮次先读包再针对性读；包须带 sha/时间戳管理失效，被审对象本身（当前 diff/spec）必须新鲜读。想法三本质是想法二的基础设施，配套落地才成立。

user:
反正我觉得你比我懂，你帮我优化下这些 skill 吧，它们的 DAG 阶段确实需要优化下。

assistant:
按讨论的组合拳改了三个 skill。code-dev-loop：trivial 豁免重写为「轻量直改」——新增快路径（1–2 个文件、位置已知、验证输出短即直改），主判据仍是上下文消耗，并补了「试错预期高即使文件少也派遣」的提醒；impl/fix/verify 保持默认子代理（实现类节点整体吃上下文，全面反转会撑爆主代理），与两个 loop 的「文档默认直改」形成分工。code-review-loop：新增 evidence 节点与「大对象两段式评审」（wave-1 并行取证落盘 → wave-2 单 review 读包裁决，judge 禁止仅凭包下结论），动态 DAG 表加两段式拆分行。spec-check-loop：review 明确为 judge 角色，新增「大文档两段式审查」（并行 evidence 取证 + 单 judge 回读原文裁决），重编排表的两段式行替换原「拆分审查」。三个 skill 都加了「证据包」约定：生产者写（首个读该区域的子代理落盘稳定背景）、消费者读（后续节点先读包再针对性读）、路径 docs/iterations/<name>/cache/、包头记 sha 管失效、被审对象必须新鲜读。agile-dev 自己的 trivial 豁免自洽未动，术语与轻量直改不一致可后续对齐。三个文件已同步 .agents/skills 部署目录。

user:
统一吧（agile-dev 术语也对齐），然后提交 push，并更新根目录下的 .agents/skills。

assistant:
agile-dev 的 trivial 豁免统一为「轻量直改」：快路径（1–2 文件、位置已知、验证输出短）+ 主判据（上下文消耗）+ 不适用补充试错预期提醒，正文四处引用与检查清单、失败处理全部改名。确认仓库根目录没有 .agents 目录，部署位置就是用户主目录 C:\Users\BloodyCrown\.agents\skills，四个 skill 文件同步过去并做哈希校验一致。全部改动（四个 skill + 本记忆文件）提交到 main 并 push 到 origin。
