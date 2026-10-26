# 03 · Gates

A gate is a function from a tool call to `allow` or `block + reason`. This chapter is the part
worth copying even if you copy nothing else.

## Anatomy

```
rules/<name>.json      policy: matchers, terms, mode, human-facing reason templates
lib/core.js            pure decision logic (testable without a runtime)
lib/plugin.js          thin I/O adapter: read config, run the policy script, emit telemetry
test/<name>.test.mjs   paired cases: must-block, must-allow
```

The split matters. If the decision logic touches the filesystem or spawns subprocesses, you cannot
test the decision, only the deployment. Keep pure functions pure; put every side effect in one
adapter file that the tests stub out.

## The block contract

A block must carry four things, or the model (and the human) cannot recover:

1. **what fired** — rule id and the human-readable reason
2. **why** — the matched term/path/state, quoted from the call
3. **how to comply** — the exact command or edit that would pass
4. **the escape hatch, if one exists** — and it must be human-only

```json
{ "block": true,
  "blockReason": "gate:dispatch-flow — unregistered spawn.\nCreate a token first:\n  dispatch prepare --id <slug> --repo <path> --paths <files>\nthen put `queue=<slug>` on its own line in the task text." }
```

A block without the fix instruction trains the model to retry blindly. A block with an escape
hatch the model can type itself is not a gate.

## Mode semantics

| mode | behaviour | when to use |
|---|---|---|
| `block` | refuses the call | anything protective |
| `append` | allows, adds a note, records telemetry | observation period for a new rule |
| `off` | disabled | never in production; keep the pack, flip `enabled` |

**Unknown mode resolves to `block`.** New rule packs ship in `append` for a week, then flip to
`block` once telemetry shows the false-positive rate is zero.

## Five false positives that will happen to you

1. **Substring matching in CJK.** A term like `等放行` ("awaiting approval") is contained inside
   `幂等放行` ("idempotent release"). Byte-level containment has no notion of word boundaries.
   Fix: separate *intent* patterns (structured: negation + verb + object) from *state* words, and
   never let a state word alone trigger a block.
2. **Whole-text scanning.** Matching terms anywhere in a long prompt flags "mentions the word" as
   "is about the word". Fix: classify the goal section first; fall back to whole-text only when
   the goal section is inconclusive.
3. **Scope substrings.** `skipped=gateway-not-stopped` is a legitimate guard string used 14 times;
   a rule that bans it wholesale forces guards to be deleted. Fix: scope the predicate to the
   domain it belongs to (probe markers), and assert the surviving guards still exist.
4. **Escaping the escape.** If the exemption is written into a comment inside the gated file, do
   not special-case comments to let it pass — that is the gate biting its own documentation.
   Rewrite the comment so it is not shaped like a command.
5. **Policy scripts that time out under load.** A subprocess-based resolver has a timeout; when
   it expires the verdict is "unknown", which must be treated as a hit. Make the timeout generous
   and log it, or you will silently over-block on a busy machine.

## Testing a gate

```
node test/probe-runner.mjs        # exits non-zero if any paired case fails
```

Each pair is `(case, expected)`. In our deployment: 274 assertions across 92 pairs. Two habits
make this useful:

- **Mutate the gate, not the test.** Before trusting a paired case, break the gate on purpose and
  confirm the test goes red. A test that cannot fail is documentation.
- **Replay real history.** Keep a corpus of past blocked calls; after changing a matcher, re-run
  the corpus in both directions and diff the verdicts. Any newly-allowed case must be explainable
  as a false positive, and any newly-blocked case needs a reason you would accept in review.

## Catalogue

| Group | Packs | Sample refusal |
|---|---|---|
| Dispatch flow | flow, duplicate, route, concurrency, collector-wake | spawn without a single-use token |
| Accounting | ledger, ask, commit-discipline, one-shot | "I will do it later" without a row |
| Safety | dangerous-commands, protected-paths, inline-payload, upstream-sync | writing a protected path directly |
| Budget | token-budget, model-tier, spawn-concurrency | over-budget call |
| Observation | probe-expectations, registration-observe, startup-verify | gate suite did not run |

## Adding one, in order

1. Write the failure mode as a sentence: *"X happens, and it costs Y."*
2. Decide the enforcement point: entry script (preferred), hook (when the call can come from
   anywhere), or scan (when it is drift, not intent).
3. Implement the pure predicate; add the adapter; emit telemetry with a stable `reason` code.
4. Add paired probes, break the gate, watch them fail.
5. Ship `append`, watch a week of telemetry, then flip to `block`.

---

## 中文摘要

一道闸 = 一次工具调用 → `allow` 或 `block + 理由`。要点：

**结构**：规则 JSON（策略）+ 纯函数 core（可测）+ 单一 I/O 适配器（可 stub）+ 成对用例。
决策逻辑里出现文件系统或子进程，你就只能测部署、不能测判断。

**拒绝契约**：必须同时给「哪条规则」「命中了什么」「怎么改才过」「有没有人工逃生口」。
只给一半会训练模型盲目重试；逃生口若模型能自己写，那就不叫闸。

**模式**：block / append（观察期）/ off；未知模式一律 block。新规则先在 append 跑一周遥测，假阳为零再升 block。

**五类必然遇到的误拦**：中文子串（「等放行」被「幂等放行」包含）、整篇扫描把「提到」当「关于」、
作用域过宽（`skipped=gateway-not-stopped` 是合法守卫串，一刀切会逼人删守卫）、闸咬自己的注释、
策略脚本在负载下超时被判「未知」而按命中处理。

**测试**：成对用例 + 故意改坏闸确认变红 + 用历史真实调用做双向回放（修后新增放行必须能解释成假阳）。
