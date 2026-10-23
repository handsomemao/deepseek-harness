# 03 模型请求与日志

[返回目录](README.md) · 上一页：[轮次与收件箱](02-turns-and-inbox.md) · 下一页：[工具、取消与恢复](04-tools-and-recovery.md)

## 概要

模型请求的历史由 Session 日志派生。AgentLoop 先准备并接纳输入，再写入可重建的请求记录，最后冻结请求交给模型服务。系统提示词、动态上下文、请求配置和实时流分别有自己的记录方式。

## 从候选输入到一次模型尝试

```text
preStep()
  领取收件箱消息
  await systemPrompt.assemble(Agent 的作用域上下文)
  渲染 runtime context，形成可能追加的候选消息
  await agent/pre-step → reject 或 enter(messages, startsRequestSeries?)

接纳后写 step/start
step() 的第一次尝试
  渲染本 Step 的系统提示词
  await agent/request → 请求配置
  await llm.prepareCall(config, signal) → 实际路由、能力与默认值
  同步协调并写 system/message
  写准入决策的 user/message 批次
  写必要的 request/header、工具变化提示、request/context
  session.deriveMessages() → 冻结历史与请求
  preparedCall.stream(request)，或 llm.stream(request)
```

`agent/request` 和 `prepareCall()` 执行时，本 Step 待写的系统提示词与接纳用户消息尚未加入历史。在这两处 await 期间取消，不会把本次待接纳内容留下为 `system/message` 或 `user/message`，但已经写入的领取记录和 `step/start` 仍然存在。

提示词协调采用本次 `prepareCall()` 返回的能力；先前的 `request/context` 可能属于旧路由，不能据此决定新请求的提示词更新方式。若 prepareCall 仅因 `NO_ADAPTER` 失败，源码保留经 `agent/request` 提议的配置，让 LLM waterfall 仍有机会处理该路由。

## request/header 与 request/context

`request/header` 保存规范化的请求配置、工具声明，以及哪些值来自适配器默认值。它不保存消息历史，历史仍由日志中的消息事件派生。

| header 的 reason | 写入条件 |
|---|---|
| `initial` | 当前 Agent 首次记录 header，且会话没有既有 header |
| `resume` | 当前 Agent 首次记录 header，且会话已有 header |
| `change` | 本实例后续请求的规范化 header 有变化 |
| `series` | header 不变，但当前请求开始新的消息序列 |

首次请求或配置变化同时开启新序列时，记录原有 reason 并加 `startsSeries: true`。同序列、同 header 的普通后续请求不重复写 header。

新消息序列由 pre-step 的显式 `startsRequestSeries`，或者自上次请求以来 surface 的 `contentGeneration` 变化触发。普通追加用户消息不会单独触发这种变化；替换历史和插件改变已有消息内容会触发。

`request/context` 单独保存实际 provider、model、可选上下文窗口和提示词更新模式；任一字段变化才写入，包括新路由不再提供旧字段的情况。它描述路由元数据，和 header 中的请求输入分开。

工具名称集合相对上一 header 有增删时，循环还追加来源为 `tool-registry` 的 `developer/message`。工具完整定义保存在 header；新工具提示通过 `headerSeq` 引用这份记录。

最终请求包含 header 的配置、`deriveMessages()`、当前工具声明、`toolHistory()`、`sessionId` 和活跃取消信号。它没有独立的 `system` 字段。历史消息和声明被冻结，而取消信号仍能传递取消；`WeakSet` 记住已冻结的消息对象，避免每次请求重复遍历它们。

## 系统提示词如何更新

`SystemPromptProjection` 查看当前 surface 中仍保留的 system 节点，给出待追加或待替换操作；实际写日志由 `step()` 完成。替换只针对具体节点，不删除夹在 system 节点之间的用户、助手或工具历史。

| 当前情况 | 提示词操作 |
|---|---|
| 没有 system 节点 | 追加首个节点，即使提示词为空；空节点不产生模型消息 |
| 连续序列，路由支持 `systemPromptUpdate: 'in-history'`，有效文本未变 | 不写更新 |
| 连续序列，支持 in-history，新的文本非空且有变化 | 在历史后追加新 system 节点 |
| 路由不支持 in-history，或需要开启新序列 | 清空后面的有效 system 节点，在首个节点写入当前文本 |
| 当前渲染为空 | 清空所有有效提示词版本，防止旧指令继续进入请求 |

