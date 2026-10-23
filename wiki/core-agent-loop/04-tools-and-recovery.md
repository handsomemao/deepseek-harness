# 04 工具、取消与恢复

[返回目录](README.md) · 上一页：[模型请求与日志](03-model-requests.md) · 下一页：[源码与测试索引](05-source-map.md)

## 概要

模型返回完整助手消息后，AgentLoop 从中提取工具调用并调度。执行可以并发，写入结果和追加上下文仍保持模型调用顺序。取消与异常不仅要停止工作，还要留下可供后续模型请求读取的完整工具结果。

## 工具调度如何分组

`executeToolCalls()` 取得当前发起 Agent，把模型参数解析成工具执行输入，再询问 `ctx.tools.executionMode()`。空参数文本转换为 `{}`；非法 JSON 保留为原始文本，交给工具管线处理。

| executionMode | 调度方式 |
|---|---|
| `parallel` | 使用有上限的滚动池，已开始调用结束后补入后续调用 |
| exclusive 模式 | 一次只处理当前调用；前一池排空后才开始，完成后再处理后面的调用 |

`maxParallelToolCalls` 默认值为 10，设为 1 时串行。配置属于服务，每个工具组通过 volatile 配置读取当前上限；限制的是一个 Agent 步骤的并行安全调用，不是所有 Agent 加起来的全局并发数。

```text
模型调用顺序：A(parallel), B(parallel), C(exclusive), D(parallel)

执行：并行池 [A, B] 排空
        → 单独执行 C
        → 执行 D

若 B 先完成：结果仍等待 A，再按 A、B 顺序提交
```

后续调用开始前会重新检查执行模式。工具可能被插件替换，或先前结果的处理改变了注册表；待开始调用如果变为 exclusive，会成为下一道屏障。

## 哪些部分可以并行

工具调度把准备、执行和结果处理分开。真正可以重叠的是 dispatch 与工具函数执行；前置策略、后置处理、日志结果和 additionalContexts 保持模型调用顺序。

```text
按顺序：写 tool/call → prepare / 前置策略
可并发：dispatch / 工具函数
按顺序：finalize 或 finish → 写 tool/result → 接纳 additionalContexts
```

每个 `tool/result` 用 `sourceEventSeqs` 引用对应 `tool/call`，结果包含模型内容及可选错误、UI 元数据。额外上下文进入 `next-step` 收件箱，由后续 pre-step 接纳；不会被插到尚未结束的工具结果批次之间。

某个结果声明 `concludesTurn`，表示工具执行后无需自动再请求模型回应。它不会跳过同一助手消息里剩余的工具，也不会丢弃已经排队的 steering 或 additionalContexts；这些输入仍能让 Turn 继续。

并行分类针对单个调用。若安全性需要比较不同调用的共享资源，不能单靠本调度器的 parallel 分类保证，应保留 exclusive 执行。工具实现、参数校验和策略细节见 [工具执行管线](../../docs/tool-execution-pipeline.zh.md)。

## 取消当前活动

`agent.cancel(cause)` 默认清空两份待处理队列，并中止当前 Turn 或 maintenance 的信号。`keepInbox: true` 只保留尚未领取的消息；它不重新入队已经 claim 的批次，也不自动唤醒保留下来的队列。

如果当前没有活动，取消不会设置一个影响未来输入的永久标记；默认的队列清空仍然生效。当前活动的第一次取消原因保留，后续取消不能覆盖 AbortSignal 已记录的原因。

| cause.kind | 来源 |
|---|---|
| `user` | 用户停止 |
| `parent` | 父方取消 |
| `hook` | 扩展钩子停止，携带 reason 文本 |
| `disposed` | 实例生命周期清理 |

新的唤醒输入在取消收尾期间到达时，放入下一 Turn，并在非 disposed 活动停稳后重放唤醒。disposed 不启动替换工作，确保 teardown 能结束。

