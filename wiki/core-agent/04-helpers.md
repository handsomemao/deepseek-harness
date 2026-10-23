# 04 辅助模块

[返回目录](README.md)

## dispatch

文件：[src/dispatch.ts](../../packages/core/agent/src/dispatch.ts)

解决的问题：`agent/*` 事件按作用域过滤，需要把 Agent 作为 `this`（作用域载体）传给分发器，同时 payload 里也有 `agent` 字段。两者如果不一致，监听器会收到错误的 Agent。

- `agentCarrier(agent)`：构造作用域载体（`scopeTarget(agent, agent)`）。驱动器在构造时建一次，热路径复用。
- `agentEvents(ctx, agent, carrier?)`：返回融合分发器 `{ emit, serial, waterfall }`。调用方传的 payload 不带 `agent`，分发器自动注入，并且注入值会覆盖 payload 中同名字段。
  - `emit`：逐个调用监听器，同步异常和 Promise 拒绝都只记 warn，不影响其他监听器。
  - `serial` / `waterfall`：转发给 Cordis 对应方法，带上载体。
- `emitAgentEvent(ctx, agent, name, payload)`：一次性 emit，不保留分发器。
- `AgentSubjectEvent`：类型层面筛出 `this` 为 `Scoped<Agent>` 且 payload 含 `agent` 的事件名。
- `assembleContextFor(agent, signal?)`：返回 `{ agent, scope: agent, signal? }`，保证提示词组装时 Agent 作用域的提示词段和工具不会被漏掉。

## consumed-work

文件：[src/consumed-work.ts](../../packages/core/agent/src/consumed-work.ts)

`foldConsumedWork(events)` 单次遍历日志（或其后缀），回答：

- `end`：最近一个真正处理了工作的 `turn/end`。条件是该轮次进入过模型步骤，或者领取了收件箱输入后以 `blocked`、`aborted`、`interrupted`、`error` 结束。
- `droppedUnrun`：在该轮次之后，是否有工作没跑就被取消出收件箱（`agent/inbox/spliced` 带 `outcome: 'canceled'` 且没有插入替代消息）。

为什么需要它：只看轮次事件无法区分"开始前就被中止的轮次"和"被拒绝或空领取的空轮次"，两者的 `turn/end` 形式相同。收件箱 splice 记录补上了这个信息。输入只有日志本身，所以不管谁发起的取消（所有者、祖先、卸载中的插件），读到的结果都一样。

## model-selection

文件：[src/model-selection.ts](../../packages/core/agent/src/model-selection.ts)

`installModelSelection(agentCtx, ref)` 在 Agent 作用域上注册三个监听器，返回统一 disposer：

| 监听器 | 作用 |
|---|---|
| `system-prompt/assemble` | 组装前锁定 `ref.current` 到 `ref.assembled`，并把 `provider`、`model` 写入提示词变量 |
| `agent/request` | 用锁定值覆盖本步请求的 provider、model、reasoningEffort；选择里没有 effort 时清掉继承来的 effort |
| `agent/pre-step`（prepend） | 锁定的路由与上次请求头不同时，追加一条 `[model changed: ...]` 用户消息 |

要点：

- 组装时锁定，所以并发切换在下一步才生效，同一步的提示词和请求不会分裂。
- 只改推理强度不追加提示。同一提供方内只显示模型名，跨提供方显示 `provider/model`。
- 拒绝、已取消、空批次不追加提示。请求头写入前失败时，下一次请求会再次追加。
- 提示消息的 source 是 `{ kind: 'model-selection', form: 'notice', ... }`。

<a id="archive-admission"></a>
## archive-admission

文件：[src/archive-admission.ts](../../packages/core/agent/src/archive-admission.ts)

注册表构造时调用 `installTurnArchiveAdmission(ctx, lookup)`，对每个已发布的 Agent 生效：

- `workspace/session-activity`（waterfall）：会话的 Agent 处于 `running`（包括等待审批或回答）时，在结果前面加上 `{ kind: 'turn' }`。
- `workspace/session-stop`：Agent 处于 `running` 时调用 `agent.cancel({ kind: 'user' })`，不带 `keepInbox`，所以排队输入被丢弃并记录取消 splice，归档后的会话不会被再次唤醒。

两者都不等待收敛。没有活 Agent 的会话没有可运行的回合。`turn` 这个键通过声明合并加入 `SessionActivityKindMap`（[src/types.ts](../../packages/core/agent/src/types.ts)）。
