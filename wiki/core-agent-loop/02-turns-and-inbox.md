# 02 轮次与收件箱

[返回目录](README.md) · 上一页：[创建与生命周期](01-creation-and-lifecycle.md) · 下一页：[模型请求与日志](03-model-requests.md)

## 概要

创建完毕后，Agent 等待输入。`agent.ts` 的 `send()` 决定输入进入哪份队列、是否唤醒驱动器；`kick()`、`turn()` 和 `step()` 负责把这些输入变成执行记录。理解队列与执行区间的区别，可以避免把一次 `followup()` 当作一个带结果的异步调用。

## 两份队列与三种输入

收件箱由 `ReactLoopInbox` 提供，底层状态来自 `inbox` 日志投影。

| 调用 | 正常情况下进入的队列 | 是否唤醒 | 含义 |
|---|---|---|---|
| `followup(message)` | `next-turn` | 是 | 排一个独立的后续 Turn；每个 Turn 最多领取一条普通 followup |
| `steer(message)` | `next-step` | 是 | 运行中在最近的下一 Step 接纳；idle 时启动 Turn |
| `inject(message)` | `next-step` | 否 | 等待下一次输入领取；idle 时留在队列里 |

`send(message, target, wakeup)` 是这三者使用的通用入口。若当前活动已经被取消，新的唤醒输入会被改放到 `next-turn`，在活动收尾后开启新 Turn；`disposed` 取消不安排这种后续唤醒。

每次队列变更先追加 `agent/inbox/spliced`，再发出对应的实时 inserted、discarded 或 claimed 通知。两份队列共享待处理消息身份的唯一性检查；恢复日志时也会拒绝越界 splice 和重复身份。

### 领取时的顺序

`claim('next-turn', turn)` 先领取全部 `next-step` 输入，再领取一条 `next-turn` 输入；`claim('next-step', turn)` 只领取全部 `next-step` 输入。领取会把消息从待处理投影中移除，但还没有把它们追加成 `user/message`。

```text
领取前：next-step = [引导 S, 上下文 I]
        next-turn = [提问 A, 提问 B]

Turn 1 首次领取 → [S, I, A]；B 留给后续 Turn
Turn 1 后续步骤 → 只领取后来到达的 next-step 输入
Turn 2 首次领取 → 当时的 next-step 输入，再加 B
```

这也意味着“收到消息”“领取消息”和“消息进入模型历史”是三个时间点。`agent/pre-step` 可以拒绝或改写领取的候选消息，接纳后的持久化时点见 [03](03-model-requests.md)。

在 pre-step 或 request 的 await 期间到达的新输入不加入已经领取的当前批次，而是等待后续领取。准入拒绝不会把已经领取的消息重新入队。

## phase 比公开 status 更细

内部 `Phase` 有三种状态，公开 `agent.status` 只有两种。

| 内部 phase | 公开 status | 行为 |
|---|---|---|
| `idle` | `idle` | 没有活动，可以启动驱动或维护任务 |
| `maintenance` | `idle` | 维护任务占用 Agent；唤醒输入排队，任务结束后再驱动 |
| `running` | `running` | 执行、等待扩展回调、关闭或检查轮次的驱动器仍在活动 |

因此，公开 `idle` 不足以证明可以再次调用 `runMaintenance()`。该方法检查内部 phase，只允许从真正的 `idle` 开始。创建时的 setup 和发布也运行在维护任务里，确保创建监听器中排入的输入暂不执行。

一次 `running` 区间可以连续包含多个 Turn；Turn 切换时不必发出 `idle`。销毁通过注册表移除表达，不是第三种公开 status。

## kick、turn、step 的分工

