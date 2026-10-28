# 03 · Gates

A gate is a function from a tool call to `allow` or `block + reason`. This chapter is the part
worth copying even if you copy nothing else.

一道闸就是一个函数：输入一次工具调用，输出 `allow` 或者 `block + reason`。这一章是哪怕你别的都不抄、也值得抄走的部分。

## Anatomy

规则包由四部分组成（代码块不翻译，中文说明在此）：`rules/<name>.json` 只放策略——匹配器、词项、模式、给人看的理由模板；`lib/core.js` 是纯决策逻辑（不依赖运行时即可测试）；`lib/plugin.js` 是薄薄一层 I/O 适配器，读配置、跑策略脚本、上报遥测；`test/<name>.test.mjs` 放成对用例——必须拦、必须放。

```
rules/<name>.json      policy: matchers, terms, mode, human-facing reason templates
lib/core.js            pure decision logic (testable without a runtime)
lib/plugin.js          thin I/O adapter: read config, run the policy script, emit telemetry
test/<name>.test.mjs   paired cases: must-block, must-allow
```

The split matters. If the decision logic touches the filesystem or spawns subprocesses, you cannot
test the decision, only the deployment. Keep pure functions pure; put every side effect in one
adapter file that the tests stub out.

这个拆分的意义在于：一旦决策逻辑碰了文件系统或者去起子进程，你就只能测「部署」，测不了「判断」。纯函数就让它纯着；把所有副作用收进一个适配器文件，由测试把它 stub 掉。

## The block contract

A block must carry four things, or the model (and the human) cannot recover:

一次拦截必须带四样东西，否则模型（以及人）都没法恢复：

1. **what fired** — rule id and the human-readable reason
2. **why** — the matched term/path/state, quoted from the call
3. **how to comply** — the exact command or edit that would pass
4. **the escape hatch, if one exists** — and it must be human-only

1. **命中了什么** —— 规则 id，加上给人看的理由
2. **为什么** —— 命中的词/路径/状态，从这次调用里原样引出来
3. **怎么才能合规** —— 能通过的那条具体命令，或者那处具体改法
4. **逃生口，如果存在的话** —— 而且它必须只有人能开

下面是一段真实的拦截返回（代码块不翻译，中文说明在此）：`blockReason` 里同时写了规则名、缺了什么令牌、以及补令牌的那条命令。

```json
{ "block": true,
  "blockReason": "gate:dispatch-flow — unregistered spawn.\nCreate a token first:\n  dispatch prepare --id <slug> --repo <path> --paths <files>\nthen put `queue=<slug>` on its own line in the task text." }
```

A block without the fix instruction trains the model to retry blindly. A block with an escape
hatch the model can type itself is not a gate.

一条不带修复指令的拦截，只会把模型训练成盲目重试；一条逃生口模型自己能敲出来的拦截，就不叫闸。

## Mode semantics

| mode | behaviour | when to use |
|---|---|---|
| `block` | refuses the call | anything protective |
| `append` | allows, adds a note, records telemetry | observation period for a new rule |
| `off` | disabled | never in production; keep the pack, flip `enabled` |

| 模式 | 行为 | 何时用 |
|---|---|---|
| `block` | 拒绝这次调用 | 任何防护性场景 |
| `append` | 放行，追加一条提示，记录遥测 | 新规则的观察期 |
| `off` | 关闭 | 生产环境永不使用；保留规则包，改翻 `enabled` |

**Unknown mode resolves to `block`.** New rule packs ship in `append` for a week, then flip to
`block` once telemetry shows the false-positive rate is zero.

**认不出的模式一律按 `block` 处理。** 新规则包先以 `append` 上线一周，等遥测显示误拦率为零，再翻成 `block`。

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

1. **中文里的子串匹配。** `等放行` 会被包含在 `幂等放行` 里面。按字节判断包含关系，根本没有「词边界」这个概念。改法：把*意图*模式（结构化：否定 + 动词 + 宾语）与*状态*词分开，并且绝不让一个状态词单独触发拦截。
2. **整篇扫描。** 在长 prompt 里哪儿匹配都算，就把「提到了这个词」当成了「在说这个词」。改法：先给目标段分类；只有目标段判不出来时，才回退到整篇。
3. **作用域过宽的子串。** `skipped=gateway-not-stopped` 是一条正当的守卫串，用了 14 次；一条把它整体封掉的规则，会逼着人把守卫删掉。改法：把谓词限定在它所属的域（探针标记）里，并且断言活下来的守卫仍然存在。
4. **把逃生口自己堵死。** 如果豁免是写进被闸管的文件里的注释中，不要给注释开特例让它通过——那就是闸在咬自己的文档。改法是改写那句注释，让它长得不像一条命令。
5. **策略脚本在负载下超时。** 基于子进程的解析器有超时；超时一到，判决就是「未知」，而未知必须按命中处理。把超时放宽并记日志，否则你会在忙碌的机器上悄悄过度拦截。

