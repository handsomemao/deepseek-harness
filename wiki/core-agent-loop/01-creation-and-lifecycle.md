# 01 创建与生命周期

[返回目录](README.md) · 下一页：[轮次与收件箱](02-turns-and-inbox.md)

## 概要

`AgentLoop` 是一个服务与工厂，`ReactLoopAgent` 是它创建的执行实例。本页沿服务挂载、创建或恢复、发布、销毁阅读，重点区分运行时资源清理与持久历史保存。

## 服务挂载

`AgentLoop` 的静态注入依赖是 `agents`、`sessions`、`llm`、`tools`、`systemPrompt` 和 `sessionProjections`。持久化服务通过 `ctx.get('sessionPersistence')` 获取，不是普通创建的必需依赖。

构造器先应用 launcher 指定的会话身份、校验配置，再注册 `turnBoundary` 与 `inbox` 投影，建立 `FactoryOwnership`，把自己设置为 `ctx.agents` 的工厂，并注册 provider、model、cwd 提示词变量。最后启动配置中的 Agent。

同一个 Agent 注册表只能有一个工厂，但这个工厂可以创建多个 Agent。两个投影属于服务的生命周期，即使当前没有活 Agent，消费者也能用它们读取历史状态。

### 配置标签与会话身份

`config.agents[].id` 是配置标签。实际 Agent 身份是 SessionId，是否跨启动保持不变取决于另外两个字段。

| 配置情况 | 启动行为 |
|---|---|
| 只有 `id`，未设置两种会话身份字段 | 生成新的 `${id}-session-<uuid>`，每次启动使用新会话 |
| 设置 `sessionId`，当前没有持久化服务 | 使用该身份创建内存会话 |
| 设置 `sessionId`，当前有持久化服务 | 尝试恢复该身份；仅存储确认不存在时新建 |
| 设置非空 `resumeSessionId` | 等持久化服务可用，再恢复已有会话；不存在也不转为新建 |
| 同时设置 `sessionId` 和非空 `resumeSessionId` | 加载时报配置冲突 |
| 多条配置指定相同的精确会话身份 | 加载时报重复身份 |

launcher 可以通过 `configuredAgentIdentities` 按配置标签指定身份，替换对应行的 sessionId/resumeSessionId。配置启动异步进行；错误会写日志并发出 `agent-loop/config-start-failed`，让等待该身份的消费方能够处理失败。工厂正常卸载引起的启动取消不会作为启动失败报告。

配置中没有逐 Agent 的 setup 或 persona 字段。需要为某个实例组合工具、提示词和插件时，使用公开的 `ctx.agents.create()` 或 `resume()` 的 setup 选项。

## 程序化创建

消费方通过 `ctx.agents.create(options)` 调用工厂。注册表传入实际调用方的 Context 作为 `ownerCtx`，因此创建者插件的卸载可以清理这次创建和返回的实例。

```text
ctx.agents.create(options)
  → AgentLoop.createAgent(ownerCtx, options)
  → sessions.prepare(sessionId, seed/meta/inheritedEventCount)
  → 有后端时：取得新会话写句柄，暂不写 seed
  → prepare()：构造 ReactLoopAgent、scope、agent.ctx
                提前登记创建者和工厂的共享清理函数
  → initializeAgent()：进入 maintenance
       await setup(agent.ctx, agent)
       setupCommit?.commit()
       await appendUnstoredSuffix()，有后端时存入 seed 与 setup 事件
       sessions.enter(session)
       agents.enter(agent, parentAgent)
       sessions.announce(session)
       await agents.announce(agent, 'startup', signal)
  → maintenance 结束，允许排队的唤醒输入启动驱动器
  → 返回 AgentHandle { agent, dispose }
```

`SessionPreparation` 持有尚未发布的会话准备结果，失败时负责释放它。`prepare()` 建立执行实例和一份可重复调用的异步 disposer。setup 能看到 Agent 及其 Session，但两个对象此时都不能通过注册表找到。

### 为什么 setup 早于公告

通过 `agent.ctx` 注册的工具、提示词和监听器必须在外部观察者首次看到 Agent 时就已经存在。setup 是可信的同进程组合代码，约定只做组合、创建返回后再驱动；这不是一个禁止调用 Agent 方法的运行时权限检查。

setup 返回的可选 `commit()` 是同步函数，可以完成最终检查。实际实现随后还会 await 持久化后缀写入，再进入注册表；因此，`commit()` 与 `enter()` 之间可能存在异步存储操作。

整个初始化在 maintenance 中运行。创建监听器可以排入输入，但其唤醒必须等所有串行 `agent/created` 监听器完成。创建成功只说明实例可用，不要求已有模型回答，也不要求它始终处于公开 idle。

