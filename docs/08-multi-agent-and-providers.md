# 08 · Multi-agent cross-validation

> One agent's claim is a hypothesis. Independent agents reproducing it is evidence.

This is the centre of the whole design. Everything else — gates, ledgers, budgets — exists so that
several agents can check each other without stepping on each other, and so that a disagreement
survives long enough to be adjudicated instead of being averaged away.

---

## 1. Why a single agent cannot verify itself

An agent asked "is this correct?" will answer with the same weights, the same context, and the same
blind spots that produced the work. Self-review catches typos and catches almost nothing else:

| Failure | Self-review outcome |
|---|---|
| gate matches the wrong substring | the same substring looks right to the same model |
| destructive script mis-parses its input | the parser "looks correct" to its author |
| test bound to live data | the test "passes", which is the problem |
| stale fact in memory | the stale fact is what the review reads |

So verification is not a step inside the agent. It is **a second agent with no access to the first
one's reasoning**, given the same question and the same raw material.

---

## 2. The mechanism, end to end

```
question
  → brief (the only shared artefact; states the question, the evidence paths, the output contract)
  → leg 1..N  (independent, isolated context, read-only, explicitly forbidden to read each other)
  → claims: each leg returns N claims, every one with file:line evidence
  → merge: agreement / partial / contradiction, plus which claim has a reproduction
  → verdict ledger: per-claim decision, with the reason it went that way
  → only reproduced claims become tasks
```

### What makes the legs independent

Independence is a property of the setup, not a promise from the model:

1. **Separate context.** Each leg is a fresh session; none inherits the author's reasoning.
2. **Read-only.** Legs cannot write files, cannot commit, cannot spawn further agents. The only
   output is a report at a path only the parent reads.
3. **No cross-reading.** Stated in the brief and enforced by file placement: leg *i* writes only
   its own report, and is told that reading another leg's output invalidates the run.
4. **Different families.** Where possible the panel is heterogeneous — the same family shares
   most blind spots, so three samples of one model are worth roughly one opinion.
5. **Same brief, same raw material.** Independence without a shared question produces three
   different answers to three different questions.

### What makes the merge honest

The merge is where cross-validation is usually lost. The rule is: **count evidence, never count votes.**

| Situation | Wrong merge | Right merge |
|---|---|---|
| 2 legs say X, 1 says not-X with a reproduction | majority: X | not-X, with the reproduction attached |
| 3 legs say "partly" with different gaps | average into one soft statement | three named gaps |
| all 3 agree, none cite evidence | "confirmed" | unverified; downgrade to hypothesis |
| 1 leg failed to run | "2 of 3 agreed" | the batch failed `min_legs`; rerun |

Disagreement is the product. A panel that always agrees is either homogeneous or not independent,
and the disagreement is exactly the signal that tells a human where to look.

---

## 3. Fail-closed dispatch: you cannot forget to be rigorous

Cross-validation that depends on the operator remembering it is not a mechanism. So:

- **The entry script refuses a single leg for judgement-type work.** Not a lint warning, an exit
  code. There is exactly one way to run an audit: through the panel entry, with `min_legs` ≥ 3.
- **Exceptions are tokens, not opinions.** A legitimate single-leg run needs a one-shot token
  issued by the entry point, bound to one leg index, TTL-bounded. It cannot be self-granted, and
  its use is logged.
- **A dead leg is a failed batch.** If fewer than `min_legs` return a well-formed report, the
  batch fails. "Two out of three" is not a smaller answer, it is a broken one.
- **Malformed reports count as failures.** A leg that returns prose without the required
  `STATUS:` line and metrics block is not silently parsed as "no findings".

---

## 4. Cross-validation applied to the harness itself

The same panel reviews the gates, and that is how the panel pays for itself:

`question: a dispatch was blocked with classification=audit; is the gate wrong?`

| Leg | Finding |
|---|---|
| A | gate is correct; the task text genuinely reads as an audit request |
| B | gate is wrong; the matched term is a substring of an ordinary word |
| C | partly: the classification is wrong, but the same text would also trip a second check |

Merge outcome: the claim with a reproduction wins on that specific point (B: the term `awaiting
approval` sits inside `idempotent release`), the two partial findings become separate follow-ups,
and the "gate is correct" leg is recorded with its reasoning rather than discarded.

Result: the predicate was rewritten from substring matching to semantic layers, then **replayed**
against 1,017 historical dispatch events — 0 new blocks, 0 missed true positives, 25 signals
explained. That replay is the second half of cross-validation: the panel proposes, the corpus decides.

Without an independent panel this fix would have been "add the word to the ignore list", which is
what a single agent optimises for: the smallest edit that stops the error message.

---

## 5. Providers are plumbing for independence

Heterogeneity is not about brand loyalty; it is the cheapest way to buy independence. Hence a
**routing table by role**, not by vendor:

