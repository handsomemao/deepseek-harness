# 05 源码与测试索引

[返回目录](README.md) · 上一页：[工具、取消与恢复](04-tools-and-recovery.md)

## 概要

本页提供进一步阅读的入口。先沿 `index.ts` 找到创建的实例，再沿 `agent.ts` 追踪驱动过程；其余文件分别承担收件箱、消息更新、流记录与工具调度。测试链接用于核对已有用例，执行方法以仓库 [测试文档](../../docs/testing.zh.md) 为准。

包的运行时 API 从根入口公开，`ReactLoopAgent`、`ReactLoopInbox` 等没有从该入口导出。本页链接用于阅读内部实现；消费方通过 `ctx.agents` 使用 Agent，不把这些源码路径当作包的可导入子路径。

## 全部源码文件

| 文件 | 关键对象或函数 | 阅读重点 |
|---|---|---|
| [index.ts](../../packages/core/agent-loop/src/index.ts) | `AgentLoop`、`FactoryOwnership`、`prepare`、`setupAndPublish`、`resumeWith` | 服务注册、配置启动、创建事务、存储写权与共享清理 |
| [agent.ts](../../packages/core/agent-loop/src/agent.ts) | `ReactLoopAgent`、`Phase`、`kick`、`turn`、`step`、`buildRequest` | 单 Agent 的状态、输入准入、请求尝试、Turn 继续与结束 |
| [inbox.ts](../../packages/core/agent-loop/src/inbox.ts) | `inboxProjectionDefinition`、`ReactLoopInbox` | 日志驱动的待处理队列、splice 校验、领取与实时通知 |
| [runtime-context.ts](../../packages/core/agent-loop/src/runtime-context.ts) | `SystemPromptProjection`、`RuntimeContextProjection` | 系统提示词的节点更新与动态上下文候选快照 |
| [assistant-stream.ts](../../packages/core/agent-loop/src/assistant-stream.ts) | `AssistantStreamAttempt` | chunk 快照、块组装、实时帧与最终日志结算 |
| [tool-calls.ts](../../packages/core/agent-loop/src/tool-calls.ts) | `executeToolCalls`、`runGroup` | 并发分类、执行屏障、有序结果与取消后的补齐 |
| [constants.ts](../../packages/core/agent-loop/src/constants.ts) | `DEFAULT_MAX_PARALLEL_TOOL_CALLS` | 并行安全调用上限的默认值；部署通过 Config 修改 |

## 按问题查测试

| 要核对的问题 | 测试入口 |
|---|---|
| setup 完成前能否找到 Agent？创建监听器失败后如何清理？ | [scope-lifecycle.spec.ts](../../packages/core/agent-loop/tests/scope-lifecycle.spec.ts) |
| 配置 id 是否等于 session id？重挂载如何恢复？ | [config-session-id.spec.ts](../../packages/core/agent-loop/tests/config-session-id.spec.ts) |
| followup、steer、inject 和 maintenance 如何组合？ | [agent.spec.ts](../../packages/core/agent-loop/tests/agent.spec.ts)、[interception.spec.ts](../../packages/core/agent-loop/tests/interception.spec.ts) |
| 待处理输入如何从日志恢复？非法 splice 如何拒绝？ | [inbox.spec.ts](../../packages/core/agent-loop/tests/inbox.spec.ts) |
| 从日志重建的模型请求是否与实际请求一致？ | [request-reconstruction.spec.ts](../../packages/core/agent-loop/tests/request-reconstruction.spec.ts) |
| 模型请求是否冻结，而取消信号仍可使用？ | [request-freeze.spec.ts](../../packages/core/agent-loop/tests/request-freeze.spec.ts) |
| request/prepareCall 期间取消，系统与用户消息是否接纳？ | [system-prompt-admission.spec.ts](../../packages/core/agent-loop/tests/system-prompt-admission.spec.ts) |
| in-history、清空提示词、工具变化与消息序列如何衔接？ | [system-prompt-projection.spec.ts](../../packages/core/agent-loop/tests/system-prompt-projection.spec.ts)、[tool-updates.spec.ts](../../packages/core/agent-loop/tests/tool-updates.spec.ts) |
| runtime context 被保留、清空或压缩后如何处理？ | [runtime-context.spec.ts](../../packages/core/agent-loop/tests/runtime-context.spec.ts)、[system-prompt-projection.spec.ts](../../packages/core/agent-loop/tests/system-prompt-projection.spec.ts) |
| 重试是否沿用同一个 Step？插件错误是否交给请求重试？ | [request-error.spec.ts](../../packages/core/agent-loop/tests/request-error.spec.ts) |
| 工具并发、屏障、结果排序和动态分类如何工作？ | [tool-calls.spec.ts](../../packages/core/agent-loop/tests/tool-calls.spec.ts) |
| 工具声明如何按规范顺序进入 header？ | [tool-order.spec.ts](../../packages/core/agent-loop/tests/tool-order.spec.ts) |
| 取消时留下哪些流记录？后续唤醒是否继续？ | [cancel.spec.ts](../../packages/core/agent-loop/tests/cancel.spec.ts) |
| 恢复尾轮、关闭写句柄、卸载停稳如何处理？ | [resume.spec.ts](../../packages/core/agent-loop/tests/resume.spec.ts)、[shutdown-drain.spec.ts](../../packages/core/agent-loop/tests/shutdown-drain.spec.ts) |

## 调试时先找哪类记录

| 现象 | 先查什么 |
|---|---|
| 消息进了队列却没有模型请求 | inbox 插入与领取、maintenance、pre-step 决策、无 Step 的 turn/end |
| 模型收到了意外上下文 | 接纳的 user/message、system/message surface、runtime-context 快照、request/header |
| 一次提问调用了多次模型 | 多个 Step，或同 Step 的失败 assistant/attempt 与重试 |
| UI 有实时文本，重启后却没有完整记录 | 尝试是否已结算；实时 chunk 本身不代表存储已完成 |
| 工具已经执行但显示结果未知 | tool/call 是否存在、tool/result 是否提交、失败补齐与尾部修复 |
| 状态 idle 但仍有待处理输入 | 不唤醒的 inject、keepInbox 保留项，以及 maintenance 的公开状态 |

`agent/*` 实时事件有作用域过滤，适合观察当前实例；Session 日志适合判断已经提交的事实。调试单条输入时用消息身份关联记录，不能只依赖全局 status 或一次 whenIdle。

## 当前实现的限制

循环没有内建 Turn 预算；工具和 steering 可以继续当前 Turn，限制策略需要使用扩展事件与取消接口。配置 Agent 没有逐实例 setup；需要专属组合时走公开工厂选项。并发分类不能比较兄弟调用的共享资源，结果未知的工具也不会自动重试。

这些限制及模型 token、缓存影响以 [包 README](../../packages/core/agent-loop/README.zh.md) 为准。Agent 的完整事件类型见 [Core 子系统](../../docs/subsystems/core.zh.md)，其余包的细节沿 [架构索引](../../docs/architecture.zh.md) 继续阅读。
