# 04 · Orchestration

Delegation is where an unattended agent does the most damage. Two sub-agents editing one file set,
a task nobody collects, a "queued" item that exists only in prose.

## The ledger is the truth

Every delegated task is a row with a state machine:

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

A row without `expect` cannot be verified; a row without `evidence` cannot be audited. Both are
required at close time, not at creation time.

## Single-use tokens

Registration and permission are the same action:

```
dispatch prepare --id <slug> --repo <path> --paths <files> --expect <artefact>
  → registers the row, claims it, issues a one-shot token (TTL, default 15 min)
spawn(task containing  queue=<slug>  on its own line)
  → the gate consumes the token, flips the row to inflight, strips the line
```

Properties worth copying:

- **the token is spent by the spawn**, so a crashed spawn needs a fresh one (`reissue`), and the
  ledger records the difference rather than silently retrying;
- **the row cannot be revived** once its token was used; a new attempt gets a new id;
- **the gate is the consumer**, so there is no code path that spawns without an issued token.

## Conflict detection

Two comparisons decide whether a new task may start:

1. **file-set intersection** — same repository and overlapping declared paths, or either side
   declaring nothing (unknown counts as conflicting);
2. **same expected artefact** — two tasks claiming to produce the same report path will overwrite
   each other.

The check runs at *claim* time, not at *registration* time. That asymmetry is deliberate: you can
always record intent, you cannot always start work.

```
add  --paths <set> --after <other-id>     # always succeeds → queued(blocked)
claim                                     # decides: free, or blockedReason=waiting: <id>
promote-dependents                        # on completion: clear deps, make runnable
```

Two edge kinds, one predicate: `blockedBy[]` (hard dependency) and `after[]` (soft, "queue behind")
both resolve through a single function. If you implement them separately they will drift, and the
panel will show one while the scheduler honours the other.

## Owner gate and close contract

`close` requires (a) the caller is the row owner or an explicitly authorised delegate, (b) a
non-empty `evidence`. The gate refuses with exit code 5 rather than writing a partial close — a
row closed without evidence is worse than an open row, because it looks finished.

## Stale detection

A low-frequency scan lists `inflight` rows whose session no longer exists or has ended. It does
**not** change state; it writes one line to the daily log and leaves the decision to the owner.
Auto-cancelling on suspicion is how you lose work.

## What goes wrong without this

| Symptom | Underlying absence |
|---|---|
| "I thought it was queued" | no row; the promise lived in chat |
| two commits touching the same file | no file-set comparison at claim |
| a task that ran but was never reported | no collector / no evidence field |
| the same task dispatched twice | tokens reusable or missing |

---

## 中文摘要

派活是无人值守 agent 最容易出事的地方。台账即真相：每条任务一行，状态机 `planned→ready→queued→inflight→done`，
必须带 `repo/paths/owner/expect`，收口必须带 `evidence`（commit 或报告路径）。

**一次性令牌**：登记与许可合一——`prepare` 原子完成「登记 + claim + 签令牌」，spawn 时闸核销令牌并翻 inflight。
令牌被 spawn 消耗，崩了要 reissue 而不是静默重试；令牌用过的行不能复活，只能换新 id。

**冲突判据**在 claim 时判、不在登记时判：登记永远允许（写意图），执行才判相交（同仓 + 路径相交 + 任一方未声明路径都算冲突；
`expect` 相同也拦）。

**两种依赖边一套判据**：`blockedBy`（硬）与 `after`（软，排队等前单）都走同一个函数，完成时由 `promote-dependents` 幂等放行；
分开实现必然漂移——面板显示一种、调度遵守另一种。

**陈旧检测只报不改**：低频扫描找「inflight 但会话已结束」的行，只写一行日志，绝不自动 cancel。
