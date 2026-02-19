# Message Tool 设计深度解析

## 一、设计目标

用**一个** tool（`message`）统一所有频道的所有消息操作。Agent 不需要知道"Telegram 怎么发消息"和"Discord 怎么发消息"的区别，只需要调 `message(action="send", channel="telegram", target="123", message="hello")`。

## 二、架构分层

```
┌──────────────────────────────────────────────────┐
│  LLM 看到的                                       │
│  message tool (单一工具，动态 schema)               │
│    action: send | react | poll | pin | ...        │
│    channel: telegram | discord | slack | ...      │
│    target: "user123" / "group456"                 │
│    message: "hello"                               │
│    buttons?: [...] (频道支持时才出现)               │
└──────────────────┬───────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────┐
│  message-tool.ts                                  │
│  - 动态构建 schema (buildMessageToolSchema)        │
│  - 路由到 runMessageAction()                       │
└──────────────────┬───────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────┐
│  message-action-runner.ts                         │
│  - 解析频道 + target                               │
│  - 分发到对应 channel plugin                       │
│  - 处理跨上下文策略(cross-context decoration)       │
└──────────────────┬───────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────┐
│  Channel Plugin (ChannelMessageActionAdapter)     │
│  每个频道实现自己的:                                │
│    listActions()      → 声明支持哪些 action        │
│    supportsButtons()  → 是否支持 inline buttons    │
│    supportsCards()    → 是否支持 Adaptive Cards     │
│    handleAction()     → 实际执行动作                │
└──────────────────────────────────────────────────┘
```

## 三、53 种 Action（全量清单）

定义在 `src/channels/plugins/message-action-names.ts`：

| 类别 | Actions |
|------|---------|
| 发送 | send, broadcast, sendAttachment, sendWithEffect |
| 回复 | reply, thread-reply |
| 编辑/删除 | edit, unsend, delete |
| 表情 | react, reactions |
| 读取 | read, search |
| 置顶 | pin, unpin, list-pins |
| 投票 | poll |
| 线程 | thread-create, thread-list, thread-reply |
| 群管理 | renameGroup, setGroupIcon, addParticipant, removeParticipant, leaveGroup |
| 权限 | permissions |
| 频道管理 | channel-info, channel-list, channel-create, channel-edit, channel-delete, channel-move |
| 分类 | category-create, category-edit, category-delete |
| 话题 | topic-create |
| 成员/角色 | member-info, role-info, role-add, role-remove |
| Emoji/Sticker | emoji-list, emoji-upload, sticker, sticker-search, sticker-upload |
| 语音 | voice-status |
| 活动 | event-list, event-create |
| 管理 | timeout, kick, ban |
| 状态 | set-presence |

不是所有频道都支持所有 action。每个频道 plugin 通过 `listActions()` 声明自己支持哪些。

## 四、动态 Schema 的三层适配

### 第 1 层：Action 列表根据频道裁剪

```typescript
function resolveMessageToolSchemaActions(params) {
  const currentChannel = normalizeMessageChannel(params.currentChannelProvider);
  if (currentChannel) {
    // 当前频道明确时，只暴露该频道支持的 actions
    const scopedActions = listChannelSupportedActions({ cfg, channel: currentChannel });
    return ["send", ...scopedActions]; // send 永远保留
  }
  // 无明确频道时，合并所有已配置频道的 actions
  return listChannelMessageActions(cfg);
}
```

**效果**：如果当前在 Signal 里，LLM 只看到 `send | react | read | delete` 等 Signal 支持的 action，看不到 Discord 的 `channel-create`、`emoji-upload` 等。

### 第 2 层：参数字段根据频道能力删除

```typescript
// buttons: 只有 Telegram 等支持 inline keyboard 的频道才保留
if (!options.includeButtons) delete props.buttons;

// card: 只有 Microsoft Teams 等支持 Adaptive Cards 的频道才保留
if (!options.includeCards) delete props.card;

// components: 只有 Discord 才保留（buttons v2 / select menus / modals）
if (!options.includeComponents) delete props.components;
```

**效果**：LLM 在 Signal 频道里根本看不到 `buttons`、`card`、`components` 这些字段。

### 第 3 层：Description 根据频道变化

```typescript
function buildMessageToolDescription(options) {
  if (options?.currentChannel) {
    const channelActions = listChannelSupportedActions({ cfg, channel: currentChannel });
    return `...Current channel (${currentChannel}) supports: ${actionList}.`;
  }
  return `...Supports actions: send, delete, react, poll, pin, threads, and more.`;
}
```

**效果**：工具描述里明确告诉 LLM 当前频道支持什么。

## 五、Channel Plugin 的扩展接口

每个频道通过实现 `ChannelMessageActionAdapter` 来扩展 message tool 的能力：

```typescript
type ChannelMessageActionAdapter = {
  // 声明支持哪些 action
  listActions?: (params: { cfg: OpenClawConfig }) => ChannelMessageActionName[];

  // 查询是否支持某个 action
  supportsAction?: (params: { action: ChannelMessageActionName }) => boolean;

  // 是否支持 inline buttons (Telegram inline keyboard)
  supportsButtons?: (params: { cfg: OpenClawConfig }) => boolean;

  // 是否支持 Adaptive Cards (Teams)
  supportsCards?: (params: { cfg: OpenClawConfig }) => boolean;

  // 从 tool args 提取发送目标
  extractToolSend?: (params: { args: Record<string, unknown> }) => ChannelToolSend | null;

  // 执行实际动作（核心）
  handleAction?: (ctx: ChannelMessageActionContext) => Promise<AgentToolResult<unknown>>;
};
```

