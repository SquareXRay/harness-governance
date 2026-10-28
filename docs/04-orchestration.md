# 04 · Orchestration

Delegation is where an unattended agent does the most damage. Two sub-agents editing one file set,
a task nobody collects, a "queued" item that exists only in prose.

委派，正是无人值守 agent 造成最大破坏的地方：两个子 agent 改同一批文件、一个没人收的任务、一条只存在于散文里的「排队中」。

## The ledger is the truth

Every delegated task is a row with a canonical state vocabulary:

每一条被委派的任务就是一行，状态词表只有一份契约（代码块不翻译，中文说明在此）：写入只写规范名，旧名（planned、queued、inflight、unsettled）读出时归一；合法迁移与守卫集中在同一张表里，运行期先校验契约再用。

```
backlog | todo | ready | running | review | blocked | done | cancelled
```

Two properties matter more than the exact drawing: the vocabulary is **one** table shared by the
ledger, the cards and the panel — a status the contract does not know is an error, not a new state;
and `running → done` is legal only as an explicit acceptance, never as a declaration.

比具体画法更重要的是两条性质：词表是**一份**表，台账、卡片、面板共用——契约不认识的状态是错误，不是新状态；`running → done` 只能走显式验收，不能靠一句「做完了」把行收口。

| Field | Why it exists |
|---|---|
| `id` | the only handle; every later command takes it |
| `repo` + `paths` | what it will touch; the basis for conflict detection |
| `owner` | which session may close it |
| `expect` | the artefact that proves it worked |
| `evidence` | commit sha / report path, written on close |
| `sessionKey` | the run that owns it right now |

| 字段 | 为什么存在 |
|---|---|
| `id` | 唯一句柄；之后每条命令都拿它定位 |
| `repo` + `paths` | 它会碰什么；冲突检测的依据 |
| `owner` | 哪个会话有权收口 |
| `expect` | 证明它干成了的那个产物 |
| `evidence` | commit sha / 报告路径，收口时写入 |
| `sessionKey` | 当下拥有它的那次运行 |

A row without `expect` cannot be verified; a row without `evidence` cannot be audited. Both are
required at close time, not at creation time.

没有 `expect` 的行无法验收，没有 `evidence` 的行无法审计。这两项都在收口时要求，而不是在创建时要求。

## Single-use tokens

Registration and permission are the same action:

登记与许可，就是同一个动作（代码块不翻译，中文说明在此）：`prepare` 原子地一次完成「登记 + 占位 + 签发一次性令牌」，令牌带 TTL，默认 15 分钟；spawn 时由闸核销令牌并翻成 running。

```
dispatch prepare --id <slug> --repo <path> --paths <files> --expect <artefact>
  → registers the row, claims it, issues a one-shot token (TTL, default 15 min)
spawn(task containing  queue=<slug>  on its own line)
  → the gate consumes the token, flips the row to running, strips the line
```

Properties worth copying:

值得照抄的几个性质：

- **the token is spent by the spawn**, so a crashed spawn needs a fresh one (`reissue`), and the
  ledger records the difference rather than silently retrying;
- **the row cannot be revived** once its token was used; a new attempt gets a new id;
- **the gate is the consumer**, so there is no code path that spawns without an issued token.

## Start is claim-first

`dispatch start` asks the ledger for the claim token first. If the claim is refused — another writer
holds the file set, a dependency is unmet, the row is not runnable — the start **downgrades to a
plain `move` and records `claimFallback`** on the row. The work still starts, but the ledger shows
it did not go through the claim path. Starting silently without a claim is exactly how you get cards
that look running and stay running forever.

- **令牌是被 spawn 消耗掉的**，所以崩掉的 spawn 需要新签一个（`reissue`），台账会把这次差异记下来，而不是静默重试；
- **令牌用过之后，那一行不能复活**；新的尝试拿一个新的 id；
- **闸就是那个消费者**，所以不存在任何一条「没有签发令牌却还能 spawn」的代码路径。

## 起跑走 claim 优先

`dispatch start` 起跑时先向台账要 claim。claim 被拒（别的写者占着同一批文件、依赖没满足、条目不可跑）时，起跑会**降级成一次普通 `move`，并在条目上记 `claimFallback`**。活照样开，但台账上留了「没走 claim」的痕迹。不声不响地无 claim 起跑，正是「看起来在跑、其实一直挂着」的卡的来源。

