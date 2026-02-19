# 创新点 4：主动触发机制

## 核心思想

LLM agent 本质是被动的（有人调 `session.prompt()` 才执行）。OpenClaw 通过三套机制让 agent 能"主动行动"。

## 三套触发机制

### 1. Heartbeat（心跳）

- **用途**: 定期检查有没有事要做
- **默认间隔**: 30 分钟
- **流程**:
  1. Gateway 定时器触发
  2. 检查 `HEARTBEAT.md` 是否为空（为空则跳过，省 API 调用）
  3. 向 agent 注入 prompt: "Read HEARTBEAT.md, if nothing needs attention reply HEARTBEAT_OK"
  4. Agent 回复 HEARTBEAT_OK → 静默丢弃 + 裁剪对话记录
  5. Agent 回复实际内容 → 投递到配置的频道
- **去重**: 同样的回复 24 小时内不重复发
- **也用于**: exec 后台命令完成的通知、cron systemEvent 的投递
- **代码**: `src/infra/heartbeat-runner.ts`, `src/auto-reply/heartbeat.ts`

### 2. Cron（定时任务）

- **用途**: 按计划执行具体任务
- **调度方式**: `at`（一次性）、`every`（固定间隔）、`cron`（cron 表达式）
- **Payload 类型**:
  - `systemEvent`: 往主 session 注入系统消息（等 heartbeat 处理）
  - `agentTurn`: 在隔离 session 里启动完整 agent run
- **Delivery**: announce（发到聊天）、webhook（HTTP POST）、none（静默）
- **代码**: `src/cron/service.ts`, `src/agents/tools/cron-tool.ts`, `src/cron/isolated-agent.ts`

### 3. Followup Queue（后续消息队列）

- **用途**: 处理当前 agent run 产生的连锁动作
- **触发场景**: sessions_send 跨 session 消息、长回复分块发送、工具产生的后续动作
- **不是定时的**: 事件驱动
- **代码**: `src/auto-reply/reply/followup-runner.ts`

## 三者对比

| | Heartbeat | Cron | Followup Queue |
|---|---|---|---|
| 触发方式 | 定时器（默认 30m） | 定时调度 | 事件驱动 |
| Session | 主 session | 主或隔离 | 继承触发源 |
| 调 API | 可能跳过 | 必调 | 必调 |
| 典型场景 | "定期检查" | "每天9点做XX" | "做完A触发B" |

## 共同入口

三者最终都调用 `runEmbeddedPiAgent()`，对 Pi SDK 来说没有区别。

## 讨论记录

（待补充）
