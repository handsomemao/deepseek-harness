# 源码学习 wiki

这里按包整理 DeepSeek Harness 的源码阅读笔记。先理解运行时对象，再沿创建、输入、模型请求和工具执行阅读；公开接口与配置以对应包的 README 和源码为准。

## 专题

| 专题 | 适合解决的问题 |
|---|---|
| [core/agent](core-agent/README.md) | Agent 的公开接口、注册表、事件和工厂约定是什么？ |
| [core/agent-loop](core-agent-loop/README.md) | 默认驱动器如何创建 Agent，处理输入，调用模型，执行工具并收尾？ |

## 先区分四个概念

一个 `AgentLoop` 服务可以管理多个 Agent 实例；一个活跃 Agent 实例固定驱动一个 Session。一个 Session 保存多个 Turn，每个 Turn 可以包含多个 Step。同一个 Agent preset 可以用于创建多个实例，它不是正在执行的 Agent 对象。

持久化 Session 可以跨进程保留，Agent 对象和它的作用域需要在恢复时重新构造。详细关系见 [agent-loop 概览](core-agent-loop/README.md#运行时对象关系)。
