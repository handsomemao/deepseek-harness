# 02 创建一个 Agent

[返回目录](README.md)

## 调用方写法

```ts
const handle = await ctx.agents.create({
  sessionId,                  // 同时是 agent id 和 session id
  agentOptions: { provider: 'deepseek', model: 'deepseek-chat' },
  meta: { cwd: '/abs/path' }, // 可选：会话元数据
  // parentAgent,              // 可选：传了就是运行时子 Agent
  // seed,                     // 可选：fork 或回放用的初始事件
  // signal,                   // 可选：只用于取消创建过程
  setup(agentCtx, agent) {
    // 只做组合，不要在这里驱动 agent
    agentCtx.on('agent/pre-step', async (payload, next) => next())
  },
})

handle.agent.followup({
  content: [{ type: 'text', text: 'Summarize this workspace.' }],
  source: { kind: 'user' },
})
await handle.agent.whenIdle()
await handle.dispose()
```

## `CreateAgentOptions`

| 字段 | 含义 |
|---|---|
| `sessionId` | 必填，Agent 和会话共用的身份 |
| `parentAgent` | 运行时父 Agent。省略时是根 Agent，出现在 `roots()` 中 |
| `meta` | 会话元数据：`cwd`、`parentSession`（fork 血缘）、`isSeeded`、`origin: 'subagent'`、`delegationDepth`、`agentPreset` |
| `inheritedEventCount` | `isSeeded` 时 fork 继承的前缀长度 |
| `seed` | 初始事件，从 seq 0 连续 |
| `agentOptions` | 模型路由等配置 |
| `signal` | 只用于取消创建；返回 handle 后中止该信号不取消运行中的轮次 |
| `setup(agentCtx, agent)` | 发布前组合作用域，可返回 `{ commit() }` |

`resume()` 使用 `ResumeAgentOptions`：`resumeSessionId`、`parentAgent`、`agentOptions`、`signal`、`setup`。它先加载持久化会话，再走同样的组合与发布流程，公告来源是 `'resume'`。

## 两条发布路径

注册表提供两种方式把一个已构造的 Agent 公开出去。

简单版 `register(agent)`：

```ts
const dispose = await ctx.agents.register(agent) // 插入并以 'startup' 公告
ctx.agents.get(agent.id)                         // → agent
await dispose()                                  // 移除并发出 agent/disposed
```

拆分版 `enter()` + `announce()`，工厂使用这一种：

```ts
const detach = ctx.agents.enter(agent, parentAgent) // 插入，不公告
await ctx.agents.announce(agent, 'startup')        // 串行执行 agent/created
detach()                                           // 移除；公告过才发 agent/disposed
```

注意：`enter()` 之后 Agent 已经在注册表里，`get(id)` 能查到，只是还没发 `agent/created`。

## 完整顺序

`AgentFactory` 和 `CreateAgentOptions.setup` 的 JSDoc 规定的顺序如下。括号中是 agent-loop 的对应实现位置，便于对照。

```
ctx.agents.create(options)
  └─ factory.createAgent(ownerCtx, options)
       ├─ 准备会话、构造 Agent 和 agent.ctx（尚未插入任何注册表）
       ├─ await setup(agent.ctx, agent)
       ├─ commit()                                 ← setup 返回了 AgentSetupCommit 时
       ├─ 保存 seed 和 setup 期间追加的事件           ← 配置了持久化后端时
       ├─ sessions.enter(session)
       ├─ agents.enter(agent, parentAgent)
       ├─ sessions.announce(session)               ← session/created
       ├─ await agents.announce(agent, 'startup')  ← agent/created（serial，可否决）
       └─ 返回 AgentHandle { agent, dispose }
```

（agent-loop：`setupAndPublish()` 依次调用 setup、`commit()`、持久化写入、`publish()`；`publish()` 依次调用上面四个 enter/announce。）

setup 或 commit 失败时，两个对象尚未进入注册表。enter/announce 期间失败时，对象可能已经短暂可查，创建通知也可能已经发出；回滚撤销注册，并为已开始的创建公告配对销毁通知。调用方收到最初的失败；已经写入存储的事件或监听器造成的外部效果不保证被反向撤销。默认工厂的具体顺序见 [agent-loop 的创建与生命周期](../core-agent-loop/01-creation-and-lifecycle.md)。

## 顺序由谁保证

| 规则 | 保证方式 |
|---|---|
| 先 `enter` 再 `announce`，同一个 entry，只公告一次 | 注册表代码检查，违反即抛错 |
| id 等于 session id，id 不重复 | 注册表代码检查 |
| setup 与 commit 完成后才能发布 | 接口 JSDoc 规定，由工厂实现；默认实现 commit 后还 await 持久化写入 |
| 所有 created 监听器完成前不处理排队输入 | 只写在 JSDoc 里，由工厂遵守 |

换一个不遵守 JSDoc 的工厂也能通过编译，`core/agent` 只能保证表中前两行。

## Agent 与会话的关系

在任意时刻，一个会话最多对应一个活 Agent，两者共用同一个 id。不是会话创建 Agent，而是工厂把两者一起创建。

反过来，一个运行中的 Agent 实例也只驱动构造时绑定的那一个 Session，不在多个会话之间切换。一个 `AgentLoop` 服务可以管理多个这样的实例；同一个 Agent preset 则可以复用于不同会话。Session 内的多次提问增加的是 Turn，不是 Session。

- `CreateAgentOptions` 只有一个 `sessionId` 字段，JSDoc 称之为 "the live agent/session identity"。
- `enter()` 检查 `agent.id === agent.session.id`。
- 注册表是 `Map<SessionId, AgentEntry>`，同 id 已有活 Agent 时再登记抛 `already registered`。

例外情况：

- 会话可以暂时没有 Agent：持久化会话在 `resume()` 之前只是一份日志，`get(id)` 返回 `undefined`。
- 同一个会话可以先后对应不同的 Agent 对象：`dispose()` 后再 `resume()` 会在同一个 id 上构造新 Agent。detach 绑定到具体 entry，旧 disposer 删不掉新 Agent。
- 多个 Agent 意味着多个会话：子 Agent 和 fork 都有自己的 `sessionId`。`parentAgent` 是运行时所有权（决定 `roots()`、`isOwnedBy()`），`meta.parentSession` 是持久血缘，两者互不影响。

## 销毁

默认 agent-loop 的实际清理顺序是：停止并排空循环、撤销 Agent 作用域、关闭会话写句柄、移除 Agent、移除会话。注册表侧的行为是：公告过的 Agent 被 detach 时发出 `agent/disposed`；`agent/disposed` 监听器的错误只记日志。持久历史是否保留与运行时注销是不同事项，见 [agent-loop 的生命周期](../core-agent-loop/01-creation-and-lifecycle.md#销毁与所有权)。
