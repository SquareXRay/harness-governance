# 08 · Multi-agent cross-validation

> One agent's claim is a hypothesis. Independent agents reproducing it is evidence.

> 一个 agent 的结论只是假设；多个 agent 各自独立复现出来，才是证据。

This is the centre of the whole design. Everything else — gates, ledgers, budgets — exists so that
several agents can check each other without stepping on each other, and so that a disagreement
survives long enough to be adjudicated instead of being averaged away.

这是整套设计的中心。其余一切——闸、台账、预算——之所以存在，都是为了：让多个 agent 能互检而不互相踩踏，并让分歧活到被裁决的那一刻，而不是被平均抹平。

---

## 1. Why a single agent cannot verify itself

An agent asked "is this correct?" will answer with the same weights, the same context, and the same
blind spots that produced the work. Self-review catches typos and catches almost nothing else:

一个被问「这做得对吗」的 agent，会用产生这份工作的同一批权重、同一份上下文、同一组盲点来回答。自审抓得到错别字，几乎抓不到别的：

| Failure | Self-review outcome |
|---|---|
| gate matches the wrong substring | the same substring looks right to the same model |
| destructive script mis-parses its input | the parser "looks correct" to its author |
| test bound to live data | the test "passes", which is the problem |
| stale fact in memory | the stale fact is what the review reads |

| 故障 | 自审的结果 |
|---|---|
| 闸匹配到了错误的子串 | 同一个子串在同一个模型眼里看着是对的 |
| 破坏性脚本解析错了输入 | 解析器在作者眼里「看着没问题」 |
| 测试绑在活数据上 | 测试「通过」了，而这正是问题所在 |
| 记忆里有一条过期事实 | 复核读到的，就是那条过期事实 |

So verification is not a step inside the agent. It is **a second agent with no access to the first
one's reasoning**, given the same question and the same raw material.

所以验证不是发生在 agent 内部的一个步骤。它是**第二个 agent，拿不到第一个的推理过程**，被给到同一个问题和同一批原始材料。

---

## 2. The mechanism, end to end

整个机制端到端长这样（代码块不翻译，中文说明在此）：一个问题 → 一份 brief（唯一共享物，写清问题、证据路径、输出契约）→ 第 1..N 条腿（独立、上下文隔离、只读、明确禁止互读）→ 每条腿交回若干条主张，每条都带 file:line 证据 → 合并（一致 / 部分 / 矛盾，并标出哪条主张带复现）→ 判决台账（逐条给决定和理由）→ 只有被复现的主张才变成任务。

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

独立性是设置的性质，不是模型给的承诺：

1. **Separate context.** Each leg is a fresh session; none inherits the author's reasoning.
2. **Read-only.** Legs cannot write files, cannot commit, cannot spawn further agents. The only
   output is a report at a path only the parent reads.
3. **No cross-reading.** Stated in the brief and enforced by file placement: leg *i* writes only
   its own report, and is told that reading another leg's output invalidates the run.
4. **Different families.** Where possible the panel is heterogeneous — the same family shares
   most blind spots, so three samples of one model are worth roughly one opinion.
5. **Same brief, same raw material.** Independence without a shared question produces three
   different answers to three different questions.

1. **独立上下文。** 每条腿都是全新会话，没有一条继承作者的推理。
2. **只读。** 腿不能写文件、不能提交、不能再派生 agent。唯一输出是一份报告，路径只有父会话会读。
3. **禁止互读。** 在 brief 里写明，并靠文件放置来落实：第 *i* 条腿只写自己的报告，并且被告知——读别的腿的输出会让这次运行作废。
4. **不同家族。** 尽可能让评审团异构——同一家族共享大部分盲点，所以同一个模型的三个样本，只值大约一个意见。
5. **同一份 brief、同一批原始材料。** 没有共同问题的独立性，只会产出三个问题、三个答案。

### What makes the merge honest

The merge is where cross-validation is usually lost. The rule is: **count evidence, never count votes.**

合并这一步，是交叉验证最常丢掉的地方。规则是：**数证据，绝不数票。**

| Situation | Wrong merge | Right merge |
|---|---|---|
| 2 legs say X, 1 says not-X with a reproduction | majority: X | not-X, with the reproduction attached |
| 3 legs say "partly" with different gaps | average into one soft statement | three named gaps |
| all 3 agree, none cite evidence | "confirmed" | unverified; downgrade to hypothesis |
| 1 leg failed to run | "2 of 3 agreed" | the batch failed `min_legs`; rerun |

