# 07 · Adoption

You do not need 21 rule packs. You need three, chosen by cost of the failure, plus one ledger.

## Minimum viable harness (one afternoon)

### Step 1 — pick three gates by damage, not by elegance

Ask: *which three failures have actually cost me time this month?* Typical answers:

| Failure | Gate | Enforcement point |
|---|---|---|
| two agents editing the same files | file-set conflict at claim | entry script |
| destructive command run by accident | dangerous-command check | hook |
| "I will do it later" with no row | promise carrier check | hook on the promise tool |

Write each as one sentence with a cost. If you cannot state the cost, it is not a gate, it is a
preference.

### Step 2 — build the single entry

One script per action, doing register + validate + issue token atomically. Until this exists, every
gate you add can be bypassed by the older path.

### Step 3 — add paired probes before you trust it

For each gate: one input that must block, one that must allow. A runner that exits non-zero on
either failure. Then break the gate and watch the runner fail.

### Step 4 — add one silent scan

Something that can only be detected over time: stale inflight rows, memory budget drift, index/file
mismatch. Silent when clean, one line per finding, plus a liveness timestamp.

### Step 5 — start the decision ledger

Every question that needs a human becomes a row with numbered options and a four-part trade-off
analysis (what it does / upside / downside / risk). Close it with evidence. This is the cheapest
part and the one people skip; without it, the same trade-off is re-argued every month.

## What to skip

| Tempting | Why to skip it early |
|---|---|
| 20+ rule packs on day one | each needs probes and a telemetry review; you will disable them |
| LLM-as-judge for every gate | non-deterministic control is not control; use it for review, not refusal |
| Auto-fixing violations | a gate that edits state can destroy work; report, then let the owner act |
| OS-level sandboxing first | high setup cost, does not stop "two agents on one file" |
| Dashboards before telemetry schema | you will visualise noise and call it insight |

## Migration order that works

1. Take inventory of failures from the last 30 days of logs.
2. Convert the top three into gates with paired probes.
3. Move those actions behind single-entry scripts.
4. Only then start moving rules out of the prompt into packs.
5. Add the evaluation layer once you have ~10 packs; before that it is overhead.

## Signs it is working

- The model quotes a gate's reason string back to you instead of retrying blindly.
- A false positive is fixed by narrowing the predicate, not by adding an exemption.
- Removing a gate requires an argument, and the argument is recorded.
- Your weekly report has fewer findings than last week, and you can explain why.

## Signs it is theatre

- Gates exist that no test exercises.
- Exemptions are granted with a phrase rather than a pattern.
- Nothing has been blocked in a month — either you are perfect, or a gate is dead.
- The rules file is longer than the code that enforces it.

---

## 中文摘要

不需要 21 个规则包。需要的是「按损失挑三道闸 + 一本台账」，一个下午可以搭起来。

五步：① 按本月真实损失挑三件事（同仓并发写 / 误执行破坏性命令 / 「稍后做」没落行）；
② 每个动作收成一个入口脚本，原子完成「登记 + 校验 + 签令牌」；③ 先写成对探针，再故意改坏闸确认会红；
④ 加一条只能靠时间发现的静默巡检（陈旧 inflight / 预算漂移 / 索引与磁盘不一致）；⑤ 开始决策台账，
每条问题带编号选项与四段权衡，收口带证据。

**早期别做**：一上来 20 个规则包、用 LLM 做每道闸的裁判（不确定的控制不算控制）、自动修复违规
（会毁工作，只报不动手）、先上 OS 级沙箱（挡不住同仓双写）、没有遥测 schema 就先做面板。

**算成功的迹象**：模型开始引用闸给的理由而不是盲目重试；误拦靠收窄判据修，而不是加豁免；
删一道闸需要论证且论证被记录。**算表演的迹象**：有闸没人测、豁免靠一句话而不是精确模式、
一个月没拦过任何东西、规则文件比执行它的代码还长。