### enter 与 announce 是不同时间点

`enter()` 将对象放入注册表，`announce()` 发创建通知。`session/created` 发出时，Session 和 Agent 都已经在各自注册表中。`agent/created` 是串行事件，监听器的 throw 或 rejection 可以使创建失败。

setup 或 commit 失败时，两个对象尚未 enter；发布阶段失败时，对象可能已短暂可查，创建通知也可能已被观察。清理会撤销注册，并为已经开始的公告配对销毁通知。它不能撤回监听器的外部效果，也不能保证删除已经写入的存储事件。

fresh 写句柄在真正 append 前可以尚未物化，因此 setup 失败时关闭这种句柄可以不留下持久日志；这项条件不能推广到已经存储 seed、已公告或已修复的会话。

## 恢复会话

`ctx.agents.resume()` 重建运行时对象，不是把原来的 Agent 对象重新唤醒。程序化 resume 要求当前已经配置持久化服务，缺失时直接报错；它和配置启动等待注入服务的路径不同。

```text
persistence.open(id, 'write')：先取得独占写权
  → read：取得物理有效的日志及元数据
  → interruptedTurnClosers：补齐未关闭尾轮的语义记录
  → 通过同一个句柄追加 closers
  → sessions.prepare：用历史、closers 和原会话元数据重建 Session
  → 构造新的 ReactLoopAgent 和新的 scope
  → setup、commit、保存未存后缀、enter 与 announce
  → agent/created 的 source 为 'resume'
```

写权先于加载，防止另一个实例同时恢复并写同一会话。存储层负责读取物理有效日志，Agent 层负责补齐未结束的工具、Step 和 Turn。恢复保留历史，不自动重试结果未知的工具操作。

工具、提示词和监听器属于新的运行时 scope，需要由插件与 setup 重新组合。日志可以恢复会话事实，不能序列化并恢复原来的活 Context 或函数对象。

## 持久化写入归谁负责

发布前，seed 和 setup 期间的事件不会通过活会话的 `session/event` 通知重新播放，循环直接用自己持有的写句柄保存未存后缀。发布后，持久化后端按会话身份接收实时事件批次、flush 屏障和会话退场通知。

循环持有写句柄并负责关闭，不直接操纵 JSONL 文件。没有持久化后端时，新会话只存在于内存；有后端且历史已存储时，销毁 Agent 不等于删除会话历史。

## 销毁与所有权

`handle.dispose()`、创建者 Context 卸载和工厂卸载共用同一个 memoized disposal Promise，并等待实际清理停稳。

```text
中止尚未完成的创建，等待正在执行的发布监听器结束
  → agent.cancel({ kind: 'disposed' })
  → await agent.whenIdle()
  → await agent.scope.dispose()
  → await sessionHandle.close()，排空存储缓冲并释放写权
  → detach Agent
  → detach Session
  → 移除工厂与创建者的生命周期登记
```

驱动、scope 或写句柄清理失败时，实现收集错误并继续后续清理；一个错误原样拒绝，多个错误合为 `AggregateError`。创建失败时优先向调用方保留原始创建错误。

| 信息 | 表达什么 |
|---|---|
| `ownerCtx` | 谁在 Cordis 生命周期中拥有创建事务与实例，卸载时要清理 |
| `parentAgent` | 注册表记录的运行时父子关系，用于 roots、isOwnedBy 等查询 |
| `meta.parentSession` | 存储中的 fork 血缘，即使原父 Agent 不在运行也可保留 |

三者不能互相替代。仅设置 parentAgent 不会改变会话身份，也不能据此推断 scope 嵌套；scope 的资源归属来自实际调用 Context。

创建选项的 `signal` 用于中止创建，创建返回后中止它不会取消运行中的 Turn。运行期间用 `agent.cancel()` 停止活动，用 handle.dispose 清理实例。

## 源码与测试

- [index.ts](../../packages/core/agent-loop/src/index.ts)：构造器、`FactoryOwnership`、`prepare`、`setupAndPublish`、`resumeWith`、共享 dispose。
- [scope-lifecycle.spec.ts](../../packages/core/agent-loop/tests/scope-lifecycle.spec.ts)：setup 未发布窗口、公告顺序、所有权和失败清理。
- [config-session-id.spec.ts](../../packages/core/agent-loop/tests/config-session-id.spec.ts)：配置身份、恢复或创建、重挂载。
- [resume.spec.ts](../../packages/core/agent-loop/tests/resume.spec.ts)、[shutdown-drain.spec.ts](../../packages/core/agent-loop/tests/shutdown-drain.spec.ts)：尾轮修复、写权释放和卸载停稳。
