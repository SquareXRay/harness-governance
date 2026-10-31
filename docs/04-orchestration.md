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
row closed without evidence is worse than an open row, because it looks finished. Ownership falls
back to the row's own `owner` field, so a re-issued or recovered run still has an identity to be
checked against.

**Path evidence is one predicate.** When `expect` names a path, four candidate bases are tried in a
fixed order — absolute, repository, workspace, current directory — and the answer is "delivered"
only for a non-empty file or a non-empty directory. Missing, empty, `tbd`, a bare commit sha, and a
*report shell* all fail: a file whose first lines still say "pending" is a skeleton written at
dispatch time, not a result. The same predicate serves the settle scan and the close gate, so a
task cannot settle under one rule and be rejected under another.

`close` 要求：(a) 调用者是该行的 owner，或者一个被显式授权的代理；(b) `evidence` 非空。闸会以退出码 5 拒绝，而不是写一个半成品收口——没有证据的收口比一条开着的行更糟，因为它看起来已经完成了。归属判定会回落到条目自己的 `owner` 字段，所以重签或被恢复的运行仍然有一个可核的身份。

**路径证据只有一个判据。** `expect` 指向路径时，按固定顺序试四个基准——绝对路径、仓库、工作区、当前目录——只有非空文件或非空目录才算「已交付」。缺失、空文件、`tbd`、单独一个 commit sha、以及**报告壳**全部不通过：开头还写着 pending 的文件，是派单时先占位的骨架，不是结果。同一套判据同时服务销账扫描与收口闸，所以任务不会在一处通过、在另一处被拒。

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

A separate guard enforces the time limit, and it too refuses to kill on age alone: a child run past
its 20-minute soft limit is cancelled only if it is *idle* (no writes, no processes) or if its
context has crossed the rollover threshold; an absolute hard limit exists above it. The guard runs
as a plain command, not as a model turn, so enforcing it costs nothing.

判死之后，对账器把结论落成 `blocked` 并带规则码，**不 cancel**。凭猜疑 cancel，正是弄丢别人工作的方式。

时限制由另一道守卫执行，它同样不肯只凭年龄就杀：超过 20 分钟软限的子单，只有在**闲置**（不写盘、无进程）或上下文已越过换卷阈值时才被取消；上面另有一道硬上限。这道守卫以普通命令运行，不占模型回合，所以执行它的成本是零。

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
writing only under `--apply` — now checks nine drifts, because each surface added a new way two
views can disagree:

| Check | Drift | Under `--apply` |
|---|---|---|
| D1 | ledger `ready`/`running` with no card | create the card (idempotent: title prefix `[<entryId>]`) |
| D2 | card bound to no ledger row through `cardId` | report only, counted as unowned — never hidden, never auto-deleted |
| D3 | ledger terminal (`done`/`cancelled`) but the card still open | close the card and archive it |
| D4 | card `running` while the ledger is not, or no claim, or all activity silent >20 min | demote to `ready` + diagnostic |
| D5 | ledger open (`running`/`blocked`) but the card is done, or the `cardId` is no longer on the board | report only by default; two explicit flags close the row or reopen the card |
| D6 | ledger terminal, card in no terminal state, not archived | archive the card (does not require a claim) |
| D7 | ledger `blocked` while its carrier is still alive (a false block) | after a second hit ≥60 s later: `blocked → ready → running`; it corrects, never closes |
| D8 | a retired ledger mirror still present on disk | archive, then delete; content is not compared, and a symlink to the canonical file is not drift |
| D9 | an `automation` entry still carrying a `cardId` | report only; the fix is the release-cards command |

Two of these carry a discipline worth stating. **D4 does not judge on structure alone**: a missing
claim or a ledger-not-running is recorded as a diagnostic and the card is left alone; only silence
across all three activity signals justifies demotion, and an unreadable activity signal
(`activity-unknown`) demotes nothing. **D7 requires confirmation**: one probe hit only records a
candidate (visible to `--check`); it takes a second hit at least a minute later, or an updated
heartbeat, before `--apply` touches the row — and `automation` carriers are exempt, because their
liveness is the scheduler's last run status, not a resident heartbeat.

The exit code has four tiers, not three. `0` converged (`--check` clean, or `--apply` followed by a
re-check of D1/D3/D4 all zero); `1` drift found in read-only mode; `2` a real error — contract
missing or drifted, ledger unreadable, unknown state, card backend unreadable, apply errors, or
leftovers after `--apply`, which are written to a to-do file and still exit `2` so the scan shows
red; `3` partial — the wall clock or the per-run write slice ran out. `3` exists because the outer
scheduler treats any non-zero exit as a failure and disables a job after ten in a row: "did not
finish" is not "found drift". A write conflict hit twice in a row on the same item is `deferred`,
counted in neither leftovers nor the exit code.

