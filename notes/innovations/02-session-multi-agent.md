# 创新点 2：Session 管理 + 多 Agent

## 核心思想

一个用户可以有多个 agent（如 main、work、ops），每个 agent 有独立的 workspace 和 session 存储。不同频道、不同群聊、不同话题映射到不同的 session key，形成一套 session 拓扑结构。

## Session Key 格式

```
agent:<agentId>:main                              — DM 主 session
agent:<agentId>:<channel>:direct:<peerId>         — 某频道的 DM
agent:<agentId>:<channel>:group:<groupId>         — 群聊
agent:<agentId>:<channel>:channel:<id>:thread:<t> — 话题/线程
```

## Agent 结构

每个 agent 有独立的：
- `~/.openclaw/agents/<agentId>/workspaces/` — 工作目录
- `~/.openclaw/agents/<agentId>/sessions/` — Session JSONL 文件
- `~/.openclaw/agents/<agentId>/auth.json` — 认证配置

## 关键代码位置

- Agent 解析: `src/agents/agent-scope.ts`
- Agent 路径: `src/agents/agent-paths.ts`
- Session 管理: `src/config/sessions.ts`
- Session key 路由: `src/routing/session-key.ts`
- 子 agent 生成: `src/agents/tools/sessions-spawn-tool.ts`

## 设计要点

- Session 数据为 JSONL 格式，由 Pi SDK 的 `SessionManager` 管理
- Session write lock 保证同一个 session 不会并发写入
- 子 agent（subagent）可以被 spawn，有独立的 session 和 workspace

## 讨论记录

（待补充）
