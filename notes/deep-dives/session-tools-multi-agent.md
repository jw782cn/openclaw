# Session 工具与多 Agent 协作

## 核心思想

这组工具让 agent 不再是"一个孤立的对话"，而是能**感知、通信、协调**多个 session 和子 agent 的协作系统。

## 工具清单

| 工具 | 用途 | 一句话 |
|------|------|--------|
| sessions_list | 列出存在的 session | "我有哪些对话在进行？" |
| sessions_history | 读取另一个 session 的记录 | "群里刚才聊了什么？" |
| sessions_send | 往另一个 session 发消息 | "告诉 ops agent 部署完了" |
| sessions_spawn | 创建子 agent 后台执行任务 | "开一个子 agent 去分析这个 PR" |
| subagents | 管理正在跑的子 agent | "列出/指挥/杀死子 agent" |
| agents_list | 列出可用的 agent id | "有哪些 agent 可以 spawn？" |
| session_status | 查看当前 session 状态 | "用的什么模型？花了多少 token？" |

## 逐个详解

### sessions_list

```typescript
sessions_list(kinds=["subagent"], activeMinutes=60, limit=10)
```

列出 session，可按类型过滤（dm/group/subagent）、按活跃时间过滤、限制数量。返回 session key、最后更新时间、最近消息预览等。

有可见性控制：沙箱里的 agent 只能看到自己 spawn 的子 session。

### sessions_history

```typescript
sessions_history(sessionKey="agent:main:telegram:group:123", limit=10)
```

读取另一个 session 的对话记录。返回消息数组（role + content），单条文本上限 4000 字符，总量上限 80KB。

用途举例：你在 DM 里问"刚才群里聊了什么"，agent 用这个工具去读群聊历史。

### sessions_send

```typescript
sessions_send(sessionKey="agent:ops:main", message="开始部署到 staging")
```

往另一个 session 发消息。这条消息会被当作 user message 注入到目标 session，触发目标 agent 执行一轮。

支持：
- 跨 agent 通信（main agent → ops agent）
- 同 agent 跨 session 通信（DM session → 群聊 session）
- Agent-to-Agent (A2A) 协议流

**注意**：如果往自己的 session 发消息，会形成自我循环。

### sessions_spawn

```typescript
sessions_spawn(
  task="分析 PR #42 的代码变更，总结主要修改",
  model="claude-4.6-opus",
  thinking="high",
  runTimeoutSeconds=300
)
```

在后台创建一个**隔离 session**，用独立的上下文跑一个完整 agent turn。

关键特性：
- **异步执行**：spawn 立即返回，不阻塞主 agent
- **Push-based 完成通知**：子 agent 完成后自动 announce 结果回主 session
- **不需要轮询**：system prompt 里明确说 "do not poll subagents list in a loop"
- **可选模型/thinking**：子 agent 可以用不同模型和 thinking level
- **可选清理**：`cleanup: "delete"` 完成后删除 session，`"keep"` 保留

### subagents

```typescript
subagents(action="list")    // 列出正在跑的子 agent
subagents(action="steer", sessionKey="...", message="改成只看最近 7 天的")  // 给子 agent 发指令
subagents(action="kill", sessionKey="...")   // 终止子 agent
```

管理层工具，控制已经 spawn 的子 agent。

### agents_list

```typescript
agents_list()
→ [{ id: "main", default: true }, { id: "ops" }, { id: "work" }]
```

列出配置中可用的 agent id。spawn 子 agent 时可以指定 `agentId` 让它跑在不同 agent 的 workspace 里。

### session_status

```typescript
session_status()
→ {
    model: "claude-4.6-opus",
    thinking: "high",
    usage: { inputTokens: 8000, outputTokens: 2000 },
    time: "2026-02-19T10:30:00+08:00",
    ...
  }
```

元数据查询。Agent 用这个回答"我们在用什么模型"、"花了多少 token"、"现在几点"等问题。

## 多 Agent 协作模式

### 模式 1：并行分发

```
用户: "帮我分析这三个 PR"

主 Agent:
  ├─ sessions_spawn(task="分析 PR #1")  → 子 Agent A
  ├─ sessions_spawn(task="分析 PR #2")  → 子 Agent B
  ├─ sessions_spawn(task="分析 PR #3")  → 子 Agent C
  │
  │  三个子 agent 并行跑，完成后各自 announce 结果
  │
  └─ 主 agent 收到三个结果，汇总给用户
```

### 模式 2：跨 Agent 委托

```
用户（在 main agent DM 里）: "让 ops agent 部署到 staging"

Main Agent:
  └─ sessions_send(agentId="ops", message="部署到 staging 环境")
       │
       ▼
     Ops Agent（独立 workspace、独立工具集）:
       └─ 执行部署任务，完成后通知
```

### 模式 3：上下文收集

```
用户: "总结一下今天各个群里讨论的内容"

Agent:
  ├─ sessions_list(kinds=["group"], activeMinutes=1440)
  │  → 得到今天活跃的群聊 session 列表
  │
  ├─ sessions_history(sessionKey="...群聊A...", limit=50)
  ├─ sessions_history(sessionKey="...群聊B...", limit=50)
  ├─ sessions_history(sessionKey="...群聊C...", limit=50)
  │
  └─ 汇总三个群的历史，给用户摘要
```

## 安全与可见性

### Session 可见性控制

不是所有 session 都对所有 agent 可见：
- **沙箱内的 agent**：只能看到自己 spawn 的子 session（`restrictToSpawned`）
- **Agent-to-Agent 策略**：不同 agent 之间有跨访问控制（`createAgentToAgentPolicy`）
- **可见性配置**：通过 `resolveEffectiveSessionToolsVisibility()` 按配置决定

