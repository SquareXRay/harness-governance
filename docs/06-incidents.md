# 06 · Incidents

Five failures that changed the design. Each is written as symptom → root cause → fix → general
lesson, because that is the only form that transfers.

五次改变了设计的故障。每一条都写成「症状 → 根因 → 修法 → 通用教训」，因为这是唯一能迁移的形式。

## 1. A gate blocked a task because of one character

**Symptom.** A dispatch containing the phrase "idempotent release" (幂等放行) was refused with
`classification=audit`. Three retries, same result; deleting the declared task type changed the
reason but not the outcome.

**症状。** 一条含「幂等放行」（idempotent release）的派发，被以 `classification=audit` 拒绝。重试三次，结果一样；把声明的任务类型删掉，只改了理由，没改结果。

**Root cause.** The read-only detector matched 13 raw substrings anywhere in the text. One was
"awaiting approval" (等放行) — and "idempotent release" contains it as a substring. No word-boundary
logic exists in CJK, so containment was never a valid predicate.

**根因。** 只读检测器在整段文本里匹配 13 个原始子串。其中一个是「等放行」（awaiting approval）——而「幂等放行」正好把它包含成子串。中文里根本不存在词边界逻辑，所以「包含」从来就不是一个有效的谓词。

**Fix.** Split the vocabulary by semantic role: *intent* patterns (negation + verb + object),
*scope* descriptors (stripped before matching), and *state* words (`awaiting approval`) which no
longer participate in the verdict at all. Replayed 1,017 historical dispatch events: 25 signals
disappeared (9 true releases, 16 still caught by the ambiguity tier), zero new hits, zero missed
true read-only tasks.

**修法。** 按语义角色拆词表：*意图*模式（否定 + 动词 + 宾语）、*作用域*描述符（匹配前先剥掉）、以及*状态*词（`awaiting approval`），后者完全不再参与判决。回放了 1,017 条历史派发事件：25 个信号消失（其中 9 条是真放行，16 条仍被歧义层抓住），新增命中 0 条，真正的只读任务漏拦 0 条。

**Lesson.** A substring table is not a classifier. When a rule's job is to read intent, encode
structure; when a word's job is to describe state, keep it out of the verdict path.

**教训。** 子串表不是分类器。当一条规则的职责是读意图时，就把结构编进去；当一个词的职责是描述状态时，就别让它进入判决路径。

---

## 2. The maintenance command that would have eaten its own index

**Symptom.** `--split-archive --dry-run` reported "115 entries across 3 volumes". The files on disk
were 5 volumes / 155 entries. The real command would have rewritten them.

**症状。** `--split-archive --dry-run` 报「3 卷共 115 条」。磁盘上是 5 卷 / 155 条。真跑那条命令，就会把它们重写掉。

**Root cause.** Three defects stacked:

**根因。** 三个缺陷叠在一起：

1. the "is this already an index?" guard required *zero* `N. ` lines in the file — but the index
   contained 114 human-maintained entry mappings shaped exactly like entries;
2. the write path backed up the hot file, the index and the counter, **not the volumes** — the
   same-name `-unknown.md` volume would have been silently overwritten;
3. the index generator emitted one extra column that the on-disk index did not have, proving the
   index had drifted away from its generator long before.

1. 「这已经是索引了吗」那道守卫要求文件里**一行** `N. ` 都没有——可索引里本来就含 114 条人维护的条目映射，形状和条目一模一样；
2. 写入路径备份了热文件、索引和计数器，**唯独没备份分卷**——那个同名的 `-unknown.md` 分卷会被静默覆盖；
3. 索引生成器多吐出一列，而磁盘上的索引并没有这一列，说明索引早就和自己的生成器漂移了。

**Fix.** Structural detection (a volume table with declared ranges/counts/bytes) instead of a line
pattern; full pre-write backup including every volume; and a pre-write consistency check — parsed
total must equal the sum of per-volume counts, otherwise refuse. The dry run now reports 5 volumes /
155 entries, each marked consistent.

**修法。** 改成结构检测（一张带声明区间/条数/字节数的分卷表），不再依赖行模式；写前全量备份，包括每一个分卷；再加一道写前一致性检查——解析出的总数必须等于各分卷条数之和，否则拒绝写入。现在 dry run 报的是 5 卷 / 155 条，每一卷都标为一致。

**Lesson.** A maintenance script is production code. If it rewrites data, it must (a) identify its
targets structurally, (b) back up everything it can touch, (c) prove its arithmetic before writing,
and (d) have a dry-run that is byte-identical in behaviour to the real run.

**教训。** 维护脚本就是生产代码。如果它会重写数据，它就必须 (a) 用结构识别目标，(b) 备份它可能碰到的每一样东西，(c) 写之前先证明自己的算术，(d) 有一个行为与真跑逐字节相同的 dry run。

---

## 3. Tests bound to live production data

**Symptom.** Two tests failed for weeks. They asserted that the production work queue contained
specific commented-out lines "for reference", and counted the string "needs approval" at least twice.

**症状。** 两个测试红了整整几周。它们断言生产工作队列里「为了参照」保留着某些被注释掉的行，并且断言字符串「needs approval」至少出现两次。