`turn/end` 写 `aborted` 和取消原因的拷贝，只保留规范字段。活信号的 reason 仍可供执行中的调用读取，但传输层向 reason 添加的临时属性不进入持久日志。

### 流式输出取消

已交付的安全文本或推理前缀会作为 `interrupted: true` 助手消息保留，后续请求能够读取用户已经看到的内容；半截工具调用块不会执行。没有安全前缀时保留 log-only attempt，详见 [流结算](03-model-requests.md#实时输出与最终结算)。

### 工具执行取消

调度器停止补入新调用，等待已经开始的调用收敛，并按顺序提交这些结果。尚未开始的模型调用会补写 synthetic `tool/call` 与错误 `tool/result`，错误码为 `ABORTED_BEFORE_DISPATCH`。

取消不回滚工具已经产生的外部副作用。已经开始的调用必须自行响应信号并完成退出；调度器等待其停稳，不把“已发 abort”当作“已经结束”。

## Step 失败时的工具补齐

`turn()` 在 `step/start` 后安装 `ToolCallRecovery`，只观察当前 Session 的事件。若 Step 抛错，在写 `step/end` 前为当前尚无结果的助手工具请求补齐错误结果。

| 日志中的情况 | 补齐结果 |
|---|---|
| 已记录 `tool/call`，没有已提交结果 | `TOOL_OUTCOME_UNKNOWN`：可能已经执行，结果未知 |
| 助手已请求工具，但没有 `tool/call` 起始记录 | `TOOL_NOT_STARTED`：Harness 尚未把它记为已开始 |
| 已有提交结果 | 保留结果，不覆盖 |

这和取消时明确知道未派发的 `ABORTED_BEFORE_DISPATCH` 是不同情况。结果未知时，恢复消息要求先检查外部状态或询问用户；只有只读或幂等操作适合直接重试。AgentLoop 不自动重跑未知操作。

调度器自身失败时先停止新增派发并等待已开始调用，再向 Step 抛出首个错误。若补齐写入也失败，错误会合并成 `AggregateError`；成功补齐不会把原始失败改为成功。

## Turn 错误与后续输入

未取消的异常写入 `turn/end { kind: 'error' }`；`LlmError` 保留结构化失败，其余异常用 `UNKNOWN` 和错误链文本记录。实时 `agent/error` 让观察者得知失败，驱动器随后收在 `kick()` 内并恢复 idle。

因此，一个 Turn 失败不等于 Agent 被销毁。后续新的唤醒输入仍可继续使用该实例；但不能把 `whenIdle()` 的完成当作上一条消息处理成功。模型请求重试的入口与可重试错误范围见 [03](03-model-requests.md#重试属于同一个-step)。

## 进程重启修复什么

resume 从持久日志重建 Session，并用 `interruptedTurnClosers()` 补齐未关闭的尾部工具结果、Step 和 Turn。它不会改写已经关闭的历史轮次中的不一致记录，也不会撤销旧工具的副作用。

修复完成后由新的 Agent 实例接受后续输入；它不会恢复旧进程中断的函数调用栈。恢复写权和发布时序见 [01](01-creation-and-lifecycle.md#恢复会话)。

## 源码与测试

- [tool-calls.ts](../../packages/core/agent-loop/src/tool-calls.ts)：分组、屏障、滚动池、有序提交、取消与排空。
- [agent.ts](../../packages/core/agent-loop/src/agent.ts)：取消信号、Step 补齐、Turn 结局与错误收尾。
- [tool-calls.spec.ts](../../packages/core/agent-loop/tests/tool-calls.spec.ts)：逆序完成、并发上限、动态重分类、额外上下文与失败停稳。
- [cancel.spec.ts](../../packages/core/agent-loop/tests/cancel.spec.ts)：keepInbox、取消与新输入竞态、流前缀和取消原因。
- [resume.spec.ts](../../packages/core/agent-loop/tests/resume.spec.ts)：重启时的尾部恢复与独占写权。
