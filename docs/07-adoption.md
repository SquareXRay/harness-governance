# 07 · Adoption

You do not need 21 rule packs. You need three, chosen by cost of the failure, plus one ledger.

你并不需要 21 个规则包。你需要三个——按故障代价挑出来的三个，再加一本台账。

## Minimum viable harness (one afternoon)

### Step 1 — pick three gates by damage, not by elegance

Ask: *which three failures have actually cost me time this month?* Typical answers:

第一步——按损失、而不是按优雅程度挑三道闸。先问：*这个月真正让我花了时间的三次故障是哪三次？* 典型答案：

| Failure | Gate | Enforcement point |
|---|---|---|
| two agents editing the same files | file-set conflict at claim | entry script |
| destructive command run by accident | dangerous-command check | hook |
| "I will do it later" with no row | promise carrier check | hook on the promise tool |

| 故障 | 闸 | 执行点 |
|---|---|---|
| 两个 agent 改同一批文件 | claim 时的文件集冲突 | 入口脚本 |
| 误跑破坏性命令 | 破坏性命令检查 | hook |
| 「我稍后做」却没有行 | 承诺载体检查 | 挂在承诺工具上的 hook |

Write each as one sentence with a cost. If you cannot state the cost, it is not a gate, it is a
preference.

把每一条写成一句带代价的话。如果你说不出代价，那它就不是闸，只是偏好。

### Step 2 — build the single entry

One script per action, doing register + validate + issue token atomically. Until this exists, every
gate you add can be bypassed by the older path.

第二步——建那个唯一入口。每个动作一个脚本，原子地完成登记 + 校验 + 签发令牌。在这个入口存在之前，你加的每一道闸都能被旧路径绕过去。

### Step 3 — add paired probes before you trust it

For each gate: one input that must block, one that must allow. A runner that exits non-zero on
either failure. Then break the gate and watch the runner fail.

第三步——先写成对探针，再去信它。每道闸：一个必须拦的输入，一个必须放的输入；一个在任一侧失败时就非零退出的 runner。然后把闸改坏，看着 runner 失败。

### Step 4 — add one silent scan

Something that can only be detected over time: stale running rows, memory budget drift, index/file
mismatch. Silent when clean, one line per finding, plus a liveness timestamp.

第四步——加一条静默巡检。专找那种只能靠时间才能发现的东西：陈旧的 running 行、记忆预算漂移、索引与文件不一致。健康时静默，每条发现一行，外加一个活性时间戳。

### Step 5 — start the decision ledger

Every question that needs a human becomes a row with numbered options and a four-part trade-off
analysis (what it does / upside / downside / risk). Close it with evidence. This is the cheapest
part and the one people skip; without it, the same trade-off is re-argued every month.

第五步——把决策台账开起来。每一个需要人来定的问题，都变成一行：编号选项 + 四段权衡（做什么 / 上行 / 下行 / 风险）。收口时带上证据。这是最便宜的一步，也是大家跳过的那一步；没有它，同一场取舍每个月都要重吵一遍。

## What to skip

| Tempting | Why to skip it early |
|---|---|
| 20+ rule packs on day one | each needs probes and a telemetry review; you will disable them |
| LLM-as-judge for every gate | non-deterministic control is not control; use it for review, not refusal |
| Auto-fixing violations | a gate that edits state can destroy work; report, then let the owner act |
| OS-level sandboxing first | high setup cost, does not stop "two agents on one file" |
| Dashboards before telemetry schema | you will visualise noise and call it insight |

| 看起来诱人 | 为什么早期别做 |
|---|---|
| 第一天就上 20+ 规则包 | 每个都要探针和遥测复核；你最后会把它们全关掉 |
| 每道闸都用 LLM 当裁判 | 不确定的控制不算控制；它适合做评审，不适合做拒绝 |
| 自动修复违规 | 会改状态的闸能毁掉工作；先报告，让 owner 动手 |
| 先上 OS 级沙箱 | 搭建成本高，而且挡不住「两个 agent 碰一个文件」 |
| 没有遥测 schema 就先做面板 | 你会把噪声可视化，然后管它叫洞察 |

## Migration order that works

1. Take inventory of failures from the last 30 days of logs.
2. Convert the top three into gates with paired probes.
3. Move those actions behind single-entry scripts.
4. Only then start moving rules out of the prompt into packs.
5. Add the evaluation layer once you have ~10 packs; before that it is overhead.

1. 先盘点最近 30 天日志里的故障。
2. 把最痛的三条变成带成对探针的闸。
3. 把这些动作挪到单一入口脚本后面。
4. 到这时才开始把规则从 prompt 里搬进规则包。
5. 规则包到约 10 个时再加评估层；在那之前它只是负担。

## Signs it is working

- The model quotes a gate's reason string back to you instead of retrying blindly.
- A false positive is fixed by narrowing the predicate, not by adding an exemption.
- Removing a gate requires an argument, and the argument is recorded.
- Your weekly report has fewer findings than last week, and you can explain why.

- 模型开始把闸给的理由串引用回来，而不是盲目重试。
- 误拦是靠收窄谓词修掉的，而不是靠加一条豁免。
- 删掉一道闸需要论证，而且论证被记录下来了。
- 这周的周报发现比上周少，而且你能解释为什么。

## Signs it is theatre

- Gates exist that no test exercises.
- Exemptions are granted with a phrase rather than a pattern.
- Nothing has been blocked in a month — either you are perfect, or a gate is dead.
- The rules file is longer than the code that enforces it.

- 存在没有任何测试碰过的闸。
- 豁免是用一句话给的，而不是用一个精确模式给的。
- 一个月里什么都没被拦过——要么你完美了，要么某道闸死了。
- 规则文件比执行它的代码还长。