| 情形 | 错误的合并 | 正确的合并 |
|---|---|---|
| 2 条腿说 X，1 条腿说非 X 且带复现 | 少数服从多数：X | 非 X，附上那份复现 |
| 3 条腿都说「部分成立」，但缺口各不相同 | 平均成一句软话 | 三个具名的缺口 |
| 3 条腿一致，但都没引用证据 | 「已确认」 | 未验证；降级为假设 |
| 1 条腿没跑起来 | 「三中二」 | 这批失败于 `min_legs`；重跑 |

Disagreement is the product. A panel that always agrees is either homogeneous or not independent,
and the disagreement is exactly the signal that tells a human where to look.

分歧就是产品。一个永远一致的评审团，要么是同质的，要么根本不独立——而分歧恰恰是告诉人「该去哪儿看」的那个信号。

---

## 3. Fail-closed dispatch: you cannot forget to be rigorous

Cross-validation that depends on the operator remembering it is not a mechanism. So:

交叉验证如果靠执行者自己记得去做，那它就不是机制。所以：

- **The entry script refuses a single leg for judgement-type work.** Not a lint warning, an exit
  code. There is exactly one way to run an audit: through the panel entry, with `min_legs` ≥ 3.
- **Exceptions are tokens, not opinions.** A legitimate single-leg run needs a one-shot token
  issued by the entry point, bound to one leg index, TTL-bounded. It cannot be self-granted, and
  its use is logged.
- **A dead leg is a failed batch.** If fewer than `min_legs` return a well-formed report, the
  batch fails. "Two out of three" is not a smaller answer, it is a broken one.
- **Malformed reports count as failures.** A leg that returns prose without the required
  `STATUS:` line and metrics block is not silently parsed as "no findings".

- **入口脚本对判断题直接拒绝单腿。** 不是一条 lint 警告，是一个退出码。跑审计只有一条路：走评审团入口，`min_legs` ≥ 3。
- **例外是令牌，不是意见。** 一次正当的单腿运行，需要由入口签发的一次性令牌：绑定到某一个腿序号，带 TTL。它不能自授，每次使用都记日志。
- **腿死了 = 批次失败。** 如果返回合规报告的腿少于 `min_legs`，这批就算失败。「三中二」不是更小的答案，而是坏掉的答案。
- **格式不合规的报告也算失败。** 一条只返回散文、缺少必需 `STATUS:` 行和指标块的腿，不会被静默解析成「没有发现」。

---

## 4. Cross-validation applied to the harness itself

The same panel reviews the gates, and that is how the panel pays for itself:

同一个评审团也会评审闸，而这正是评审团把自己挣回来的方式：

`question: a dispatch was blocked with classification=audit; is the gate wrong?`

具体问题是：一次派发被以 `classification=audit` 拦住；是闸错了吗？

| Leg | Finding |
|---|---|
| A | gate is correct; the task text genuinely reads as an audit request |
| B | gate is wrong; the matched term is a substring of an ordinary word |
| C | partly: the classification is wrong, but the same text would also trip a second check |

| 腿 | 发现 |
|---|---|
| A | 闸是对的；这段任务文本确实读起来像一个审计请求 |
| B | 闸错了；命中的词只是一个普通词的子串 |
| C | 部分成立：分类错了，但同一段文本也会触发第二道检查 |

Merge outcome: the claim with a reproduction wins on that specific point (B: the term `awaiting
approval` sits inside `idempotent release`), the two partial findings become separate follow-ups,
and the "gate is correct" leg is recorded with its reasoning rather than discarded.

合并结果：带复现的那条主张在那个具体点上胜出（B：`awaiting approval` 就嵌在 `idempotent release` 里），两条部分成立的发现变成各自独立的后续项，而「闸是对的」那条腿连同它的推理被记录下来，不是被丢掉。

Result: the predicate was rewritten from substring matching to semantic layers, then **replayed**
against 1,017 historical dispatch events — 0 new blocks, 0 missed true positives, 25 signals
explained. That replay is the second half of cross-validation: the panel proposes, the corpus decides.

结果：谓词从子串匹配重写成语义分层，然后对着 1,017 条历史派发事件**回放**——新增拦截 0，真阳漏拦 0，25 个信号得到解释。那次回放就是交叉验证的后半段：**评审团提议，语料判决。**

Without an independent panel this fix would have been "add the word to the ignore list", which is
what a single agent optimises for: the smallest edit that stops the error message.