这个接口挂在 `ChannelPlugin.actions` 上：

```typescript
type ChannelPlugin = {
  // ... 其他适配器
  actions?: ChannelMessageActionAdapter;
  // ...
};
```

### 添加新频道的 message 支持

只需要：
1. 创建 channel plugin，实现 `actions.listActions()` 声明支持的 action
2. 实现 `actions.handleAction()` 处理具体逻辑
3. 可选实现 `supportsButtons()` / `supportsCards()` 声明 UI 能力

不需要改 message tool 本身的任何代码。Tool 会自动发现新频道的能力并调整 schema。

## 六、执行流程详解

```
LLM 调用: message(action="react", channel="telegram", emoji="👍", messageId="123")
  │
  ▼
message-tool.ts: createMessageTool().execute()
  │
  ├─ 参数预处理
  │  ├─ stripReasoningTagsFromText() — 清理 <think> 标签
  │  ├─ 解析 accountId (用哪个 bot 账号发)
  │  └─ 检查 requireExplicitTarget (群聊安全)
  │
  ▼
message-action-runner.ts: runMessageAction()
  │
  ├─ resolveMessageChannelSelection() — 确定目标频道
  │  （如果 LLM 没传 channel，从 session 上下文推断）
  │
  ├─ resolveChannelTarget() — 解析目标 (target → to)
  │  （处理各种格式: 电话号码、group id、channel name 等）
  │
  ├─ enforceCrossContextPolicy() — 跨上下文安全检查
  │  （防止 agent 在 A 频道冒充 B 频道身份发消息）
  │
  ├─ 路由到具体 action handler:
  │  ├─ action="send" → executeSendAction() → channel plugin
  │  ├─ action="poll" → executePollAction() → channel plugin
  │  └─ 其他 → dispatchChannelMessageAction() → plugin.actions.handleAction()
  │
  ▼
Channel Plugin 执行实际 API 调用 (Telegram API / Discord API / ...)
  │
  ▼
返回 AgentToolResult → 转成 JSON 给 LLM
```

## 七、频道特有的参数（Schema 组合式设计）

Schema 由多个 builder 函数组合而成，每个处理一类参数：

| Builder | 参数 | 适用场景 |
|---------|------|---------|
| buildRoutingSchema() | channel, target, targets, accountId, dryRun | 所有 action |
| buildSendSchema() | message, media, buffer, buttons, card, components, ... | send 类 |
| buildReactionSchema() | messageId, emoji, remove | react 类 |
| buildFetchSchema() | limit, before, after | read/search 类 |
| buildPollSchema() | pollQuestion, pollOption, pollDurationHours | poll |
| buildChannelTargetSchema() | channelId, guildId, userId, roleId | Discord 管理类 |
| buildStickerSchema() | emojiName, stickerId | sticker 类 |
| buildThreadSchema() | threadName, autoArchiveMin | thread 类 |
| buildEventSchema() | eventName, startTime, endTime, location | event 类 |
| buildModerationSchema() | reason, deleteDays | kick/ban/timeout |
| buildPresenceSchema() | activityType, activityName, status | set-presence |
| buildChannelManagementSchema() | name, type, parentId, topic | channel CRUD |

所有这些都合并成一个 flat schema（`buildMessageToolSchemaProps()`）。这样 LLM 看到的是一个大的 `message` 工具，通过 `action` 字段区分操作类型，运行时校验哪些参数对当前 action 有效。

## 八、设计权衡

### 为什么用一个大 tool 而不是拆成多个小 tool？

**优点**：
- LLM 只需要记住一个 tool 名字
- 路由逻辑集中，频道切换简单
- 新频道不需要注册新 tool 名（只需要实现 plugin 接口）

**缺点**：
- Schema 很大，有很多对当前 action 无效的参数
- 不同 action 的参数混在一起，LLM 可能传错参数
- 运行时需要按 action 校验参数（而非 schema 级别强制）

### 为什么用 flat schema 而不是嵌套？

Tool schema 的 provider 兼容性问题（见 AGENTS.md 的 `tool-schema-guardrails`）：
- 不能用 `Type.Union`（`anyOf`/`oneOf`/`allOf` 被某些 validator 拒绝）
- 必须保持 `type: "object"` + `properties` 的简单结构
- 所以只能把所有参数 flatten 到一层，通过 `action` 字段在运行时判断

## 九、安全设计

### 跨上下文保护 (Cross-Context Policy)

防止 agent 在 Telegram 群聊里被人指示去 Discord 发消息：

```typescript
enforceCrossContextPolicy() // 检查是否允许跨频道操作
applyCrossContextDecoration() // 如果允许，给消息加上来源标记
```

### Reasoning Tag 清理

LLM 有时候在 tool 参数里带 `<think>...</think>` 标签，发送前必须清理：

```typescript
for (const field of ["text", "content", "message", "caption"]) {
  if (typeof params[field] === "string") {
    params[field] = stripReasoningTagsFromText(params[field]);
  }
}
```

### 显式目标要求

在某些上下文（如 cron 任务）中，要求 LLM 必须提供明确的 target，不能依赖隐式路由：

```typescript
if (requireExplicitTarget && actionNeedsExplicitTarget(action)) {
  if (!explicitTarget) {
    throw new Error("Explicit message target required for this run.");
  }
}
```

## 讨论记录

（待补充）
