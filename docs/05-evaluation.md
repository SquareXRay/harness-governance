# 05 · Evaluation

A gate you cannot measure is a belief. This layer exists to answer three questions weekly:

一道你无法度量的闸，只是一种信念。这一层之所以存在，是为了每周回答三个问题：

1. Is the harness blocking things it should block? *(coverage)*
2. Is it blocking things it should not? *(false-positive rate)*
3. Is it still running at all? *(liveness)*

1. 该拦的东西，闸拦住了吗？*（覆盖）*
2. 不该拦的东西，闸拦错了吗？*（误拦率）*
3. 它到底还在跑吗？*（活性）*

## Telemetry

One JSON line per event, append-only, daily files, 30-day retention:

一个事件一行 JSON，只追加，按天分文件，保留 30 天（代码块不翻译，中文说明在此）：一次「派发被拦」事件长什么样。

```json
{"ts":"...","event":"dispatch_blocked","tool":"sessions_spawn","rule":"dispatch-flow",
 "reason":"no-token","matchedTerms":[],"sessionKey":"...","action":"blocked"}
```

Rules for a usable stream:

一条可用的流，要守这几条：

- **stable `reason` codes**, not prose (`no-token`, `paths-overlap`, `expect-same`);
- **the matcher output included** (`matchedTerms`), so a false positive can be reproduced offline;
- **never log full prompts**; log the goal section, truncated, plus hashes for the rest;
- **blocked attempts are events too** — a gate that only logs successful blocks loses its own
  false-positive data.

- **`reason` 码要稳定**，不要写散文（`no-token`、`paths-overlap`、`expect-same`）；
- **把匹配器的输出带上**（`matchedTerms`），这样误拦可以离线复现；
- **绝不记录完整 prompt**；只记目标段（截断），其余部分记哈希；
- **被拦的尝试同样是事件** —— 一个只记录成功拦截的闸，会把自己误拦的数据丢掉。

## Export before delete

Retention is not "drop rows older than N days"; it is "export them, prove the export, then drop".
Each deletion batch is written per (database, table, month) as `jsonl.zst`, with a `.sha256` file
and a `.manifest.json` recording row count, byte count, time range, source database, sqlite and
zstd versions, and the predicate used.

保留期不是「删掉 N 天前的行」，而是「先导出、再验明导出、最后才删」。每一批待删行按（库、表、月）写成 `jsonl.zst`，旁边一份 `.sha256` 和一份 `.manifest.json`，记下行数、字节数、时间范围、源库、sqlite 与 zstd 版本，以及所用的谓词。

Four properties make this safe to run unattended:

四个性质，让它可以无人值守地跑：

- **one predicate.** Export and `DELETE` share the same `where` + params, so "the rows we exported"
  and "the rows we deleted" cannot drift. When the reuse check compares rows, it compares params
  too — same month and same row count with a different window is a different export.
- **fail-closed.** Any write, checksum or compression problem raises; the caller then deletes
  **nothing**. Half an export is not an archive.
- **idempotent, never overwriting.** A target is reused only when the manifest, the sha256 and a
  re-decompressed row count all agree; otherwise it is rewritten atomically (temp file + rename).
  When a month already has an archive, the new batch gets its own shard with a unique suffix —
  overwriting is the one failure that destroys data already deleted.
- **restorable.** A restore tool verifies sha256 and row count before inserting, and inserts
  idempotently, so replaying the same archive twice is harmless.

- **一份谓词。** 导出与 `DELETE` 共用同一组 `where` + params，「导出的行」与「删掉的行」就不可能有偏差。复用检查比对行时也一并比对 params——同一个月、行数相同但时间窗不同，那是另一批导出。
- **失败朝严。** 写盘、校验、压缩任何一步出问题都抛错，调用方随后**一行都不删**。导出一半不叫归档。
- **幂等且绝不覆盖。** 只有 manifest、sha256 与解压后复算的行数三者一致时才复用；否则原子重写（临时文件 + rename）。某个月已有归档时，新批次另写独立分片并加唯一后缀——覆盖是唯一会毁掉「已删数据」的失败。
- **可回灌。** 回灌工具先校验 sha256 与行数，再幂等插入，同一份归档重放两次也不会重复。