```yaml
tasks:
  mechanical:      { mode: single,           legs: [fast-coder],  min_legs: 1 }
  ui_frontend:     { mode: produce_verify,   producer: fast-coder, verifiers: [reviewer], min_legs: 2 }
  backend_fix:     { mode: single,           legs: [fast-coder],  min_legs: 1 }
  audit_review:    { mode: cross_validate,   legs: [analyst, reviewer, second-opinion], min_legs: 3 }
  design_planning: { mode: cross_validate,   legs: [analyst, reviewer, second-opinion], min_legs: 3 }
```

- **roles** (`fast-coder`, `reviewer`, `analyst`, `second-opinion`) map to whatever provider wins on
  that axis this month; swapping a provider is a table edit;
- **model resolution is verified at spawn**; an unresolvable model fails the dispatch rather than
  being silently substituted;
- **the producer never judges its own output**: `produce_verify` pairs a writer with a different
  reviewer, and the reviewer's veto is a task, not a comment.

### Cost discipline (panels are expensive, unattended is not)

| Control | Value |
|---|---|
| budget checked before dispatch | tokens and wall clock, not after the fact |
| wall-clock cap | breach = hard exit, no partial merge |
| cheap tier by default | mechanical work never opens a panel |
| panels only for judgement | audit, design, security-adjacent review |
| parent merges | the merge runs in one context, not a fourth opinion |

---

## 6. Several agents, one ledger

Cross-validation scales only if the agents cannot damage each other while disagreeing. That is the
orchestration layer, compressed:

| Rule | Failure prevented |
|---|---|
| one task = one declared file set | two writers, one file, conflicting commits |
| claim-time conflict check (not registration-time) | "queued" promises that never run |
| concurrency caps by kind (writers vs reviewers) | host saturation, rate limits |
| worktrees for same-repo parallelism | one child's dirty tree blocking another |
| close requires evidence (commit sha / report path) | "done" that cannot be verified |
| one-shot dispatch tokens, never revived | double dispatch after a failed spawn |
| fallback wake registered at spawn | dropped completion events, silently stalled parents |
| stale detection reports, never cancels | killing work that was merely quiet |

---

## 7. Anti-patterns (each one looks like validation)

1. **Judge = producer.** Same session reviews its own work. Measures nothing.
2. **Legs read each other.** The second leg anchors on the first; you get one opinion in three fonts.
3. **Majority vote.** Turns a reproduction into a minority report.
4. **Averaging three "partly".** Produces a confident statement nobody has verified.
5. **"Three agents agreed" with no evidence column.** Consensus is cheap; reproduction is not.
6. **Silent fallback to one leg** when a provider is unavailable. The most dangerous case: it looks
   like a normal result.
7. **Panel for everything.** Cost explodes, legs get shallow, and you stop reading them.

---

## 8. Adopting it in a day

1. Pick one question you got wrong recently. Write it as a brief with the raw evidence paths.
2. Run two independent read-only legs on different families. No shared context, no cross-reading.
3. Merge by evidence, and write down where they disagreed — even if you were sure.
4. Turn the reproduced claims into tasks; leave the rest as hypotheses in the ledger.
5. Only then wire the entry script so a single leg is refused for judgement work.

---

## 中文摘要

**一句话**：一个 agent 的结论只是假设，**多个 agent 各自独立复现**才是证据。这一章讲的就是这个机制，
也是整套体系的中心——闸、台账、预算都是为了让多个 agent 能互检而不互相踩踏，并让分歧活到被裁决的那一刻。

**为什么必须独立**：同一个模型自审，用的是产生错误的同一套权重与上下文，所以它只会抓错别字。独立要靠设置保证，
不能靠模型自觉：独立上下文、只读、禁止互读彼此报告、尽量异构（同族模型共享盲点）、同一份 brief 与同一批原始材料。

**合并纪律：数证据，不数票。** 两票赞成一票反对但后者带复现 → 听复现；三票「部分成立」→ 写清三个缺口而不是
揉成一句模糊话；三票一致但都没证据 → 降级为假设；一腿挂掉 → 判批次失败，而不是「三中二」。**分歧就是产品**。

**fail-closed 派单**：判断题单腿派单由入口脚本直接拒（退出码，非提示）；例外只能由入口签发一次性 leg 令牌；
腿数不足即失败；格式不合契约的报告按失败计，绝不当作「无发现」。

**真实案例**：三腿评审一次闸误拦——一腿说闸对、一腿说撞词、一腿说部分成立。带复现的那条胜出，闸从子串匹配改为
语义分层，再用 1017 条历史派单回放做裁判（新增命中 0、真阳漏拦 0）。**评审团提议，语料判决**——这是互检的后半段。

**provider 是独立性管道**：路由表按角色（fast-coder / reviewer / analyst / second-opinion）而非牌子；模型解析不出
就失败不静默替换；产出者永不评审自己的输出。成本纪律：预算与墙钟在派单前判，机械活不开评审团，合并在父会话一个上下文完成。

**七个反模式**（都长得像验证）：自查、互相看报告、多数表决、三个「部分成立」取平均、无证据的一致性、provider 挂了
静默退回单腿、什么都开评审团。