台账是真相，卡片与面板只是投影。对账器默认只读、只有 `--apply` 才写；它现在查九类漂移，因为每增加一个展示面，就多出一种两边对不上的方式：D1 台账 `ready`/`running` 却没卡 → 补建卡（按标题前缀 `[<entryId>]` 幂等查重）；D2 卡没有通过 `cardId` 绑到任何台账行 → 只报告、计入 unowned，绝不隐藏、绝不自动删；D3 台账已终态（`done`/`cancelled`）但卡还开着 → 收口并归档；D4 卡在 `running` 而台账不是、没有 claim、或活性三项静默 >20 分钟 → 退回 `ready` 并留诊断；D5 台账非终态（`running`/`blocked`）但卡已 done、或 `cardId` 在板上查不到 → 默认只报不动，另有两个显式开关才收口或重开卡；D6 台账终态、卡非终态且未归档 → 直接归档（不要求 claim）；D7 台账 `blocked` 而载体还活着（假阻塞）→ 隔 ≥60 秒的第二次命中后才 `blocked → ready → running`，只回正不收口；D8 已退役的台账镜像还在盘上 → 先留档再删除，不比内容，指向 canonical 的软链不算漂移；D9 `automation` 条目仍挂着 `cardId` → 只报告，修复走 release-cards 命令。

其中两条值得单独说清纪律。**D4 不单凭结构判死**：没有 claim、台账不在跑，都只记诊断、不动卡；只有三项活性信号同时静默才够降级，活性信号取不到（`activity-unknown`）时一律不降级。**D7 要两次确认**：探针第一次命中只记候选（`--check` 看得见）；要等至少一分钟后的第二次命中，或者心跳已更新，`--apply` 才动那一行——而 `automation` 载体豁免，因为它的活性是调度器的最后运行状态，不是常驻心跳。

退出码是四档而不是三档：`0` 收敛（`--check` 干净，或 `--apply` 后复核 D1/D3/D4 全 0）；`1` 只读模式查到漂移；`2` 真错误——契约缺失或漂移、台账读不了、未知状态、卡后端读不到、apply 报错，或 `--apply` 后仍有残留（残留会写成待办文件，但退出码仍是 `2`，让巡检看得见红）；`3` 部分完成——墙钟或单轮写配额用完。之所以要有 `3`：外层调度把任何非零退出都当失败，连续十次就把任务禁用；「没查完」不是「查到漂移」。同一个条目连续两次撞写冲突则记 `deferred`，既不进残留也不影响退出码。

Card/ledger sync follows the same asymmetry: a drift check that **finds** drift exits `0` (a finding,
not a failure) while the check **erroring** exits `2`; and one pass reads the cards from a **single
snapshot**, because per-card calls accumulate until the whole check times out. When the card backend
holds no job container, the sync no longer skips silently — an unresolved terminal row is handed to
the reconciler instead of vanishing.

卡/台账同步是同一个不对称：**查到**漂移交 `0`（发现不是失败），检查**出错**才交 `2`；一次同步从**单次快照**取卡，因为逐卡调用会累积到整轮超时。卡片后端没有 job 容器时，同步不再静默跳过——没收口的终态行交给对账器，而不是凭空消失。

## One writer, and no RPC inside the lock

Every write command holds an exclusive `flock` for the whole read-modify-write, on a lock file next
to the ledger, and releases it only after the atomic replace. Three rules make that lock safe
instead of merely present:

- **no RPC inside the critical section.** Creating a card is a network call; it used to happen while
  the lock was held, so one slow backend stalled every writer. Card creation, claim-side card
  repair and archiving now run *after* the lock is released, queued as jobs the CLI drains in its
  `finally`; the card layer refuses a call that arrives with the lock already held, so the rule
  cannot be re-broken by a new caller.
- **a hold watchdog.** A single hold is capped (5 s by default, overridable per environment) and
  policed on a real clock at 0.1 s granularity. On overrun it force-releases the lock and writes
  telemetry plus a stderr warning — normal holds are milliseconds, so a watchdog trip is a defect
  signal, not routine noise.
- **bounded waiting.** Acquisition retries a handful of times inside a 6 s budget and then exits
  with its own code rather than writing concurrently. A holder that is provably gone is taken over;
  a live holder is waited for.

卡点还在于：取不到锁时绝不并发写，而是明确失败。

## Projection happens on release

Callers no longer have to remember to sync the card. The lock release is the single trigger: every
entry touched in that transaction is marked dirty and projected once, in the same order, after the
lock drops. The projection is idempotent (already the target state → no-op, no meaningless move),
reads its status mapping from the **one** state vocabulary, and never raises: a failure becomes a
warning plus a pending card job that the worker later replays. Two consequences worth copying: a
`cancelled` row projects to **archive**, not to "done" (the old mapping made the vocabulary say
archived while the card said completed); and a card the projection cannot find is recorded as
`missing` and skipped, never conjured into existence — creation is a separate, explicit intent.

卡片同步不再靠调用方自觉。**释放写锁就是唯一触发点**：这次事务里改过的条目被标脏，出锁后按同一顺序各投影一次。投影是幂等的（已是目标态就 no-op，不做无意义的 move），状态映射只读**那份**词表，而且绝不抛异常：失败变成一条告警加一条待重放的卡任务，由 worker 事后补。两条值得照抄的后果：`cancelled` 投影到**归档**而不是「完成」（旧映射让词表说归档、卡说已完成）；投影找不到卡时记 `missing` 跳过，绝不凭空建卡——建卡是另一个显式意图。

## Caps

Writing sub-agents run at most three at a time in one deployment; reviewers and read-only legs are
counted separately, and the cap is enforced before the spawn, not by watching the host afterwards. A
load ceiling sits in front of it: when load is already above the ceiling, no new write task starts.
Context rollover has the same shape: one constant (60 % of the window) is defined once and every
consumer — the guard, the patrol, the docs — reads it, so raising the threshold is one edit rather
than a hunt for stale copies. It was lowered to 40 % once and raised again, because rotating early
throws away exactly the context the next step needs.

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