### 子 Agent 继承限制

- 子 agent 继承父 agent 的沙箱（不能逃逸）
- 子 agent 的工具集受 subagent tool policy 限制
- 子 agent 不能 spawn 无限层级的子子 agent（有深度限制）

## 关键代码位置

| 文件 | 作用 |
|------|------|
| `src/agents/tools/sessions-list-tool.ts` | sessions_list 实现 |
| `src/agents/tools/sessions-history-tool.ts` | sessions_history 实现 |
| `src/agents/tools/sessions-send-tool.ts` | sessions_send 实现 |
| `src/agents/tools/sessions-spawn-tool.ts` | sessions_spawn 实现 |
| `src/agents/tools/subagents-tool.ts` | subagents 实现 |
| `src/agents/tools/agents-list-tool.ts` | agents_list 实现 |
| `src/agents/tools/session-status-tool.ts` | session_status 实现 |
| `src/agents/tools/sessions-helpers.ts` | 共享的可见性、解析、安全逻辑 |
| `src/agents/subagent-spawn.ts` | spawn 子 agent 的核心逻辑 |
| `src/agents/tools/sessions-send-tool.a2a.ts` | Agent-to-Agent 协议流 |

## sessions_spawn 的实现流程

代码在 `src/agents/subagent-spawn.ts` → `spawnSubagentDirect()`。

### 完整流程

```
sessions_spawn(task="分析 PR #42", model="claude-4.6-opus", thinking="high")
  │
  ▼
Step 1: 安全检查
  ├─ 检查 spawn 深度限制（默认 maxSpawnDepth=1，不能无限嵌套）
  │   callerDepth >= maxSpawnDepth → forbidden
  │
  ├─ 检查并发子 agent 数量限制（默认 maxChildrenPerAgent=5）
  │   activeChildren >= maxChildren → forbidden
  │
  └─ 检查跨 agent 权限（如果 target agentId ≠ requester agentId）
      需要在 allowAgents 配置里明确允许
  │
  ▼
Step 2: 生成唯一的子 session key
  childSessionKey = "agent:<targetAgentId>:subagent:<uuid>"
  │
  ▼
Step 3: 通过 Gateway RPC 创建子 session
  callGateway({ method: "sessions.patch", params: {
    key: childSessionKey,
    spawnDepth: callerDepth + 1
  }})
  │
  ▼
Step 4: 配置子 session 的模型和 thinking level（如果指定了）
  callGateway({ method: "sessions.patch", params: {
    key: childSessionKey,
    model: "claude-4.6-opus"
  }})
  callGateway({ method: "sessions.patch", params: {
    key: childSessionKey,
    thinkingLevel: "high"
  }})
  │
  ▼
Step 5: 构建子 agent 的 system prompt 和任务消息
  buildSubagentSystemPrompt({
    requesterSessionKey,    // 谁 spawn 的
    childSessionKey,        // 子 agent 自己的 key
    task,                   // 任务描述
    childDepth,             // 当前深度
    maxSpawnDepth,          // 最大深度
  })

  childTaskMessage =
    "[Subagent Context] You are running as a subagent (depth 1/1).
     Results auto-announce to your requester; do not busy-poll for status."
    +
    "[Subagent Task]: 分析 PR #42 的代码变更"
  │
  ▼
Step 6: 通过 Gateway RPC 触发子 agent 执行
  callGateway({ method: "agent", params: {
    message: childTaskMessage,
    sessionKey: childSessionKey,
    extraSystemPrompt: childSystemPrompt,
    deliver: false,              // 不直接投递到频道（由 announce 机制处理）
    lane: "subagent",            // 放在 subagent lane 执行
    spawnedBy: requesterKey,     // 记录父子关系
    thinking: "high",
    timeout: runTimeoutSeconds,
  }})
  │
  ▼
Step 7: 注册到子 agent 注册表
  registerSubagentRun({
    runId, childSessionKey, requesterSessionKey,
    task, cleanup, label, model, ...
  })
  │
  ▼
Step 8: 立即返回（不等子 agent 完成）
  return {
    status: "accepted",
    childSessionKey: "agent:main:subagent:abc-123",
    runId: "xyz-456",
    note: "auto-announces on completion, do not poll/sleep"
  }
```

### 关键设计细节

**不是本地 fork/spawn**：子 agent 不是直接在当前进程里开一个新线程跑。而是通过 Gateway RPC（`callGateway({ method: "agent" })`）发一个请求，让 Gateway 去排队执行。Gateway 有自己的 command queue 和 lane 机制。

**Lane 隔离**：子 agent 跑在 `AGENT_LANE_SUBAGENT` lane 上，和主 session 的 lane 分开，不会互相阻塞。

**deliver: false**：子 agent 的直接输出不投递到用户的聊天频道。完成后通过 announce 机制（`src/agents/subagent-announce.ts`）把结果注入到父 session，让父 agent 看到。

**深度限制**：默认 `maxSpawnDepth=1`，即子 agent 不能再 spawn 子子 agent。防止 agent 递归创建无限层级。

**并发限制**：每个 session 最多 `maxChildrenPerAgent=5` 个活跃子 agent。防止资源耗尽。

**跨 agent 权限**：main agent 想 spawn 一个跑在 ops agent workspace 里的子 agent，需要配置里 `allowAgents: ["ops"]` 明确允许。

## 讨论记录

（待补充）
