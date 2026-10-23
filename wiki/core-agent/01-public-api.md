# 01 对外接口

[返回目录](README.md)

## `ctx.agents` 服务（`AgentRegistry`）

定义在 [src/index.ts](../../packages/core/agent/src/index.ts)，通过 Cordis 声明合并挂到 `Context.agents`。

| 用途     | 方法                                 | 说明                                                       |
| ------ | ---------------------------------- | -------------------------------------------------------- |
| 创建     | `create(options)`                  | 转发给工厂的 `createAgent(ownerCtx, options)`，返回 `AgentHandle` |
| 恢复     | `resume(options)`                  | 转发给工厂的 `resume(ownerCtx, options)`，返回 `AgentHandle`      |
| 查询     | `get(id)`                          | 返回裸 `Agent` 或 `undefined`                                |
|        | `list()`                           | 所有活 Agent，按注册顺序，返回新数组                                    |
|        | `roots()`                          | 没有运行时父 Agent 的活 Agent                                    |
|        | `isOwnedBy(id, owner)`             | 该 id 的活 Agent 是否由 `owner` 创建                             |
| 工厂     | `setFactory(factory)`              | 注册工厂，只能有一个，返回 disposer                                   |
| 发布     | `register(agent)`                  | 插入并以 `'startup'` 公告，返回 Cordis effect disposer            |
|        | `enter(agent, owner)`              | 只插入、不公告，返回 detach 函数                                     |
|        | `announce(agent, source, signal?)` | 串行执行 `agent/created` 监听器                                 |
| 发起方作用域 | `withInitiator(agent, fn)`         | 让 `fn` 的整条异步链继承该 Agent                                   |
|        | `withoutInitiator(fn)`             | 隐藏继承的 Agent，用于共享定时器、队列泵等                                 |
|        | `currentInitiator()`               | 读当前继承的 Agent，可能为 `undefined`                             |
|        | `requireInitiator()`               | 同上，没有时抛错                                                 |

`create()`/`resume()` 以调用方的 `ctx` 作为 `ownerCtx`。测试 [agent.spec.ts](../../packages/core/agent/tests/agent.spec.ts) 中 "requires a factory and delegates through the calling context" 验证了 `ownerCtx.fiber` 就是调用方插件的 fiber，所以 Agent 的生命周期跟着创建它的插件走。

发起方作用域基于 `AsyncLocalStorage`，只是进程内归因：不代表 Agent 还活着，也不是授权。跨 worker、进程、持久化或网络时必须显式传身份。

## `AgentHandle`

```ts
interface AgentHandle {
  agent: Agent
  dispose(): Promise<void>
}
```

`dispose` 是一种能力：在消费方中只有持有者能拆掉这个 Agent。`ctx.agents.get(id)` 只返回裸 `Agent`。工厂提供方是结构上的共同拥有者，工厂卸载时会拆掉它创建的所有 Agent。

## `Agent`

基础定义只有身份（[src/types.ts](../../packages/core/agent/src/types.ts)）：

```ts
interface Agent {
  readonly id: SessionId
}
```

运行时成员通过声明合并补上（[src/runtime-types.ts](../../packages/core/agent/src/runtime-types.ts)）：

| 成员 | 作用 |
|---|---|
| `options: AgentOptions` | 提供方、模型、推理强度、`maxTokens` |
| `session: Session` | 背后的会话，日志是唯一事实来源 |
| `inbox: Inbox` | 待处理消息，分 `nextTurn` 和 `nextStep` |
| `status: AgentStatus` | `'idle'` 或 `'running'` |
| `ctx: Context` | Agent 作用域上下文，注册只对该 Agent 生效，销毁时撤销 |
| `followup(message)` | 排一个新轮次并唤醒驱动器 |
| `steer(message)` | 在最近的步骤边界插入引导输入，会唤醒 |
| `inject(message)` | 加入面向模型的上下文，不唤醒 |
| `send(message, target, wakeup)` | 上面三者的通用形式 |
| `cancel(cause, options?)` | 中止当前活动，默认清空收件箱，`keepInbox` 保留 |
| `whenIdle()` | 整个 Agent 完全停稳后 resolve |
| `runMaintenance(task)` | 在空闲期跑一个非轮次维护任务 |

