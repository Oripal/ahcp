# AHCP — Agent Heartbeat & Control Protocol

An open protocol for lifecycle monitoring, health detection, and control signal propagation in AI-native agent platforms.

一个面向 AI 原生 Agent 平台的开放协议，定义生命周期监控、健康检测和控制信号传播。

---

## Why AHCP? | 为什么需要 AHCP？

The AI agent ecosystem has protocols for tool access (MCP), agent-to-agent communication (A2A), and agent messaging (ACP). But **none of them define how to keep agents and their tasks alive, detect failures, propagate cancellation, or recover from crashes**.

AI Agent 生态已有工具连接协议（MCP）、Agent 间通信协议（A2A）和 Agent 消息协议（ACP），但**没有任何协议定义如何保持 Agent 和任务的存活检测、故障发现、取消传播和崩溃恢复**。

AHCP fills this gap. | AHCP 填补了这一空白。

## What AHCP Defines | AHCP 定义了什么

- **Dual-layer heartbeat** | **双层心跳**: Worker process liveness + individual job health — two independent concerns, two independent protocols | Worker 进程存活 + 单个 Job 健康——两个独立关注点，两套独立协议

- **Heartbeat as control channel** | **心跳即控制通道**: Cancel, drain, suspend signals delivered through heartbeat responses — no separate control channel needed | 取消、排空、挂起信号通过心跳响应传递——无需额外控制通道

- **AI-native timeout classification** | **AI 原生超时分类**: Different timeout strategies for LLM calls, tool execution, sandbox sessions, and human-in-the-loop waits | 针对 LLM 调用、工具执行、沙箱会话、人机交互等不同场景的超时策略

- **Job type policies** | **Job 类型策略**: Heartbeat interval, crash recovery strategy, and cancellation behavior configured per job type | 心跳间隔、崩溃恢复策略、取消行为按 Job 类型配置

- **Multi-agent topology** | **多 Agent 拓扑**: Heartbeat semantics for solo, pipeline, team, and simulation execution modes | 面向 Solo/Pipeline/Team/Simulation 四种执行模式的心跳语义

- **Integration points** | **集成点**: Works alongside A2A (task lifecycle), MCP (tool health), and OpenTelemetry (observability) | 与 A2A（任务生命周期）、MCP（工具健康）、OpenTelemetry（可观测性）协同工作

## Architecture | 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    AI Agent Platform                         │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │   MCP    │  │   A2A    │  │   ACP    │  │ AGENTS.md │  │
│  │ Tool     │  │ Agent ↔  │  │ Agent    │  │ Behavior  │  │
│  │ Access   │  │ Agent    │  │ Comms    │  │ Guidance  │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────────┘  │
│       └──────────────┼──────────────┘                        │
│              ┌───────┴───────┐                               │
│              │     AHCP      │  ← Liveness & Control Layer  │
│              └───────┬───────┘                               │
│              ┌───────┴───────┐                               │
│              │ OpenTelemetry │  ← Observability Layer        │
│              └───────────────┘                               │
└─────────────────────────────────────────────────────────────┘
```

## Status | 状态

**v1.0-rc1** — Release candidate. Feedback welcome. | 发布候选版。欢迎反馈。

## Specification | 规范

- [AHCP Specification](spec/AHCP_Specification.md) — Full protocol specification (English) | 完整协议规范（英文）
- [Proto Definitions](proto/ahcp/v1/ahcp.proto) — gRPC service and message definitions | gRPC 服务和消息定义

## Design Principles | 设计原则

| # | Principle | 原则 |
|---|-----------|------|
| P-1 | Dual-layer separation | 双层分离：Worker 存活与 Job 健康独立 |
| P-2 | Heartbeat as control channel | 心跳即控制通道 |
| P-3 | Cooperative cancellation | 协作式取消：请求而非强杀 |
| P-4 | Policy over code | 策略优于代码：按 Job 类型配置 |
| P-5 | AI-native timeout taxonomy | AI 原生超时分类 |
| P-6 | Zero-overhead default | 零开销默认：无心跳 Job 不付心跳代价 |
| P-7 | Transport agnostic | 传输无关：gRPC 主 / HTTP 备 |
| P-8 | Unidirectional control | 单向控制：Worker 永远是 caller |

## Conformance Levels | 一致性等级

| Level | Scope | 适用场景 |
|-------|-------|---------|
| **AHCP Core** | Worker lifecycle + Job lifecycle + Fencing tokens | Any distributed task platform | 任何分布式任务平台 |
| **AHCP Standard** | Core + Job heartbeat + Cancel + Job type policies | AI agent platforms | AI Agent 平台 |
| **AHCP Full** | Standard + AI-native extensions + OTel integration | Production AI agent platforms | 生产级 AI Agent 平台 |

## Reference Implementation | 参考实现

[VoborSpark](https://github.com/voborspark/voborspark) — AI-native agent platform, first AHCP implementation. | AI 原生 Agent 平台，首个 AHCP 实现。

## Contributing | 贡献

Issues and pull requests are welcome. For major changes, please open an issue first to discuss. | 欢迎提交 Issue 和 Pull Request。重大变更请先开 Issue 讨论。

## License | 许可

Apache License 2.0
