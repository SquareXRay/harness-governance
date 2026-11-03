# 09 · Cleanup

> Three verbs, in this order: **discover → settle → reclaim**. The middle one is the whole design; a cleanup script without it deletes work that was still running.

> 三个动词，顺序不可换：**discover → settle → reclaim**。中间那一步就是整个设计——没有它的清理脚本，删掉的是还在跑的活。

## The triad

**discover** is a read-only scan. It may write a report, and it may write nothing at all;
it never touches the state it is inspecting. Every finding carries the field values that produced
it, so a second run over the same bytes produces the same list.

**discover 是只读扫描。** 它可以写报告，也可以什么都不写；它绝不改动被检查的状态。每条发现都带上产生它的字段值，所以同一批字节跑两遍，得到同一份清单。

**settle** is the only writer. It closes an item by citing the evidence that the item is finished —
a commit, a report path, a terminal event — and it runs one item type at a time, single-writer.
Nothing is settled because it *looks* finished; it is settled because something else already said so.

**settle 是唯一的写者。** 它靠引用「这件事已经结束」的证据来收口——一个 commit、一个报告路径、一个终态事件——并且一次只处理一类对象，单写者。**看起来**结束了不算，要有别的东西已经这么说过。

**reclaim** runs only inside the maintenance window, and only on items settle has already closed.
Its order is fixed: export first, hash it, write the manifest, then delete. Delete without the export
is not reclaim, it is loss.

**reclaim 只在维护窗口里跑，且只处理已被 settle 收口的对象。** 顺序固定：先导出、再算哈希、再写清单、最后删除。没有导出的删除不叫回收，叫丢失。

```
discover  (read-only, every 30 min)   → findings, no mutation
settle    (single writer, daily)      → closed rows + evidence pointer
reclaim   (maintenance window, daily) → export → sha256 → manifest → delete
```

## Determinism: the scan never asks a model

Discovery and judgement are field comparisons — a status column, a timestamp delta, a pid that no
longer exists, a hash that no longer matches. No model sits anywhere on that path. The reason is not
cost: a model verdict is not reproducible, and a cleanup job that cannot be replayed cannot be audited,
and one that cannot be audited will eventually delete something it should not have.

发现与判定都是字段比较——一个状态列、一个时间差、一个早就不存在的 pid、一个对不上的哈希。这条路径上没有模型。理由不是成本：模型给出的判定不可复现，而不可复现的清理跑批无法审计——无法审计的清理，迟早会删掉不该删的东西。

A model is allowed in exactly one place, downstream of the verdict: rewriting an alert into a readable
sentence, or summarising a report. It receives findings and returns prose. It cannot change a verdict,
and it never sees the delete path.

模型只允许待在一个地方，而且在判定之后：把告警改写成一句人话，或者给报告写摘要。它拿到的是发现，交出的是散文。它改不了判定，也碰不到删除路径。

## What it looks for, and how each one is decided

| Object | Signal | Threshold | Verdict |
|---|---|---|---|
| **Ghost inflight** | ledger row still `running` while its subsession reached a terminal state (`done` / `failed` / `killed`) | terminal event, or no write and no process | stale row |
| **Orphan running card** | no claim, no live session bound to it | older than 2 h | orphan |
| **Stale lock** | lock file whose owner process is gone | older than 24 h | dead lock |
| **Transcript** | conversation log dated before the retention boundary | boundary | expired |
| **Archive package** | bundle produced by an earlier reclaim | older than 30 days | expired |

| 对象 | 信号 | 阈值 | 判定 |
|---|---|---|---|
| **幽灵 inflight** | 台账行仍写 `running`，而它的子会话已到终态（`done` / `failed` / `killed`） | 终态事件，或无写盘且无进程 | 陈旧行 |
| **无主 running 卡** | 无人认领，也没有任何活着的会话绑在它上面 | 超过 2 小时 | 无主 |
| **失效锁** | 锁文件，其持有进程已不存在 | 超过 24 小时 | 死锁 |
| **转录** | 早于保留边界的对话记录 | 边界 | 过期 |
| **归档包** | 早先回收产出的打包件 | 超过 30 天 | 过期 |

Each threshold is a number in one place, not a judgement call per run. Changing one is a reviewed
commit, not an argument with the script at 3 a.m.