## 主要类型

| 类型 | 位置 | 说明 |
|---|---|---|
| `CreateAgentOptions` | index.ts | 创建参数，见 [02](02-agent-creation.md) |
| `ResumeAgentOptions` | index.ts | 恢复参数：`resumeSessionId`、`parentAgent`、`agentOptions`、`signal`、`setup` |
| `AgentSetup` / `AgentSetupCommit` | index.ts | 发布前的组合回调及其可选同步提交 |
| `AgentFactory` | index.ts | 工厂接口：`createAgent`、`resume` |
| `AgentOptions` | runtime-types.ts | 可声明合并扩展的创建选项 |
| `CancelOptions` | runtime-types.ts | `keepInbox` |
| `Inbox` | runtime-types.ts | `clear`、`append`、`prepend`、`replace`、`remove`、`splice` |
| `InboxTarget` | types.ts | `'next-turn' \| 'next-step'` |
| `AgentStatus` | runtime-types.ts | `'idle' \| 'running'` |
| `SessionStartSource` | runtime-types.ts | `'startup' \| 'resume' \| 'clear' \| 'compact'`，后两个目前没有发出方 |
| `PreStepDecision` | runtime-types.ts | `{ kind: 'reject' }` 或 `{ kind: 'enter', messages, startsRequestSeries? }` |
| `RequestErrorAction` | runtime-types.ts | `{ kind: 'retry' }` 或 `undefined` |
| `AssistantStreamFrame` | runtime-types.ts | `start` / `chunk` / `end` 三种帧 |

## `agent/*` 事件

声明在 [src/runtime-types.ts](../../packages/core/agent/src/runtime-types.ts)。所有事件的 `this` 都是 `Scoped<Agent>`，按 Agent 作用域过滤：通过 `agent.ctx` 注册的监听器只收到这个 Agent 的事件。

| 类别 | 事件 | 模式 | 作用 |
|---|---|---|---|
| 生命周期 | `agent/created` | serial | 每个监听器依次 await，抛错即否决创建 |
| | `agent/disposed` | emit | Agent 离开注册表 |
| | `agent/status` | emit | `idle` 与 `running` 之间切换 |
| 收件箱 | `agent/inbox/inserted` | emit | 一条消息进入收件箱 |
| | `agent/inbox/claimed` | emit | 一条消息被轮次领取 |
| | `agent/inbox/discarded` | emit | 一条消息被丢弃 |
| 扩展点 | `agent/pre-step` | waterfall | 拒绝步骤，或替换进入步骤的消息 |
| | `agent/request` | waterfall | 替换模型调用配置 |
| | `agent/request-error` | waterfall | 返回 `{ kind: 'retry' }` 接管重试 |
| | `agent/turn-stopping` | serial | 轮次关闭前运行，可以 steer 让轮次继续 |
| 输出 | `agent/assistant-stream` | emit | 进程内流式帧，只用于展示 |
| | `agent/error` | emit | 步骤或轮次出错 |

## 持久化词汇

- 会话事件 `agent/inbox/spliced`（[src/types.ts](../../packages/core/agent/src/types.ts)）：收件箱每次变更的规范化记录，字段为 `target`、`start`、`removedCount?`、`inserted`、`outcome?: 'canceled'`。
- 投影键 `inbox`：状态为 `InboxState`，跨 wire 时为 JSON 安全的 `InboxWireState`。
- 投影键 `turnBoundary`（[src/projection.ts](../../packages/core/agent/src/projection.ts)）：`TurnBoundaryProjection`，记录当前打开的轮次、最近的步骤边界与轮次号。由 agent-loop 注册；键不存在时读者应理解为"没有轮次"，而不是数据损坏。

## 接入其他子系统

注册表构造时自动完成：

- typert：注册 `agent` 的 lookup 和 host context，远端可以用 `agentId` 解析出 `Agent` 和 `agent.ctx`。
- Workspace 归档准入：见 [04 辅助模块](04-helpers.md#archive-admission)。
