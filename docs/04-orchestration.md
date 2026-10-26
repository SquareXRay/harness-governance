# 04 · Orchestration

Delegation is where an unattended agent does the most damage. Two sub-agents editing one file set,
a task nobody collects, a "queued" item that exists only in prose.

委派，正是无人值守 agent 造成最大破坏的地方：两个子 agent 改同一批文件、一个没人收的任务、一条只存在于散文里的「排队中」。

## The ledger is the truth

Every delegated task is a row with a state machine:

每一条被委派的任务就是一行，带状态机（代码块不翻译，中文说明在此）：planned → ready → queued → inflight → done，另有旁支走到 cancelled。

```
planned → ready → queued → inflight → done
                          ↘ cancelled
```

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

登记与许可，就是同一个动作（代码块不翻译，中文说明在此）：`prepare` 原子地一次完成「登记 + 占位 + 签发一次性令牌」，令牌带 TTL，默认 15 分钟；spawn 时由闸核销令牌并翻成 inflight。

```
dispatch prepare --id <slug> --repo <path> --paths <files> --expect <artefact>
  → registers the row, claims it, issues a one-shot token (TTL, default 15 min)
spawn(task containing  queue=<slug>  on its own line)
  → the gate consumes the token, flips the row to inflight, strips the line
```

Properties worth copying:

值得照抄的几个性质：

- **the token is spent by the spawn**, so a crashed spawn needs a fresh one (`reissue`), and the
  ledger records the difference rather than silently retrying;
- **the row cannot be revived** once its token was used; a new attempt gets a new id;
- **the gate is the consumer**, so there is no code path that spawns without an issued token.

- **令牌是被 spawn 消耗掉的**，所以崩掉的 spawn 需要新签一个（`reissue`），台账会把这次差异记下来，而不是静默重试；
- **令牌用过之后，那一行不能复活**；新的尝试拿一个新的 id；
- **闸就是那个消费者**，所以不存在任何一条「没有签发令牌却还能 spawn」的代码路径。

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

这项检查发生在 **claim** 时，而不是在 **登记** 时。这个不对称是故意的：你永远可以记录意图，但你不可能永远开工。下面三条命令的分工（代码块不翻译，中文说明在此）：`add` 永远成功，先落成 queued(blocked)；`claim` 才做判断；`promote-dependents` 在前单完成时把依赖放行。

```
add  --paths <set> --after <other-id>     # always succeeds → queued(blocked)
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

## Stale detection

A low-frequency scan lists `inflight` rows whose session no longer exists or has ended. It does
**not** change state; it writes one line to the daily log and leaves the decision to the owner.
Auto-cancelling on suspicion is how you lose work.

一条低频扫描会列出那些 `inflight`、但所属会话已不存在或已结束的行。它**不**改状态：只往当日日志写一行，把决定权留给 owner。凭怀疑自动 cancel，正是你丢掉别人工作的方式。

## What goes wrong without this

| Symptom | Underlying absence |
|---|---|
| "I thought it was queued" | no row; the promise lived in chat |
| two commits touching the same file | no file-set comparison at claim |
| a task that ran but was never reported | no collector / no evidence field |
| the same task dispatched twice | tokens reusable or missing |

| 症状 | 底下缺的是什么 |
|---|---|
| 「我以为它排上队了」 | 没有行；承诺只活在聊天里 |
| 两个提交碰了同一个文件 | claim 时没有做文件集比较 |
| 任务跑了，却从没被报告 | 没有 collector / 没有 evidence 字段 |
| 同一个任务被派了两次 | 令牌可重用，或者根本没令牌 |

