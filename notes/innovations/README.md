# OpenClaw 架构创新点分析

OpenClaw 基于 Pi Agent SDK（一个 coding agent 内核），在其上构建了一个完整的"全频道个人 AI 助手平台"。

Pi SDK 提供核心的 ReAct loop（prompt → LLM → tool call → tool result → repeat），OpenClaw 在其之上做了大量产品化和工程化工作。

## 创新点列表

| # | 创新点 | 一句话描述 | 详细文档 |
|---|--------|-----------|----------|
| 1 | [多频道路由层](./01-multi-channel-routing.md) | 把终端 coding agent 接入 WhatsApp/Telegram/Slack 等十几个 IM 平台 |
| 2 | [Session 管理 + 多 Agent](./02-session-multi-agent.md) | 多 agent、多频道、多群聊的 session 拓扑与路由 |
| 3 | [生产级容错（外层循环）](./03-production-resilience.md) | Auth 轮转、模型 failover、context overflow 压缩、thinking 降级 |
| 4 | [主动触发机制](./04-active-triggers.md) | Heartbeat / Cron / Followup Queue，让被动 LLM 变成主动助手 |
| 5 | [Gateway 即服务](./05-gateway-as-service.md) | 本地常驻 daemon，开机自启，macOS menubar app / launchd / systemd |
| 6 | [工具生态扩展](./06-extended-tools.md) | 在 coding tools 之上加 message/browser/canvas/cron/nodes 等助手工具 |
| 7 | [安全与隔离](./07-security-isolation.md) | Sandbox、多层工具 policy、owner-only、群聊权限控制 |
| 8 | [插件系统](./08-plugin-system.md) | 37 个扩展频道、plugin SDK、社区可扩展 |

## 架构全景

```
┌─────────────── OpenClaw 上层 ───────────────┐
│                                              │
│  [1] 多频道路由   [2] Session/多Agent         │
│  [4] 触发机制     [5] Gateway daemon          │
│  [6] 扩展工具     [7] 安全隔离                 │
│  [8] 插件系统                                 │
│                                              │
│  ┌─── [3] 外层循环 (容错) ───┐               │
│  │  runEmbeddedPiAgent()     │               │
│  │  重试/failover/compaction │               │
│  └────────────┬──────────────┘               │
│               │                              │
├───────────────┼──────────────────────────────┤
│               ▼                              │
│  ┌────────────────────────┐                  │
│  │   Pi Agent SDK (黑盒)  │                  │
│  │   session.prompt()     │                  │
│  │   ReAct loop           │                  │
│  └────────────────────────┘                  │
└──────────────────────────────────────────────┘
```

## 讨论记录

后续对各创新点的深入讨论会总结到对应的子文档中。
