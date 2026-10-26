# 08 · Multi-agent and multi-provider

Two axes most single-agent setups never cross: **who does the work** (one context or several),
and **which model does it** (one provider or a panel). They are the part that actually scales an
agent from "assistant" to "operator", and they interact: parallel workers only pay off if the
workers are independent, and independent workers are only trustworthy if they are heterogeneous.

---

## Axis 1 — several agents, one ledger

### The shape

```
main (owns the plan, talks to the human)
  └─ subagent A  (isolated context, one task, one repo slice)
  └─ subagent B  (isolated context, another repo slice)
  └─ audit legs ×3 (read-only, must not see each other)
```

The parent never does the work it can delegate; the children never coordinate with each other.
All coordination is a side effect of the ledger: they claim disjoint file sets, they close with
evidence, and the parent merges.

### Rules that made it survivable

| Rule | Failure it prevents |
|---|---|
| **One task = one declared file set** | two workers editing one file, conflicting commits |
| **Concurrency cap by kind** (e.g. 2 writers, 3 read-only reviewers) | queue thrash, rate limits, host saturation |
| **Isolated context per child** | a child inheriting a 60 %-full parent and dying mid-task |
| **Worktrees for same-repo parallelism** | dirty tree from one child blocking another |
| **Child must close with evidence** | "done" that nobody can verify |
| **Stale detection reports, never cancels** | auto-killing a run that was merely quiet |

### Lifecycle of one child

```
prepare (register + claim + one-shot token)
  → spawn (gate consumes token, row: queued → inflight)
  → child works, writes files, commits with an explicit path list
  → child closes: row → done with evidence (commit sha / report path)
  → dependents promoted automatically
```

If the spawn is rejected after the token was consumed, the row is **not** revivable — you reissue
under a new id. That single asymmetry (tokens are spent, not reused) removed an entire class of
"dispatched twice" incidents.

### Collectors and wakes

A child that finishes is not useful if nobody is listening. Two mechanisms:

- **completion events** for the normal path;
- **a one-shot fallback wake** registered at spawn time, for the case the event is dropped
  (upstream: completion delivery can fail silently, see `docs/06-incidents.md` #1 and the
  upstream issue we keep open).

The fallback fires on a schedule, checks the ledger, and re-attaches. It is boring, and it is the
reason unattended runs survive a bad night.

---

## Axis 2 — several providers, one routing table

### Why heterogeneous

One model has a stable set of blind spots. Two models of the same family share most of them.
A panel of different families converts "the model was confidently wrong" into "the models
disagree", which a human can adjudicate in seconds. It also catches **harness** bugs: our
most productive session came from a gate that one model obeyed and two flagged as wrong.

### The routing table is the policy

```yaml
tasks:
  mechanical:      { mode: single, legs: [fast-coder], min_legs: 1 }
  ui_frontend:     { mode: produce_verify, producer: fast-coder, verifiers: [reviewer], min_legs: 2 }
  backend_fix:     { mode: single, legs: [fast-coder], min_legs: 1 }
  audit_review:    { mode: multi_leg,  legs: [analyst, reviewer, second-opinion], min_legs: 3 }
  design_planning: { mode: multi_leg,  legs: [analyst, reviewer, second-opinion], min_legs: 3 }
```

Roles, not brands: *fast-coder* is whatever provider is cheapest per wall-clock minute today,
*reviewer* is the one that reads carefully, *analyst* the one that holds long context.
Swapping a provider is a table edit, not a code change.

### Fail-closed dispatch

- A single-leg dispatch for an audit-type task is **refused by the entry script**, not discouraged
  in prose. The refusal explains how to get a legitimate exception (a one-shot leg token issued
  by the entry point, TTL-bounded).
- A model that cannot be resolved at spawn time is not substituted silently; the spawn fails and
  says why.
- If a leg dies, the batch is still a batch only if `min_legs` remain; otherwise it is a failure,
  not a smaller opinion.

### Merging without averaging

Three reports are three claims, not three samples. The merge step is explicit:

1. extract each leg's claims, with the file/line evidence it cited;
2. mark agreement, partial agreement, and contradiction;
3. **keep the disagreement** — write down which leg said what, and why the final decision went
   one way;
4. only claims backed by reproducible evidence become tasks.

"Two out of three said so" is a vote, and votes are exactly what you do not want when the third
leg has the reproduction.

### Cost and wall clock

Panels are expensive; unattended budgets are not. What kept it affordable:

| Control | Value |
|---|---|
| per-task wall-clock cap | enforced by the orchestrator, breach = hard exit |
| per-task token budget | checked before dispatch, not after |
| cheap tier for mechanical work | the default for renames, migrations, formatting |
| panels only for judgement | audit, design, security-adjacent review |
| no panel for the merge | the parent merges; it is one context |

---

## What goes wrong with one agent, one model

| Symptom | Missing axis |
|---|---|
| "the agent said it finished, nothing changed" | no evidence-based close |
| two tasks collide in one repo | no file-set claim |
| context dies mid-project at 60 % | no isolation per task |
| same mistake accepted three times | one model, one blind spot |
| "review" that only agrees with the author | homogeneous review, no independence |

---

## 中文摘要

这一章讲两件事，也是这套东西真正的精髓：**谁来做**（单上下文还是多子会话）与**用哪个模型**
（单 provider 还是异构评审团）。两者互相成就——并行只有在子会话彼此独立时才有价值，而独立
只有在模型异构时才可信。

**多 agent**：主会话只做拆解与合并，子会话各自隔离上下文、各自声明文件集；协调全靠台账——
claim 互斥、带证据收口、父会话合并。四条救命规则：一任务一声明文件集、按类型限并发（写码 2 / 评审 3）、
同仓并行用 worktree、收口必须有证据。令牌被消耗后不可复活，只能换新 id——这条不对称设计消灭了
「重复派单」整类事故。子会话完成事件可能被丢弃，所以 spawn 时登记一次性兜底唤醒，定时查台账再续上。

**多 provider**：路由表按**角色**而不是牌子编写（fast-coder / reviewer / analyst），换供应商是改表不是改代码。
审计类任务单腿派单会被入口脚本直接拒绝，例外只能由入口签发一次性 leg 令牌；模型解析不出来就失败，
不静默替换；腿数不足 `min_legs` 判失败而不是「意见小一点」。

**合并时不取平均**：三条报告是三份主张，不是三次采样。逐条抽取 + 标出证据 + 保留分歧 + 只把可复现的
主张变成任务。「三局两胜」恰恰是最危险的合并方式——第三腿可能握着复现。

**成本纪律**：单任务墙钟上限与 token 预算在派单前判、不是事后算；机械活走便宜档；只有判断题（审计/设计/
安全相关评审）才开评审团；合并不开评审团（父会话一个上下文就够）。