工具声明变化而本次路由没有 `toolUpdate` 能力时，也要求合并提示词到首个节点。若因此替换已有 system 节点，surface 的变化会让请求记录新序列。

恢复 Agent 或单纯切换 provider/model 不必自动开启新序列，仍由本次路由能力和上述条件决定。提示词向后追加可以保留已有消息前缀；原地替换会让请求从被替换位置开始改变，影响提供方前缀缓存的复用。实际缓存命中由提供方决定。

## 动态 runtime context

`RuntimeContextProjection` 把动态上下文形成来源为 `runtime-context` 的候选 `user/message`，交给 pre-step 一起决定是否接纳。它与系统提示词的更新规则分开。

只有当前上下文不同于仍保留的上一快照，或旧快照被历史替换移除，才提出新快照。第一次上下文为空时不发消息；已有上下文变为空时，会提出明确的清空提示，说明旧快照不再适用。

投影跟踪已经提交的消息，不把“提出候选”当作“已记录”。因此，pre-step 拒绝候选快照后，下一次仍可以提出它。恢复时也从日志与当前 surface 找到仍有效的快照。

## 重试属于同一个 Step

LLM 服务把模型请求失败归一为终态 `error` 或 `aborted`，循环先保存 `assistant/attempt`，再调用 `agent/request-error`。监听器返回 `{ kind: 'retry' }` 时，继续本 Step 的尝试循环；默认处理返回 `undefined`，结束 Turn。

每次重试重新运行 `agent/request` 和 `prepareCall()`，但复用本 Step 的提示词组装结果，不重新执行 assembly、pre-step 或用户消息接纳。用户消息只在第一次尝试追加，失败尝试不会变成供下次模型读取的助手消息。重试期间历史可能被压缩，下一次提示词协调和请求派生会读取更新后的日志。

取消优先于重试决策。请求配置监听器、工具、结果处理等插件自身抛出的异常不统一进入 request-error 重试；这类错误走轮次失败处理。循环本身不设重试次数或延迟，策略由监听器负责。

## 实时输出与最终结算

`AssistantStreamAttempt` 为一次模型尝试同时维护块组装器和紧凑流累计器。每次 push 先快照并累计 chunk，再发出实时 chunk；这时尚未逐 chunk 写入 Session 日志。

```text
实时：start → chunk* → end
结算：追加 assistant/message 或 assistant/attempt → end(committed, seq)
无法写入结算事件：end(abandoned)
```

正常完成时，`assistant/message` 保存组装后的内容、可选 usage、提供方信息和完整紧凑流。失败或待重试尝试写 `assistant/attempt`，保留流记录但不增加模型消息历史。`committed` 表示结算事件已进入 Session 日志，不等同于持久化后端已经完成磁盘 flush。

流被当前 Agent 的取消信号打断时，若已有安全的可见前缀，则写 `interrupted: true` 的 `assistant/message`；没有安全前缀时只写 attempt。未完成的工具调用块不会被当作完整调用执行。进程在结算前硬退出时，这次尚未结算的内存流没有持久记录。

## 源码与测试

- [agent.ts](../../packages/core/agent-loop/src/agent.ts)：`preStep`、`prepareRequest`、`buildRequest`、尝试循环。
- [runtime-context.ts](../../packages/core/agent-loop/src/runtime-context.ts)、[assistant-stream.ts](../../packages/core/agent-loop/src/assistant-stream.ts)：提示词协调、上下文快照与流结算。
- [request-reconstruction.spec.ts](../../packages/core/agent-loop/tests/request-reconstruction.spec.ts)、[request-freeze.spec.ts](../../packages/core/agent-loop/tests/request-freeze.spec.ts)：日志重建与请求冻结。
- [system-prompt-admission.spec.ts](../../packages/core/agent-loop/tests/system-prompt-admission.spec.ts)、[system-prompt-projection.spec.ts](../../packages/core/agent-loop/tests/system-prompt-projection.spec.ts)：接纳时点、清空和路由能力。
- [request-error.spec.ts](../../packages/core/agent-loop/tests/request-error.spec.ts)、[cancel.spec.ts](../../packages/core/agent-loop/tests/cancel.spec.ts)：同 Step 重试、失败尝试与可见前缀。
