# Spine · budget → gates → measurement

> One page. Three layers, in the order a single request passes through them. Read this first if you read nothing else.

> 一页总纲。三层，按一次请求经过它们的顺序排。如果只读一页，读这页。

---

## 1 · Budget — decide the cost before the call

Every long-lived agent fails in the same direction first: it spends more than it has. Context fills,
the model summarises its own summaries, and quality decays long before anything errors. So the cost
is fixed **before** the call, not measured after it: a token budget per task, a ceiling on what a
session may carry, a refusal to spawn when the budget is already spoken for, and a memory that is
layered rather than linear — a small index, topic files with hard size caps, and an archive that is
read on demand instead of always. Layering is what makes forgetting a design decision rather than
an accident.

长期运行的 agent，第一个失效方向总是同一个：花的比有的多。上下文满了，模型开始总结自己的总结，质量在任何报错之前就已经衰减。所以成本在调用**之前**就定死：每个任务的 token 预算、一次会话允许背负的上限、预算已被占用时直接拒绝 spawn，以及一套分层而不是线性的记忆——小索引、带硬性大小上限的主题文件、按需才读的归档。分层，是让「遗忘」变成设计决定、而不是意外的那个东西。

Reference: **[longmem](https://github.com/SquareXRay/longmem)** — memory layout, index/topic split, rollover and what to forget.

参考：**[longmem](https://github.com/SquareXRay/longmem)** —— 记忆布局、索引与主题文件的切分、换卷，以及该忘掉什么。

## 2 · Gates — the decision that survives the model

A rule the model can talk itself out of is not a control. So the second layer is a small number of
mechanical gates, each with exactly one entry path: one script per action (dispatch, ask, memory
write, commit), a hook in front of every tool call that defaults to deny, and three-state inputs where
unknown always resolves to the stricter answer. A gate is worth its cost only if it is an *event*:
every block writes one structured record — rule, tool, reason, session — so "how often did this fire
this week" is a file read, not a memory exercise. And every gate carries a pair of probes, one that
must block and one that must allow, because a gate nobody tests is a gate that will quietly stop
working after the next refactor.

模型能把自己说服的规则，不算控制。所以第二层是数量很少的机械闸，每个动作只有一条入口：一个动作一个脚本（派发、提问、写记忆、提交），每次工具调用前有一个默认拒绝的 hook，输入是三态的、未知永远解析到更严的那一侧。一道闸只有在它是个**事件**时才值这个价：每次拦截都落一条结构化记录——规则、工具、原因、会话——于是「这周响了几次」是一次读文件，不是一次回忆。每道闸还配一对探针：一个必须拦、一个必须放，因为没人测的闸，会在下一次重构之后悄悄失效。

Reference: **[harness-governance](https://github.com/SquareXRay/harness-governance)** — `docs/03-gates.md` (block/allow contract), `docs/04-orchestration.md` (dispatch ledger, single writer, cards as projection).

参考：**[harness-governance](https://github.com/SquareXRay/harness-governance)** —— `docs/03-gates.md`（拦/放契约）、`docs/04-orchestration.md`（派单台账、单写者、卡片只是投影）。

## 3 · Measurement — the system reports on itself

Gates rot silently, which is why the third layer is not optional. Three instruments, all cheap:
**paired probes** that fail the suite when a gate stops blocking (or starts blocking what it should
allow); **telemetry** that turns block records into a weekly distribution, including the complement —
rules that never fired, which names dead weight or mis-wiring; and the **cleanup triad**
(discover → settle → reclaim), where a read-only scan finds drift, a single writer closes it with
evidence, and only the maintenance window deletes — after an export and a hash. Measurement also
covers spend: budget drift and context growth are numbers before they are feelings.

闸会悄悄腐烂，所以第三层不能省。三件仪器，都很便宜：**成对探针**，在闸不再拦（或者开始拦它本该放过的）时让整套测试失败；**遥测**，把拦截记录变成一周的分布，含补集——从没响过的规则，它要么是死重，要么接线错了；以及**清理三动作**（discover → settle → reclaim），只读扫描发现漂移，唯一写者带证据收口，只有维护窗口才删除——而且在导出与哈希之后。度量也管花销：预算漂移和上下文膨胀，先是数字，然后才是感觉。

Reference: **[harness-governance](https://github.com/SquareXRay/harness-governance)** — `docs/05-evaluation.md` (probe pairs, telemetry), `docs/09-cleanup.md` (the triad).

参考：**[harness-governance](https://github.com/SquareXRay/harness-governance)** —— `docs/05-evaluation.md`（成对探针、遥测）、`docs/09-cleanup.md`（三动作）。

---

## The one-line version

**Budget without gates is a wish; gates without measurement are a rumour; measurement without a budget
is a very well-documented bankruptcy.** Each layer only holds because the one after it can prove that
the one before it is working — and the repo is linked, not merged: memory and budget live in
[longmem](https://github.com/SquareXRay/longmem), gates and measurement live in
[harness-governance](https://github.com/SquareXRay/harness-governance), and this page is the seam.

**没有闸的预算只是愿望；没有度量的闸只是传闻；没有预算的度量，是一次记录得非常好的破产。** 每一层之所以站得住，是因为后面那层能证明前面那层还在工作——而两个仓是链接关系，不是合并关系：记忆与预算在 [longmem](https://github.com/SquareXRay/longmem)，闸与度量在 [harness-governance](https://github.com/SquareXRay/harness-governance)，这一页是它们的接缝。