## Conflict detection

Two comparisons decide whether a new task may start:

两个比较，决定一条新任务能不能开工：

1. **file-set intersection** — same repository and overlapping declared paths, or either side
   declaring nothing (unknown counts as conflicting);
2. **same expected artefact** — two tasks claiming to produce the same report path will overwrite
   each other.

1. **文件集相交** —— 同一个仓、声明的路径有重叠，或者任一方什么都没声明（未知按冲突算）；
2. **期望产物相同** —— 两条任务都声称要产出同一个报告路径，就会互相覆盖。

The check runs at *claim* time, not at *registration* time. That asymmetry is deliberate: you can
always record intent, you cannot always start work.

这项检查发生在 **claim** 时，而不是在 **登记** 时。这个不对称是故意的：你永远可以记录意图，但你不可能永远开工。下面三条命令的分工（代码块不翻译，中文说明在此）：`add` 永远成功——依赖没满足就落 `todo`，而不是被拒；`claim` 才做判断；`promote-dependents` 在前单完成时把依赖放行。

```
add  --paths <set> --after <other-id>     # always succeeds → todo when a dep is unmet
claim                                     # decides: free, or blockedReason=waiting: <id>
promote-dependents                        # on completion: clear deps, make runnable
```

Two edge kinds, one predicate: `blockedBy[]` (hard dependency) and `after[]` (soft, "queue behind")
both resolve through a single function. If you implement them separately they will drift, and the
panel will show one while the scheduler honours the other.

两种边、一套谓词：`blockedBy[]`（硬依赖）与 `after[]`（软依赖，「排在某单后面」）都走同一个函数解析。如果分开实现，它们必然漂移——面板显示一种，调度遵守另一种。

## Owner gate and close contract

`close` requires (a) the caller is the row owner or an explicitly authorised delegate, (b) a
non-empty `evidence`. The gate refuses with exit code 5 rather than writing a partial close — a
row closed without evidence is worse than an open row, because it looks finished.

`close` 要求：(a) 调用者是该行的 owner，或者一个被显式授权的代理；(b) `evidence` 非空。闸会以退出码 5 拒绝，而不是写一个半成品收口——没有证据的收口比一条开着的行更糟，因为它看起来已经完成了。

## Activity evidence, not self-reports

A sub-agent's own "still running" is not evidence. Three machine signals decide whether a row is
alive, all read-only: the newest file mtime inside the declared file set; the count of
`pytest`/`chromium`/`node` processes whose working directory is that repository; and the freshest
tool-call timestamp for that session (falling back to the transcript mtime when telemetry has aged
out).

**A row is stalled only when all three are silent for more than 20 minutes.** Any one signal inside
the window means it is working. Read-only legs — no writes, no test runs — were flagged stalled
every time before the third signal existed.

When stalled, the reconciler writes the heartbeat-derived decision as `blocked` with a rule code and
**does not cancel**. Cancelling on suspicion is how you lose work.

子单自己说「还在跑」，不是证据。三个机器信号决定一行是否活着，全部只读：声明文件集里最新的文件 mtime；工作目录是该仓的 `pytest`/`chromium`/`node` 进程数；以及该会话最新的工具调用时间戳（遥测过期时回落到会话记录的 mtime）。

**三项全部静默超过 20 分钟，才判卡死。** 窗口内任一信号在动，就算在干活。只读腿（不写盘、不起测试）在第三条证据出现之前，每次都会被误判卡死。

判死之后，对账器把「心跳驱动」的结论落成 `blocked` 并带规则码，**不 cancel**。凭猜疑 cancel，正是弄丢别人工作的方式。

## Reaping what cannot finish

A parent row whose legs are gone, or whose own carrier session has gone silent, still has to be
reaped — but the reaper resolves its legs from the **full ledger**, not from a filtered "active"
view, and treats already-purged legs as an empty set (`legs = 0`) instead of as unknown.
Fail-closed on "legs not found" leaves dead parents unreapable forever; the rule is *when the ledger
proves the legs never existed, reap as usual*, recorded under a rule code so a human can unblock it
by hand.

