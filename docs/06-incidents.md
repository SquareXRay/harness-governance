# 06 · Incidents

Four failures that changed the design. Each is written as symptom → root cause → fix → general
lesson, because that is the only form that transfers.

## 1. A gate blocked a task because of one character

**Symptom.** A dispatch containing the phrase "idempotent release" (幂等放行) was refused with
`classification=audit`. Three retries, same result; deleting the declared task type changed the
reason but not the outcome.

**Root cause.** The read-only detector matched 13 raw substrings anywhere in the text. One was
"awaiting approval" (等放行) — and "idempotent release" contains it as a substring. No word-boundary
logic exists in CJK, so containment was never a valid predicate.

**Fix.** Split the vocabulary by semantic role: *intent* patterns (negation + verb + object),
*scope* descriptors (stripped before matching), and *state* words (`awaiting approval`) which no
longer participate in the verdict at all. Replayed 1,017 historical dispatch events: 25 signals
disappeared (9 true releases, 16 still caught by the ambiguity tier), zero new hits, zero missed
true read-only tasks.

**Lesson.** A substring table is not a classifier. When a rule's job is to read intent, encode
structure; when a word's job is to describe state, keep it out of the verdict path.

---

## 2. The maintenance command that would have eaten its own index

**Symptom.** `--split-archive --dry-run` reported "115 entries across 3 volumes". The files on disk
were 5 volumes / 155 entries. The real command would have rewritten them.

**Root cause.** Three defects stacked:

1. the "is this already an index?" guard required *zero* `N. ` lines in the file — but the index
   contained 114 human-maintained entry mappings shaped exactly like entries;
2. the write path backed up the hot file, the index and the counter, **not the volumes** — the
   same-name `-unknown.md` volume would have been silently overwritten;
3. the index generator emitted one extra column that the on-disk index did not have, proving the
   index had drifted away from its generator long before.

**Fix.** Structural detection (a volume table with declared ranges/counts/bytes) instead of a line
pattern; full pre-write backup including every volume; and a pre-write consistency check — parsed
total must equal the sum of per-volume counts, otherwise refuse. The dry run now reports 5 volumes /
155 entries, each marked consistent.

**Lesson.** A maintenance script is production code. If it rewrites data, it must (a) identify its
targets structurally, (b) back up everything it can touch, (c) prove its arithmetic before writing,
and (d) have a dry-run that is byte-identical in behaviour to the real run.

---

## 3. Tests bound to live production data

**Symptom.** Two tests failed for weeks. They asserted that the production work queue contained
specific commented-out lines "for reference", and counted the string "needs approval" at least twice.

**Root cause.** The queue was legitimately cleaned; the assertions encoded a snapshot of a live file
as an invariant. Green tests depended on nobody tidying up.

**Fix.** Conditional assertions: *if* a destructive item exists, it must be commented and labelled;
if none exists, skip with a printed reason. Plus a fixture-based test that proves the guard still
fires on a synthetic violation — so "no items" can never be mistaken for "guard removed".

**Lesson.** A test may read live data but must not require it. Whenever you relax an assertion that
guards something, add a fixture that proves the guard still bites.

---

## 4. The queue line that skipped its own window check

**Symptom.** Three tests red: an active line in the maintenance queue had no "only run while the
service is stopped" guard, and its command contained a token on the destructive list.

**Root cause.** A cleanup pass (well-intentioned) re-enabled a previously commented task without
carrying over the guard that had been added after an incident where a test run consumed the real
production queue.

**Fix.** Re-added the guard (evaluated per line), removed the bare destructive token, and replaced
it with an exact allowlist: one script name, all python invocations on the line must be in the list,
unrecognised invocation shapes fail closed. Any other script using the same flag is still a violation.

**Lesson.** Exceptions must be *narrower than the rule*, expressed in the syntax the check can
verify (script basename), and documented in the data file they apply to. A permission granted by
phrase ("only for the sync script") is not a permission the checker can enforce — the gate proved
this by rejecting its own explanatory comment.

---

## 中文摘要

四次事故对应四条设计结论：

1. **子串表不是分类器**：闸把「幂等放行」里的「等放行」当成只读信号。改为按语义角色拆词表（意图 / 修饰 / 状态），
   状态词不参与判定；用 1017 条历史派单回放验证——新增命中 0、真只读漏拦 0。
2. **维护脚本也是生产代码**：重建分卷的脚本把自己的索引当正文（报 3 卷 115 条，真值 5 卷 155 条），且写盘不备份分卷。
   改为结构判据 + 全量备份 + 写前算术自检。
3. **用例不能绑活数据**：断言生产队列「必须留着某条注释行」，队列一清理就恒红。改为条件断言 + 用夹具证明守卫仍咬人。
4. **例外必须比规则更窄**：队列真实行丢了「服务已停才执行」的守卫，且 `--apply` 在破坏性表里。改为按行加闸 +
   脚本名精确白名单（该行所有 python 调用都须在白名单内，识别不出即拒绝）。闸甚至拒绝了它自己的注释——这正是它没被放宽的证据。
