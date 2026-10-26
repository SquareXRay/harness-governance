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
| false-positive rate | blocks later reversed by a human | < 2 % of blocks |
| silent scans | scheduled checks that produced no output | monitored, not counted as success |

| 指标 | 定义 | 目标 |
|---|---|---|
| 必须拦的用例 | 每条防护性规则一个 | 全部通过 |
| 必须放的用例 | 每条防护性规则一个 | 全部通过 |
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

---
