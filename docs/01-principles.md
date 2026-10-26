# 01 · Principles

> Seven rules. Everything else in this repo is an application of them.

> 七条规则。本仓其余一切，都是它们的具体应用。

## 1. An enforceable instruction beats a well-written one

A prompt rule is a suggestion. If a rule protects money, data, or another agent's work, it needs
a mechanism that fails closed: a hook that returns `block`, or a script that is the only path to
the action. Prose explains *why*; mechanisms decide *what*.

写进 prompt 的规则只是建议。如果一条规则保护的是钱、数据，或者别的 agent 的工作，它就需要一个失败朝严的机制：一个返回 `block` 的 hook，或者一个作为该动作唯一路径的脚本。散文负责解释*为什么*，机制负责决定*做什么*。

## 2. Fail closed, and let unknown mean "stricter"

Every gate has a three-state input: allow, block, unknown. Unknown never means allow.
Concretely: an unrecognised mode string falls back to `block`; a resolver that cannot read the
policy file returns "policy unavailable" which the gate treats as a hit; a model that cannot be
verified is not spawned.

每道闸的输入都是三态：放行、拦截、未知。未知永远不等于放行。具体说：认不出的模式串回落到 `block`；读不到策略文件的解析器返回「策略不可用」，闸按命中处理；验证不了模型的 spawn 就不发出。

## 3. Narrow freely, widen only with a signature

Any change that makes a gate stricter can be made by whoever notices the gap.
Any change that makes a gate looser needs an explicit human decision, recorded in the decision
ledger with the reason. This single asymmetry prevents the slow erosion that kills every
home-grown control system.

任何让闸更严的改动，谁发现缺口谁就做。任何让闸更松的改动，都需要一次人的显式决定，并连同理由记进决策台账。就这一条不对称，挡住了所有自建控制系统都会遭到的慢性侵蚀。

## 4. One entry per action

If an action has two supported paths, one of them will skip the checks. Dispatch, ask-the-human,
memory writes and queue transitions each have exactly one entry script. The script is the policy:
it registers, validates, and issues the token atomically, so there is no "register first, be
careful later" window.

一个动作只要有两条受支持的路径，其中一条迟早会跳过检查。派发、问人、写记忆、队列迁移，各自都恰好只有一个入口脚本。脚本就是策略：它原子地完成登记、校验、签发令牌，所以不存在「先登记、之后小心点」的窗口。

## 5. Every promise needs a carrier

"I will pick this up after lunch" is a claim with no state. Promises become rows: a queued task,
a scheduled one-shot wake, a pending decision entry. A promise without a row is treated as
not made — and the assistant is expected to say so.

「午饭后我接着弄」是一句没有状态的主张。承诺要变成行：一条排队任务、一次定时的一次性唤醒、一条待决策条目。没有行的承诺按「没说过」处理——而且助手应当当场说出来。

## 6. Gates must be tested from both sides

For each gate, at least one case that must block and one that must allow, executed by a runner
that exits non-zero if either side fails. A gate with only must-block tests will eventually block
legitimate work; a gate with only must-allow tests is decoration.

每道闸至少要有两类用例：一个必须拦、一个必须放，由同一个 runner 执行，任一侧失败即非零退出。只有「必须拦」用例的闸，迟早会拦住正当工作；只有「必须放」用例的闸，只是装饰。

## 7. Measure the harness, not just the work

The harness has its own deterministic checks, its own telemetry, and a periodic report. If the
gate suite cannot tell you its own false-positive rate, you do not have a harness, you have a
pile of regexes.

harness 有自己的确定性检查、自己的遥测、自己的周期报告。如果闸测试集说不清自己的误拦率，你手里就不是 harness，而是一堆正则。