每个阈值都是某处的一个数字，不是每次跑批时的一次判断。改它要走一次被审阅的提交，而不是凌晨三点跟脚本吵架。

## Evidence and the guardrails around deletion

**Settle needs a pointer.** A row closes only with a commit sha or a report path attached. No pointer,
no close: the row stays, and it appears in the next report as an item with missing evidence. That list
is the point — a cleanup system that always succeeds is a cleanup system that is lying somewhere.

**收口必须带指针。** 一行只有在附上 commit sha 或报告路径后才允许关闭。没有指针就不关：行留在那里，并在下一份报告里以「缺证据」出现。这份清单本身就是目的——一个永远成功的清理系统，一定在某个地方说谎。

**Reclaim needs an export.** Before anything is deleted: write the payload out, compute `sha256`,
and append a manifest line — path, size, hash, source, reason, timestamp. The manifest is written
before the delete, so a crash mid-run leaves a manifest describing what was about to go, never a gap.

**回收必须先导出。** 删除之前：把内容写出去、算 `sha256`、追加一行清单——路径、大小、哈希、来源、理由、时间戳。清单先于删除落盘，所以跑批中途崩掉留下的是一份「本来要删什么」的清单，而不是一段空白。

**No boundary, no deletion.** If the retention boundary is missing, unreadable or contradictory, the
run refuses and reports it. Fail-closed applies to cleanup exactly as it applies to a gate: unknown
never means "old enough".

**没有边界就不删。** 保留边界缺失、读不到或自相矛盾时，跑批直接拒绝并报告。失败朝严对清理同样成立：未知永远不等于「够旧了」。

**Human conversations do not expire by time.** A transcript from a person is removed by a person,
with an explicit action, and that action is recorded. Time-based deletion applies to machine-produced
artefacts only.

**人的对话不按时间过期。** 来自人的转录由人删除，通过一次显式动作，并留下记录。基于时间的删除只适用于机器产物。

## One scheduler, two speeds

Two cadences, both on the existing scheduler: a **30-minute read-only pass** that only discovers and
stays silent when clean, and **one daily heavy pass** in the maintenance window that settles and
reclaims. No new daemons, no watchers, no second queue — a cleanup subsystem that needs its own
long-running process is a new thing to keep alive, and things that must be kept alive are the ones
that silently die.

两种节奏，都挂在既有调度上：**每 30 分钟一次只读扫描**，只做发现，干净时零输出；**每日一次重活**，在维护窗口里收口与回收。不加常驻进程、不加 watcher、不加第二套队列——一个需要自己长活进程的清理子系统，就是一个新的「要养着」的东西，而需要养着的东西，就是会悄悄死掉的东西。

The read-only pass writes no state, which is what makes it safe to run often: the cheap half is cheap
because it cannot do damage, and the half that can do damage runs once a day, in the window where a
failure is expected to be visible.

只读那一半不写状态，所以可以跑得很勤：便宜的那一半便宜，是因为它做不了坏事；能坏事的那一半每天只跑一次，而且跑在「出问题本来就该被看见」的窗口里。

## Why this belongs in the governance layer

Cleanup is the one place where a stale projection becomes a destructive act. The dispatch ledger says
a task is running; the card says it is owned; the lock says someone is inside the file set. When any
of those outlives its truth, the safe form of repair is not "trust the newest value" but "settle it
with evidence, then reclaim what is provably dead". The triad is that sentence turned into three
verbs with different permissions.

清理是唯一一处「过期的投影会变成一次破坏性动作」的地方。台账说任务在跑，卡片说有人认领，锁说有人正在改那批文件。当其中任何一个活过了它对应的事实，安全的修法不是「相信最新的那个值」，而是「带证据收口，然后回收已被证明死掉的部分」。三动作就是这句话拆成的三个动词，各自不同的权限。

Related: [`03-gates.md`](03-gates.md) for the block/allow contract, [`04-orchestration.md`](04-orchestration.md)
for the ledger and cards-as-projection, [`05-evaluation.md`](05-evaluation.md) for the probes that keep a
scan honest.

相关章节：[`03-gates.md`](03-gates.md) 的「拦/放」契约、[`04-orchestration.md`](04-orchestration.md) 的台账与「卡片只是投影」、[`05-evaluation.md`](05-evaluation.md) 里让扫描保持诚实的探针。
