# 05 · Evaluation

A gate you cannot measure is a belief. This layer exists to answer three questions weekly:

1. Is the harness blocking things it should block? *(coverage)*
2. Is it blocking things it should not? *(false-positive rate)*
3. Is it still running at all? *(liveness)*

## Telemetry

One JSON line per event, append-only, daily files, 30-day retention:

```json
{"ts":"...","event":"dispatch_blocked","tool":"sessions_spawn","rule":"dispatch-flow",
 "reason":"no-token","matchedTerms":[],"sessionKey":"...","action":"blocked"}
```

Rules for a usable stream:

- **stable `reason` codes**, not prose (`no-token`, `paths-overlap`, `expect-same`);
- **the matcher output included** (`matchedTerms`), so a false positive can be reproduced offline;
- **never log full prompts**; log the goal section, truncated, plus hashes for the rest;
- **blocked attempts are events too** — a gate that only logs successful blocks loses its own
  false-positive data.

## Deterministic checks

Ten to fifteen assertions that must hold after any change to the protocol or the rules. Ours
include: hard-rule markers still present in the protocol file, memory index matches the files on
disk, every registered path still protected, the gate suite itself restored to a known count.

```
harness_eval --check           # exit non-zero on any failure
harness_eval --list            # the check registry
```

These are cheap, boring, and catch the most common regression: *a rule was edited away by accident*.

## The double probe

Covered in `03-gates.md`; the evaluation layer adds the accounting:

| Metric | Definition | Target |
|---|---|---|
| must-block cases | one per protective rule | 100 % pass |
| must-allow cases | one per protective rule | 100 % pass |
| replay diff | verdict changes on a historical corpus | each change explained |
| false-positive rate | blocks later reversed by a human | < 2 % of blocks |
| silent scans | scheduled checks that produced no output | monitored, not counted as success |

## Silent when clean

Scheduled scans print nothing when healthy. This is the only way a daily report stays readable —
but it means **silence is ambiguous**: it can mean "all good" or "the job died". Resolve it with a
separate liveness signal (last-run timestamp + exit code), never by making the scan chatty.

## Periodic reporting

| Cadence | Content |
|---|---|
| daily | host/user-level anomalies only, one line per finding, deduplicated for 6 h |
| weekly | decision-default compliance, gate counts, memory budgets vs limits |
| monthly | rule pack inventory, dead rules, false-positive review |

## A worked anecdote

A new rule pack shipped to catch "read-only framing" in dispatch text. Within a week it blocked a
legitimate task because the phrase "idempotent release" contains the substring "awaiting approval".
The telemetry line carried the matched term, which made the false positive obvious in seconds and
also proved the gate was live. Both facts came from the same event — design for that.

---

## 中文摘要

评估层只回答三个问题：该拦的拦住了吗（覆盖）、不该拦的拦错了吗（误拦率）、还在跑吗（活性）。

**遥测**：一行一事件，必须带稳定 `reason` 码与命中项（`matchedTerms`），绝不记录完整 prompt（只记目标段截断 + 其余哈希）；
被拦的尝试同样是事件，缺了它就丢失了误拦数据。

**确定性检查**：十来条「改完必须仍成立」的断言（硬规则标记还在、索引与磁盘一致、受保护路径仍受保护、闸用例数未减），
成本极低，专抓最常见的回归——规则被顺手改掉。

**双探针 + 回放**：指标看四项——该拦全过、该放全过、历史回放的判定变化逐条可解释、误拦率 <2%。

**静默即健康，但静默有歧义**：要么全好、要么任务死了。必须另配活性信号（最后运行时间 + 退出码），
不能靠把报告写啰嗦来区分。