如果没有一个独立评审团，这个修法会变成「把那个词加进忽略名单」——而这正是单个 agent 会优化的方向：让报错消失的最小改动。

---

## 5. Providers are plumbing for independence

Heterogeneity is not about brand loyalty; it is the cheapest way to buy independence. Hence a
**routing table by role**, not by vendor:

异构不是品牌忠诚，它是买到独立性的最便宜方式。所以我们有一张**按角色排的路由表**，而不是按厂商排的。下面这张 YAML 就是它（代码块不翻译，中文说明在此）：机械活单腿，前端是产出后验证，审计与设计走三腿交叉验证，每条任务都带 `min_legs`。

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

- **角色**（`fast-coder`、`reviewer`、`analyst`、`second-opinion`）映射到这个月在该轴上最强的那个 provider；换一个 provider 就是改一行表；
- **模型解析在 spawn 时校验**；解析不出的模型会让这次派发失败，而不是被静默替换；
- **产出者永不评判自己的输出**：`produce_verify` 把一个写手配给一个不同的评审者，而评审者的否决是一条任务，不是一条评论。

### Cost discipline (panels are expensive, unattended is not)

| Control | Value |
|---|---|
| budget checked before dispatch | tokens and wall clock, not after the fact |
| wall-clock cap | breach = hard exit, no partial merge |
| cheap tier by default | mechanical work never opens a panel |
| panels only for judgement | audit, design, security-adjacent review |
| parent merges | the merge runs in one context, not a fourth opinion |

| 控制 | 取值 |
|---|---|
| 派发前先查预算 | token 与墙钟，而不是事后 |
| 墙钟上限 | 超了就是硬退出，不做部分合并 |
| 默认用便宜档 | 机械工作永远不开评审团 |
| 评审团只用于判断题 | 审计、设计、安全相关评审 |
| 由父会话合并 | 合并在一个上下文里跑，不是再要第四个意见 |

---

## 6. Several agents, one ledger

Cross-validation scales only if the agents cannot damage each other while disagreeing. That is the
orchestration layer, compressed:

交叉验证只有在 agent 们意见不合时也不会互相破坏的前提下才能扩展。那就是编排层，压缩成一张表：

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

| 规则 | 挡住什么故障 |
|---|---|
| 一个任务 = 一批声明的文件集 | 两个写者、一个文件、互相冲突的提交 |
| 冲突检查放在 claim 时（不是登记时） | 「排队中」的承诺永远不跑 |
| 按种类设并发上限（写手 vs 评审者） | 主机被打满、触发限流 |
| 同仓并行用 worktree | 一个子会话的脏树挡住另一个 |
| 收口要求证据（commit sha / 报告路径） | 无法验证的「已完成」 |
| 一次性派发令牌，永不复活 | spawn 失败之后的重复派发 |
| spawn 时登记兜底唤醒 | 丢失的完成事件、静默卡住的父会话 |
| 陈旧检测只报告、永不 cancel | 把只是安静了一会儿的工作杀掉 |

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

1. **裁判 = 产出者。** 同一个会话评审自己的工作。什么都没测到。
2. **腿之间互相读。** 第二条腿锚在第一条上；你得到的是一个意见的三种字体。
3. **少数服从多数。** 把一份复现变成了少数派报告。
4. **把三个「部分成立」取平均。** 产出一句没人验证过的自信结论。
5. **「三个 agent 都同意了」，却没有证据那一列。** 共识很便宜，复现不便宜。
6. **provider 不可用时静默退回单腿。** 这是最危险的一种：它看起来像一个正常结果。
7. **什么都开评审团。** 成本爆掉、腿变浅、你最后不读了。

---

## 8. Adopting it in a day

1. Pick one question you got wrong recently. Write it as a brief with the raw evidence paths.
2. Run two independent read-only legs on different families. No shared context, no cross-reading.
3. Merge by evidence, and write down where they disagreed — even if you were sure.
4. Turn the reproduced claims into tasks; leave the rest as hypotheses in the ledger.
5. Only then wire the entry script so a single leg is refused for judgement work.

1. 挑一个你最近答错的问题。把它写成一份 brief，附上原始证据路径。
2. 在不同家族上跑两条独立的只读腿。不共享上下文，不互读。
3. 按证据合并，并把它们分歧的地方写下来——哪怕你当时很确定。
4. 把被复现的主张变成任务；其余的作为假设留在台账里。
5. 到这时才去接入口脚本，让判断题的单腿派发被拒绝。

---