Default windows: transcript events 7 days (lowered from 14 — the tail was never read), audit events
14 days, and the archive files themselves 30 days, deleted only under `--apply`. Removal of the
archive is a separate decision from removal of the rows, and it is the only one that is irreversible.

缺省窗口：会话记录 7 天（由 14 天下调——尾部从来没人读）、审计事件 14 天、归档文件本身 30 天，且只有在 `--apply` 下才真删。删归档与删原始行是两次不同的决定，而删归档是唯一不可逆的那一次。

## Deterministic checks

Ten to fifteen assertions that must hold after any change to the protocol or the rules. Ours
include: hard-rule markers still present in the protocol file, memory index matches the files on
disk, every registered path still protected, the gate suite itself restored to a known count.

十到十五条断言，任何对协议或规则的改动之后都必须仍然成立。我们的包括：协议文件里的硬规则标记仍然在、记忆索引与磁盘上的文件一致、每个登记过的路径仍受保护、闸测试集本身回到已知的用例数。下面两条命令（代码块不翻译，中文说明在此）：`--check` 有任何失败即非零退出，`--list` 列出检查登记表。

```
harness_eval --check           # exit non-zero on any failure
harness_eval --list            # the check registry
```

These are cheap, boring, and catch the most common regression: *a rule was edited away by accident*.

这些检查很便宜、很无聊，但专抓最常见的那类回归：*一条规则被顺手改掉了*。

## The double probe

Covered in `03-gates.md`; the evaluation layer adds the accounting:

探针机制见 `03-gates.md`；评估层在这里补上记账口径：

| Metric | Definition | Target |
|---|---|---|
| must-block cases | one per protective rule | 100 % pass |
| must-allow cases | one per protective rule | 100 % pass |
| replay diff | verdict changes on a historical corpus | each change explained |
| false-positive rate | blocks later reversed by a human | <2% of blocks |
| silent scans | scheduled checks that produced no output | monitored, not counted as success |

| 指标 | 定义 | 目标 |
|---|---|---|
| 必须拦的用例 | 每条防护性规则一个 | 100 % |
| 必须放的用例 | 每条防护性规则一个 | 100 % |
| 回放差异 | 历史语料上的判决变化 | 每一处变化都有解释 |
| 误拦率 | 事后被人推翻的拦截 | 占全部拦截 <2% |
| 静默巡检 | 没有产生任何输出的定时检查 | 被监控，但不计为成功 |

## Silent when clean

Scheduled scans print nothing when healthy. This is the only way a daily report stays readable —
but it means **silence is ambiguous**: it can mean "all good" or "the job died". Resolve it with a
separate liveness signal (last-run timestamp + exit code), never by making the scan chatty.

定时巡检在健康时什么都不打印。这是日报能保持可读的唯一办法——但这也意味着**沉默是有歧义的**：它可能是「一切正常」，也可能是「任务死了」。要用一路独立的活性信号（最后运行时间 + 退出码）来消歧，绝不要靠把巡检写啰嗦。

## Periodic reporting

| Cadence | Content |
|---|---|
| daily | host/user-level anomalies only, one line per finding, deduplicated for 6 h |
| weekly | decision-default compliance, gate counts, memory budgets vs limits |
| monthly | rule pack inventory, dead rules, false-positive review |

| 节奏 | 内容 |
|---|---|
| 每日 | 只报主机/用户层面的异常，每条发现一行，6 小时内去重 |
| 每周 | 决策默认值遵守情况、闸数量、记忆预算与上限 |
| 每月 | 规则包盘点、死规则、误拦复核 |

## A worked anecdote

A new rule pack shipped to catch "read-only framing" in dispatch text. Within a week it blocked a
legitimate task because the phrase "idempotent release" contains the substring "awaiting approval".
The telemetry line carried the matched term, which made the false positive obvious in seconds and
also proved the gate was live. Both facts came from the same event — design for that.

有一个新规则包上线，专门抓派发文本里的「只读措辞」。一周之内，它拦住了一个正当任务，因为短语「idempotent release」里含有子串「awaiting approval」。遥测那一行带着命中项，于是误拦在几秒内就水落石出，同时也证明了这道闸是活的。两个事实来自同一个事件——设计时就冲着这一点去。

