# 01 · Principles

> Seven rules. Everything else in this repo is an application of them.

## 1. An enforceable instruction beats a well-written one

A prompt rule is a suggestion. If a rule protects money, data, or another agent's work, it needs
a mechanism that fails closed: a hook that returns `block`, or a script that is the only path to
the action. Prose explains *why*; mechanisms decide *what*.

## 2. Fail closed, and let unknown mean "stricter"

Every gate has a three-state input: allow, block, unknown. Unknown never means allow.
Concretely: an unrecognised mode string falls back to `block`; a resolver that cannot read the
policy file returns "policy unavailable" which the gate treats as a hit; a model that cannot be
verified is not spawned.

## 3. Narrow freely, widen only with a signature

Any change that makes a gate stricter can be made by whoever notices the gap.
Any change that makes a gate looser needs an explicit human decision, recorded in the decision
ledger with the reason. This single asymmetry prevents the slow erosion that kills every
home-grown control system.

## 4. One entry per action

If an action has two supported paths, one of them will skip the checks. Dispatch, ask-the-human,
memory writes and queue transitions each have exactly one entry script. The script is the policy:
it registers, validates, and issues the token atomically, so there is no "register first, be
careful later" window.

## 5. Every promise needs a carrier

"I will pick this up after lunch" is a claim with no state. Promises become rows: a queued task,
a scheduled one-shot wake, a pending decision entry. A promise without a row is treated as
not made — and the assistant is expected to say so.

## 6. Gates must be tested from both sides

For each gate, at least one case that must block and one that must allow, executed by a runner
that exits non-zero if either side fails. A gate with only must-block tests will eventually block
legitimate work; a gate with only must-allow tests is decoration.

## 7. Measure the harness, not just the work

The harness has its own deterministic checks, its own telemetry, and a periodic report. If the
gate suite cannot tell you its own false-positive rate, you do not have a harness, you have a
pile of regexes.

---

## 中文摘要

七条：① 可执行的约束胜过写得好的提醒；② 失败要朝严的方向倒，未知一律按更严处理；
③ 收紧随时可做，放宽必须有人签字并留下理由；④ 每个动作只留一个入口，入口脚本就是策略；
⑤ 任何「我稍后做」都要落成一条台账行，没有行就当没说过；⑥ 每道闸必须同时有「该拦」和「该放」
两类用例，否则迟早误伤或失效；⑦ 度量对象要包括 harness 本身（误拦率、覆盖率、遥测），
否则你只有一堆正则。