**Root cause.** The queue was legitimately cleaned; the assertions encoded a snapshot of a live file
as an invariant. Green tests depended on nobody tidying up.

**根因。** 那个队列是被正常清理掉的；而这些断言把一份活文件的快照当成了不变量。测试能绿，靠的是没人去整理。

**Fix.** Conditional assertions: *if* a destructive item exists, it must be commented and labelled;
if none exists, skip with a printed reason. Plus a fixture-based test that proves the guard still
fires on a synthetic violation — so "no items" can never be mistaken for "guard removed".

**修法。** 改成条件断言：*如果*存在破坏性条目，它必须被注释掉并打上标签；如果一条都没有，就跳过并打印原因。另外补一个基于夹具的测试，证明守卫在一个合成违规上仍然会响——这样「没有条目」就永远不会被误当成「守卫被删了」。

**Lesson.** A test may read live data but must not require it. Whenever you relax an assertion that
guards something, add a fixture that proves the guard still bites.

**教训。** 测试可以读活数据，但不能要求活数据存在。每当你放宽一条守卫某物的断言，就补一个夹具，证明那个守卫仍然咬人。

---

## 4. The queue line that skipped its own window check

**Symptom.** Three tests red: an active line in the maintenance queue had no "only run while the
service is stopped" guard, and its command contained a token on the destructive list.

**症状。** 三个测试变红：维护队列里有一条活着的命令行，缺了「只在服务已停止时才执行」这道守卫，而且它的命令里含有一个在破坏性清单上的 token。

**Root cause.** A cleanup pass (well-intentioned) re-enabled a previously commented task without
carrying over the guard that had been added after an incident where a test run consumed the real
production queue.

**根因。** 一次（本意良好的）清理，把一条先前被注释掉的任务重新启用了，却没有把后来补上的那道守卫一起带过来——那道守卫是在一次事故之后加的：一次测试运行吃掉了真正的生产队列。

**Fix.** Re-added the guard (evaluated per line), removed the bare destructive token, and replaced
it with an exact allowlist: one script name, all python invocations on the line must be in the list,
unrecognised invocation shapes fail closed. Any other script using the same flag is still a violation.

**修法。** 把守卫加回来（按行求值），删掉那个裸的破坏性 token，换成一个精确白名单：一个脚本名，该行上所有 python 调用都必须在名单内，识别不出的调用形状一律失败朝严。任何别的脚本用了同一个 flag，仍然算违规。

**Lesson.** Exceptions must be *narrower than the rule*, expressed in the syntax the check can
verify (script basename), and documented in the data file they apply to. A permission granted by
phrase ("only for the sync script") is not a permission the checker can enforce — the gate proved
this by rejecting its own explanatory comment.

**教训。** 例外必须*比规则更窄*，要用检查器真正能验证的语法来表达（脚本基名），并且写进它们所作用的那份数据文件里。用一句话授予的许可（「只给同步脚本用」）不是检查器能执行的许可——闸拒绝了它自己那句说明性注释，正好证明了这一点。

---

## 5. Seven cards said "running"; nobody was running

**Symptom.** A cleanup pass found seven cards in `running` that had never been claimed and had not
moved for 2.6–5.7 hours. The panel showed them as work in progress; the ledger had nothing alive
behind them.

**症状。** 一次清整发现七张卡停在 `running`：从没被 claim 过，2.6–5.7 小时没动过。面板把它们显示成「进行中」，台账里却没有任何活着的东西。

**Root cause.** Three defects stacked: the card schema carries no session or owner link, so the
panel could only render the card's own status; the board-to-ledger sync asked the card backend for
one card at a time until the whole pass timed out; and when the backend held no job container the
sync **skipped silently**, leaving terminal ledger rows looking open forever.

**根因。** 三个缺陷叠在一起：卡片 schema 里没有会话或 owner 关联，面板只能渲染卡自己的状态；看板与台账的同步逐卡调用后端，直到整轮超时；后端没有 job 容器时，同步**静默跳过**，于是台账已终态的行看起来永远开着。

**Fix.** Bind cards to ledger rows by `cardId`; sync from a single snapshot, with drift exit `0`
and only a check failure exit `2`; never skip silently — an unresolved terminal row is handed to
the reconciler; and let the reconciler demote a `running` card when the ledger is not running, has
no claim, or all three activity signals have been silent for more than 20 minutes.

**修法。** 卡片用 `cardId` 绑到台账行；同步改成单次快照，查到漂移交 `0`、只有检查出错才交 `2`；不再静默跳过——没收口的终态行交给对账器；并由对账器在「台账不在跑 / 没有 claim / 三项活性证据静默超过 20 分钟」时把 `running` 卡退回 `ready`。

**Lesson.** A status field is not a state. If a surface cannot name *who* is working and *what
changed recently*, its "running" is a claim with no evidence — and the fix is a comparison, not a
bigger button.

**教训。** 状态字段不是状态。一面展示层若说不出「**谁**在干」和「**最近**动了什么」，它的「进行中」就是一句没有证据的说辞——解法是加一层比较，而不是加一个更大的按钮。

