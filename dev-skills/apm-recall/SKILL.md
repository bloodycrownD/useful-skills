---
name: apm-recall
description: APM（Agent Persistence Memory）记忆读取侧：会话开始读 docs/apm/RULE.md 与最近 5 条记忆摘要，主动回忆按文件名或关键词定位、直读完整记忆。没有 CLI、没有服务，agent 用现成工具直接读。在提到 apm、外置记忆、会话恢复、回忆既往决策或结论，或需要按统一约定读取 docs/apm/ 时使用。读路径以本文件为权威；写什么、怎么写见 apm-record。
disable-model-invocation: true
---

# APM 记忆读取（Recall）

APM 不是工具，是一份**目录与文件约定**：没有 CLI、没有服务、没有索引，记忆就是 `docs/apm/` 下的 markdown 文件——`RULE.md` 装持久规则，`memory/` 下的 `yyyyMMdd-name.md` 装对话记忆，跟着项目一起进版本控制。本文件负责**怎么读**：会话开始读什么、中途怎么回忆。写侧（写什么、什么格式、怎么查重）见 `apm-record`。

## 快速开始

会话开始时（在项目根执行）：

```bash
# 1. 读持久规则（文件不存在就跳过）
cat docs/apm/RULE.md

# 2. 最近 5 条记忆的摘要（按 front matter date 降序，输出文件路径 + front matter）
for f in docs/apm/memory/*.md; do
  [ -e "$f" ] || continue
  d=$(grep -m1 '^date:' "$f" | sed 's/^date:[[:space:]]*//')
  printf '%s\t%s\n' "$d" "$f"
done | sort -r | head -5 | cut -f2- | xargs -I{} sed -n '1,6p' {}
```

powershell 等价片段：

```powershell
Get-ChildItem docs/apm/memory/*.md | ForEach-Object {
  $d = (Select-String -Path $_ -Pattern '^date:\s*(.+)$' | Select-Object -First 1).Matches[0].Groups[1].Value
  [PSCustomObject]@{ Date = $d; File = $_ }
} | Sort-Object Date -Descending | Select-Object -First 5 | ForEach-Object {
  Get-Content $_.File | Select-Object -First 6
}
```

看完摘要想读某条完整记忆，按路径直接读那个文件就好。之后…执行任务，值得记的东西按 `apm-record` 的约定落盘。

---

## 读之前知道在读什么

| 层 | 路径 | 装什么 | 初始化怎么读 |
|----|------|--------|--------------|
| 持久规则 | `docs/apm/RULE.md` | 换会话仍该遵守的约定、边界、已拍板决策（类 AGENTS.md） | **原样读全文** |
| 对话记忆 | `docs/apm/memory/yyyyMMdd-name.md` | 带时间的具体交流：用户问了什么、当时结论是什么 | 取**最近 5 条**的 front matter 摘要 |

记忆文件靠 front matter 的 `date` 排序（不是靠文件名），`title` / `keywords` / `abstract` 三个字段就是为「扫一眼判断要不要细读」准备的。

---

## 主动回忆

想确认某件事以前聊过没有：

1. 先看最近记忆摘要——多数近期问题在这里就能确认
2. 时间更久的，按 `docs/apm/memory/` 下文件名定位：`yyyyMMdd-简短标识` 自带日期和主题标识
3. 或用 `grep -il "关键词" docs/apm/memory/*.md` 搜正文，再直接读命中文件
4. 跨会话规则类的问题（「XX 是不是约定不许做」）先查 `RULE.md`，那里才是持久规则的家

---

## 读的边界

| 路径 | 判定 |
|------|------|
| `docs/apm/RULE.md` | 持久规则，初始化原样读取 |
| `docs/apm/memory/*.md` | 对话记忆，初始化取最近 5 条摘要 |
| `docs/`（项目根下其余 `.md`） | 项目文档，与 APM 无关，需要时直接读 |
| 仓库内其他 `memory/` 目录 | 与 APM 无关，**不是**可恢复的记忆 |
| `.apm/`（旧版运行态目录） | 已废弃，勿从其中恢复任何状态；残留按需清理 |

---

## 权威边界

读路径（何时读、读哪些、怎么检索）以本文件为权威；目录结构、记忆文件格式与写路径以 `apm-record` 为唯一权威。两个 skill 共同构成 APM 约定，其它 skill 引用时读侧指 `apm-recall`、写侧指 `apm-record`，勿另立第三处。