| 方法 | 负责的执行区间 |
|---|---|
| `wakeDriver()` | 预约唯一驱动器，把状态设为 running，建立活动完成 Promise |
| `kick()` | 在 `ctx.agents.withInitiator(agent, ...)` 中循环执行 Turn，最终回到 idle |
| `turn()` | 打开 Turn，领取输入，决定是否继续 Step，记录轮次结局 |
| `preStep()` | 领取候选消息、组装提示词与工具、投影动态上下文、运行准入 waterfall |
| `step()` | 解析模型路由，执行模型尝试和工具，返回是否还欠下一次请求 |

`withInitiator()` 让这条异步执行链能取得发起 Agent；工具调度随后把它显式写进工具执行参数。这是进程内归因机制，不能替代跨进程身份传递或授权。

### Turn 什么时候没有 Step

`turn/start` 先于首次领取和准入。`agent/pre-step` 返回 `reject` 时，Turn 以 `blocked` 结束，不写 `step/start`。首批接纳消息为空时，Turn 以 `completed` 结束，也不调用模型。提示词组装等准备操作失败时，同样可能只有 Turn 记录而没有 Step。

空收件箱不一定意味着候选批次为空：变化后的 runtime context 也可以成为候选消息。应以准入决策返回的 `messages` 判断。

### 什么时候继续当前 Turn

模型返回工具调用时，执行工具后通常需要下一次模型请求读取结果，`step()` 返回 `null` 表示还欠一个 Step。某个工具结果声明 `concludesTurn` 时，可以解除这项继续请求的要求；同一次回答里的其他工具仍会执行。

模型已经回答完毕但 `next-step` 有输入时，Turn 也继续。准备停止且队列为空时，驱动器 await 串行 `agent/turn-stopping`；监听器可以调用 `steer()` 加入消息，使当前 Turn 继续。`followup()` 则留给下一 Turn。

一次模型响应以 `max-tokens` 结束时，本 Step 不执行该响应里的工具调用。如果后续 steering 继续了 Turn，已经出现的 `max-tokens` 结局不会被后续正常完成覆盖。

执行 Step 后走到常规轮次收尾检查时，只要任一队列还有输入，当前驱动器就可以再开启 Turn。准入拒绝，以及首批消息被改写为空后的无 Step 完成，会提前结束驱动；旁边尚未领取的队列项留待后续唤醒。**inject 本身不唤醒 idle Agent，但仍在执行的驱动器可以检查并领取它。**

## whenIdle 等什么

`whenIdle()` 等待整个 Agent 当前活动停稳。它读取 `activityDone`，await 后检查是否已经换成新的活动 Promise；若发生替换则继续等待。维护任务及其释放的唤醒输入也在等待范围内。

这不是某条 followup 的完成通知，也不是模型响应的返回值。多条排队输入可以在同一个 running 区间处理；取消还可以丢弃未执行输入。判断某条消息的结局需要它的身份和日志记录，见 [core/agent 的 consumed-work 辅助模块](../core-agent/04-helpers.md)。

停稳也不保证收件箱为空：idle 时 inject 的消息，或取消后 keepInbox 保留的消息，都可能等待下一次唤醒。

驱动器把已报告的轮次错误收在 `kick()` 内，最终恢复 idle，所以 `whenIdle()` 正常 resolve 不等于每条输入都成功。错误应结合 `agent/error` 和 `turn/end` 读取。

## 源码与测试

- [agent.ts](../../packages/core/agent-loop/src/agent.ts)：`Phase`、`send`、`wakeDriver`、`runMaintenance`、`kick`、`turn`、`whenIdle`。
- [inbox.ts](../../packages/core/agent-loop/src/inbox.ts)：持久化变更、两份队列的领取顺序和投影重建。
- [agent.spec.ts](../../packages/core/agent-loop/tests/agent.spec.ts)、[interception.spec.ts](../../packages/core/agent-loop/tests/interception.spec.ts)：输入、步骤与停止扩展。
- [inbox.spec.ts](../../packages/core/agent-loop/tests/inbox.spec.ts)、[cancel.spec.ts](../../packages/core/agent-loop/tests/cancel.spec.ts)：队列重建、取消窗口与后续唤醒。
