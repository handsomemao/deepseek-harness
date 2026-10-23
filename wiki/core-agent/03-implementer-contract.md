# 03 实现者约定

[返回目录](README.md)

本包只有一小部分规则由注册表代码强制，其余写在 JSDoc 里，由工厂、驱动器和插件遵守。

## 注册表代码强制的规则

违反即抛错或被注册表自动处理，不依赖实现者自觉。对应测试见 [tests/agent.spec.ts](../../packages/core/agent/tests/agent.spec.ts)。

- `agent.id === agent.session.id`。
- 同一个 id 只能有一个活 entry。
- 只能注册一个工厂；没有工厂时 `create`/`resume` 报 `no agent factory registered`。
- 先 `enter` 再 `announce`；`announce` 的必须是同一个 entry 对象，且只能公告一次。
- `agent/created` 监听器串行 await；任一抛错则创建失败、entry 回滚，已收到 created 的监听器会补收 disposed。
- 公告过的 Agent 被 detach 时一定发出 `agent/disposed`；没公告过的不发。
- 旧 detach 函数删不掉之后用同一个 id 注册的新 Agent。
- 在 `agent/created` 监听器中调用 detach，会推迟到这次创建分发结束后才执行，后续监听器仍能看到该 Agent。
- `agent/disposed` 监听器的同步异常和 Promise 拒绝只记 warn。
- 注册表进入关闭或销毁阶段后，不能建立新的 initiator 边界。

## 工厂必须遵守的约定

来源：`AgentFactory`、`CreateAgentOptions`、`ResumeAgentOptions`、`AgentHandle` 的 JSDoc（[src/index.ts](../../packages/core/agent/src/index.ts)）。

- 先创建 `agentCtx`，再 await `setup`；setup 完成前，会话和 Agent 都不能插入或公告。
- setup 的所有 await 结束后，紧接着在发布前同步调用 `commit()`。
- 发布顺序：插入会话和 Agent，公告会话，串行 await `agent/created`。所有 created 监听器完成前不处理排队输入。
- 任何一步失败都整体回滚，不发布 id；已经开始的创建公告都要用对应的 disposed 配对。
- 生命周期挂在传入的 `ownerCtx` 上，不能挂在工厂自己注册时的上下文上。
- `signal` 只用于取消创建，返回 handle 前与 Agent 解绑。
- `handle.dispose()`：停止并排空循环、从注册表移除、移除会话、撤销作用域。工厂卸载时拆掉它创建的所有 Agent。
- `resume` 要求已配置 `sessionPersistence`，否则拒绝。

## Agent 对象的语义

来源：[src/runtime-types.ts](../../packages/core/agent/src/runtime-types.ts) 中 `Agent` 成员的 JSDoc。

- `status` 只有 `idle` 和 `running`；每次转换发出 `agent/status`；会唤醒驱动器的输入到达时同步进入 `running`。销毁不是第三种状态。
- `cancel(cause)`：
  - 同一次活动里第一个原因生效。
  - 没有活动时是空操作，不影响之后的工作。
  - 默认清空收件箱；`keepInbox: true` 保留，且不记录取消 splice。
- `whenIdle()`：整个 Agent 完全停稳后 resolve，不对应某条特定消息。
- `runMaintenance(task)`：只能在真正空闲时启动，否则同步抛错；执行期间 `status` 保持 `idle`，期间到达的输入留在收件箱，任务结束后再处理。
- 输入方法：
  - `followup`：唤醒，单独开一个新轮次。
  - `steer`：唤醒，在最近的步骤边界被处理；步骤被拒绝时 steer 留在收件箱等下次唤醒。
  - `inject`：不唤醒，等下一个步骤边界被领取。
- `agent.ctx` 在 Agent 销毁后拒绝新注册。
- 驱动器把自己的整个生命周期包在 `ctx.agents.withInitiator(agent, …)` 里。

## 收件箱与持久化

- 收件箱的每次变更写成一条 `agent/inbox/spliced` 事件；`clear()` 和 `remove()` 是持久取消，带 `outcome: 'canceled'`。
- 先把变更提交到会话日志，再发 `agent/inbox/*` 实时通知。
- 进入模型请求的内容都必须能从会话日志重建（仓库规则：Model-visible ⟺ logged）。

## 事件发出时机

- `agent/pre-step`：返回 `enter` 时携带完整、冻结的消息批次；领取会从收件箱移除候选消息，领取后插入的消息等下一个边界。
- `agent/request`：在提示词组装和 `step/start` 之后、提交系统提示词和用户批次之前触发；在此处取消则两者都不提交。
- `agent/turn-stopping`：轮次关闭前 await，然后重新读收件箱；有新 steer 就再跑一步，没有就关闭轮次。
- `agent/assistant-stream`：顺序为 start、chunk、end；chunk 的 `index` 从 0 连续递增，`end.index` 等于 chunk 总数；`end.outcome` 为 `committed` 时，最终的 `assistant/message` 或 `assistant/attempt` 必须已写入日志。
- `agent/disposed`（agent-loop 的约定）：在驱动器停稳和作用域撤销之后、会话 detach 之前发出。
- 驱动器发事件用 `agentEvents()`，组装提示词用 `assembleContextFor()`，避免作用域与 Agent 不一致（见 [04](04-helpers.md)）。

## 插件与监听器作者

- `setup` 只能组合（工具、提示词段、监听器、`restrict()`），不能驱动 Agent；创建完成后才能驱动。
- `agent/created` 监听器不能 await `agent.whenIdle()` 或自己所有者的 dispose，否则会死锁。
- waterfall 监听器必须调用 `next()`；例外是 `agent/request-error` 接管重试时直接返回 `{ kind: 'retry' }`。
- 包装 `agent/pre-step` 结果时，保留下游的 `startsRequestSeries` 和消息批次，除非有意替换。
- emit 类事件不能否决流程，监听器错误会被吞掉并记日志。
- initiator 不代表存活，也不是授权；跨 worker、进程、持久化、网络时显式传身份。

## 已知限制

摘自 [包 README](../../packages/core/agent/README.zh.md#known-limitations-and-deferred-work)：

- 发起方作用域只存在于进程内。
- 环境身份可能比存活状态更久，生命周期敏感的工作仍要检查 `agent.status` 等。
- `cancel()` 默认清空收件箱；没有"只中止步骤、让轮次继续"的操作。
- 每条 `UserMessage` 恰好携带一个 `MessageSource`。
