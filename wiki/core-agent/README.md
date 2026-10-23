# core/agent（@deepseek-ai/dsh-agent）

`packages/core/agent` 定义 Agent 是什么、怎么找到它、怎么和它交互。它本身不构造 Agent、不跑循环、不发模型请求；这些由注册进来的工厂（默认是 `dsh-agent-loop`）完成。消费方（ACP、UI、插件、编排器）只依赖本包，因此驱动器可以替换。

本 wiki 只基于 `packages/core/agent` 的源码与测试整理，涉及 agent-loop 的地方会单独标注。

运行时一个 Agent 实例固定绑定一个 Session；同一个预设可以创建多个实例，同一 Session 可以记录多个轮次。默认驱动器的具体执行流程见 [core/agent-loop wiki](../core-agent-loop/README.md)。

## 页面

| 页面 | 内容 |
|---|---|
| [01 对外接口](01-public-api.md) | `ctx.agents` 服务、类型、`agent/*` 事件、持久化词汇 |
| [02 创建一个 Agent](02-agent-creation.md) | `create`/`resume` 的参数与流程、两阶段发布、Agent 与会话的关系 |
| [03 实现者约定](03-implementer-contract.md) | 注册表强制的规则、工厂/驱动器必须遵守的约定、插件作者注意事项 |
| [04 辅助模块](04-helpers.md) | `dispatch`、`consumed-work`、`model-selection`、`archive-admission` |

## 源码地图

| 文件 | 职责 |
|---|---|
| [src/index.ts](../../packages/core/agent/src/index.ts) | `AgentRegistry`（`ctx.agents`）、工厂槽位、发起方作用域、创建参数类型 |
| [src/runtime-types.ts](../../packages/core/agent/src/runtime-types.ts) | `Agent` 运行时成员、`Inbox`、`AgentStatus`、`agent/*` 事件声明 |
| [src/types.ts](../../packages/core/agent/src/types.ts) | 纯类型：`Agent` 基础定义、收件箱投影、`agent/inbox/spliced` 会话事件 |
| [src/projection.ts](../../packages/core/agent/src/projection.ts) | `turnBoundary` 投影键的类型声明 |
| [src/dispatch.ts](../../packages/core/agent/src/dispatch.ts) | `agentEvents` 融合分发器、`assembleContextFor` |
| [src/consumed-work.ts](../../packages/core/agent/src/consumed-work.ts) | `foldConsumedWork`：从日志判断被消费的工作结局 |
| [src/model-selection.ts](../../packages/core/agent/src/model-selection.ts) | `installModelSelection`：模型选择与切换提示 |
| [src/archive-admission.ts](../../packages/core/agent/src/archive-admission.ts) | Workspace 归档准入的 `turn` 族 |

## 一句话总结

创建 Agent 经过准备、组合、提交和发布。setup 完成前，两个对象都不进入注册表；发布期间已经发出的创建通知可以被外界观察，后续失败会撤销注册并补配销毁通知。本包保证 id 唯一、公告与销毁成对、陈旧 disposer 无效；具体初始化和执行顺序由工厂和驱动器实现。

## 延伸阅读

- [包 README（中文）](../../packages/core/agent/README.zh.md)
- [agent-loop 包](../../packages/core/agent-loop/README.zh.md)
- [agent-loop 源码学习](../core-agent-loop/README.md)
- [Core 子系统](../../docs/subsystems/core.zh.md)
- [发起方作用域决策](../../.agents/notes/implemented/architecture/2026-07-15-agent-initiator-scope.zh.md)