父条目回收：腿已经不在了、或父自己所在的载体会话已静默的父行，仍然必须被回收——但回收器要从**全量台账**解析它的腿，而不是从过滤过的「活跃」视图；已经 purge 的腿按空集（`legs = 0`）处理，而不是按未知。对「查不到腿」fail-closed，会让死父永远收不掉；判据是**台账能证明腿从未存在 → 照常回收**，并留规则码，方便人工 unblock 回滚。

## Three surfaces, one authority

The ledger is the truth; cards and the panel are projections. A reconciler — read-only by default,
writing only under `--apply` — checks four drifts:

| Check | Drift | Under `--apply` |
|---|---|---|
| D1 | ledger `ready`/`running` with no card | create the card (idempotent: title prefix `[<entryId>]`) |
| D2 | card bound to no ledger row through `cardId` | report only, counted as unowned — never hidden, never auto-deleted |
| D3 | ledger terminal (`done`/`cancelled`) but the card still open | close the card and archive it |
| D4 | card `running` while the ledger is not, or no claim, or all activity silent >20 min | demote to `ready` + diagnostic |

台账是真相，卡片与面板只是投影。对账器默认只读、只有 `--apply` 才写，查四类漂移：D1 台账 `ready`/`running` 却没卡 → 补建卡（按标题前缀 `[<entryId>]` 幂等查重）；D2 卡没有通过 `cardId` 绑到任何台账行 → 只报告、计入 unowned，绝不隐藏、绝不自动删；D3 台账已终态（`done`/`cancelled`）但卡还开着 → 收口并归档；D4 卡在 `running` 而台账不是、没有 claim、或活性三项静默 >20 分钟 → 退回 `ready` 并留诊断。

Exit codes carry the meaning: `0` no drift under `--check`, or every correction applied; `1` drift
found / some correction failed; `2` fail-closed — the state contract is missing or drifted, the
ledger is unreadable, an unknown state appears, or the card backend cannot be read.

退出码带语义：`0` = `--check` 无漂移，或 `--apply` 全部纠正成功；`1` = 查到漂移 / 有纠正失败；`2` = 失败朝严——状态契约缺失或漂移、台账读不了、出现未知状态、卡片后端读不到。

Card/ledger sync follows the same asymmetry: a drift check that **finds** drift exits `0` (a finding,
not a failure) while the check **erroring** exits `2`; and one pass reads the cards from a **single
snapshot**, because per-card calls accumulate until the whole check times out. When the card backend
holds no job container, the sync no longer skips silently — an unresolved terminal row is handed to
the reconciler instead of vanishing.

卡/台账同步是同一个不对称：**查到**漂移交 `0`（发现不是失败），检查**出错**才交 `2`；一次同步从**单次快照**取卡，因为逐卡调用会累积到整轮超时。卡片后端没有 job 容器时，同步不再静默跳过——没收口的终态行交给对账器，而不是凭空消失。

## Caps

Writing sub-agents run at most three at a time in one deployment; reviewers and read-only legs are
counted separately, and the cap is enforced before the spawn, not by watching the host afterwards. A
load ceiling sits in front of it: when load is already above the ceiling, no new write task starts.

同一个部署里，写码子会话最多三个同时跑；评审与只读腿分开计数，上限在 spawn 之前就拦住，而不是事后看机器负载。它前面还有一道负载闸：当前 load 已经超过上限时，不新开任何写码任务。

## What goes wrong without this

| Symptom | Underlying absence |
|---|---|
| "I thought it was in the queue" | no row; the promise lived in chat |
| two commits touching the same file | no file-set comparison at claim |
| a task that ran but was never reported | no collector / no evidence field |
| the same task dispatched twice | tokens reusable or missing |
| a card that says "running" for hours with nobody working | card state trusted over ledger activity |
| a finished task that is still open on the board | no comparison between ledger terminal state and card state |

| 症状 | 底下缺的是什么 |
|---|---|
| 「我以为它排上队了」 | 没有行；承诺只活在聊天里 |
| 两个提交碰了同一个文件 | claim 时没有做文件集比较 |
| 任务跑了，却从没被报告 | 没有 collector / 没有 evidence 字段 |
| 同一个任务被派了两次 | 令牌可重用，或者根本没令牌 |
| 一张卡挂了几个小时「执行中」，其实没人在干 | 信卡上的状态，不信台账的活性 |
| 已经做完的任务，看板上还开着 | 台账终态与卡片状态之间没有比较 |