## Testing a gate

跑一遍全部成对用例（代码块不翻译，中文说明在此）：任何一组失败都以非零退出。

```
node test/probe-runner.mjs        # exits non-zero if any paired case fails
```

Each pair is `(case, expected)`. In our deployment: 274 assertions across 92 pairs. Two habits
make this useful:

每一组用例都是 `(case, expected)`。在我们的部署里是 92 组共 274 条断言。让这件事真正有用的两个习惯：

- **Mutate the gate, not the test.** Before trusting a paired case, break the gate on purpose and
  confirm the test goes red. A test that cannot fail is documentation.
- **Replay real history.** Keep a corpus of past blocked calls; after changing a matcher, re-run
  the corpus in both directions and diff the verdicts. Any newly-allowed case must be explainable
  as a false positive, and any newly-blocked case needs a reason you would accept in review.

- **改坏闸，而不是改测试。** 在信任一组用例之前，先故意把闸改坏，确认测试变红。一个不可能失败的测试，只是文档。
- **回放真实历史。** 留一批过去被拦住的调用语料；改完匹配器之后，把语料正反两个方向都重跑一遍，并对判决做 diff。任何新放行的用例都必须能解释成误拦，任何新拦截的用例都必须给出你在评审时会接受的理由。

## Drift is not a block

Not every enforcement is a refusal. Structural drift between the ledger, the cards and the panel is
checked by a reconciler that is read-only by default (`--check`) and corrects only under `--apply`.
Its exit codes separate *finding* drift — a result — from *failing to check* — an error; conflating
the two trains people to ignore the check. Four comparisons are the minimum: ledger without a card,
card without a ledger row, ledger terminal while the card is open, card running while the ledger is
not (or nobody is working). Adding a fourth surface without adding the fourth comparison leaves the
system with two truths.

不是所有执行都是「拦」。台账、卡片、面板之间的结构性漂移，由默认只读（`--check`）、只有 `--apply` 才纠正的对账器检查；退出码把「**查到**漂移」（一种结果）和「**查不成**」（一个错误）分开——把两者混为一谈，等于训练所有人忽略这项检查。最少要有四个比较：有台账没卡、有卡没台账行、台账已终态卡还开着、卡在跑而台账不是（或根本没人在干）。加了一面新的展示层却不加对应的比较，系统就会留下两套真相。

## Catalogue

| Group | Packs | Sample refusal |
|---|---|---|
| Dispatch flow | flow, duplicate, route, concurrency, collector-wake, worktree-origin, agent-send | spawn without a single-use token |
| Accounting | ledger, ask, commit-discipline, one-shot | "I will do it later" without a row |
| Safety | dangerous-commands, protected-paths, inline-payload, upstream-sync, write-postcheck | writing a protected path directly |
| Budget | token-budget, model-tier, spawn-concurrency | over-budget call |
| Observation | probe-expectations, registration-observe, startup-verify | gate suite did not run |

| 组 | 规则包 | 示例拒绝 |
|---|---|---|
| 派发流 | flow、duplicate、route、concurrency、collector-wake、worktree-origin、agent-send | 没有一次性令牌就 spawn |
| 记账 | ledger、ask、commit-discipline、one-shot | 没有台账行的「我稍后做」 |
| 安全 | dangerous-commands、protected-paths、inline-payload、upstream-sync、write-postcheck | 直接写受保护路径 |
| 预算 | token-budget、model-tier、spawn-concurrency | 超出预算的调用 |
| 观测 | probe-expectations、registration-observe、startup-verify | 闸测试集没跑 |

## Adding one, in order

1. Write the failure mode as a sentence: *"X happens, and it costs Y."*
2. Decide the enforcement point: entry script (preferred), hook (when the call can come from
   anywhere), or scan (when it is drift, not intent).
3. Implement the pure predicate; add the adapter; emit telemetry with a stable `reason` code.
4. Add paired probes, break the gate, watch them fail.
5. Ship `append`, watch a week of telemetry, then flip to `block`.

1. 把失效模式写成一句话：*「X 会发生，代价是 Y。」*
2. 决定执行点：入口脚本（首选）、hook（当调用的来源无法收束时）、或者巡检（当它是漂移而不是意图时）。
3. 实现纯谓词；加上适配器；用稳定的 `reason` 码上报遥测。
4. 加上成对探针，把闸改坏，看着它们变红。
5. 以 `append` 上线，看一周遥测，然后翻成 `block`。

