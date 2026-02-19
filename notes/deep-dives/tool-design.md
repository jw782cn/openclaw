# Tool 设计深度解析

## 一、Tool 的接口定义

所有 tool 都实现 Pi SDK 的 `AgentTool` 接口（来自 `@mariozechner/pi-agent-core`）：

```typescript
type AgentTool<TParams, TDetails> = {
  name: string;           // 工具名，LLM 调用时使用
  label?: string;         // 人类可读标签
  description: string;    // 工具描述，写给 LLM 看
  parameters: TSchema;    // JSON Schema，定义参数（用 @sinclair/typebox 构建）
  execute: (toolCallId: string, args: TParams, signal?: AbortSignal, onUpdate?: callback) => Promise<AgentToolResult<TDetails>>;
};
```

返回值 `AgentToolResult`：

```typescript
type AgentToolResult<T> = {
  content: Array<
    | { type: "text"; text: string }        // 文本结果
    | { type: "image"; data: string; mimeType: string }  // 图片结果（base64）
  >;
  details?: T;  // 结构化元数据（不发给 LLM，内部使用）
};
```

## 二、Schema 设计：用 TypeBox 构建 JSON Schema

所有 tool 用 `@sinclair/typebox` 的 `Type.Object()` 定义参数 schema：

```typescript
// 示例：web_search
const WebSearchSchema = Type.Object({
  query: Type.String({ description: "Search query string." }),
  count: Type.Optional(Type.Number({ minimum: 1, maximum: 10 })),
  country: Type.Optional(Type.String()),
});

// 示例：sessions_spawn
const SessionsSpawnToolSchema = Type.Object({
  task: Type.String(),
  label: Type.Optional(Type.String()),
  agentId: Type.Optional(Type.String()),
  model: Type.Optional(Type.String()),
  thinking: Type.Optional(Type.String()),
  runTimeoutSeconds: Type.Optional(Type.Number({ minimum: 0 })),
});
```

这些 TypeBox schema 在传给 LLM 之前会被转成标准 JSON Schema，不同 provider 还需要不同的清理：
- **Gemini**: 需要去掉某些 constraint 关键字（`cleanToolSchemaForGemini`）
- **OpenAI**: 不接受 root-level union schemas（`normalizeToolParameters`）
- **Anthropic OAuth**: tool 名字需要 remap 成 Claude Code 风格

## 三、Tool 的创建模式

每个 tool 是一个**工厂函数**，接收上下文配置，返回 `AnyAgentTool` 对象。

### 模式 A：简单工具（无外部依赖）

```typescript
export function createSessionsSpawnTool(opts?: { ... }): AnyAgentTool {
  return {
    name: "sessions_spawn",
    description: "Spawn a background sub-agent...",
    parameters: SessionsSpawnToolSchema,
    execute: async (_toolCallId, args) => {
      // 直接执行逻辑
      const result = await spawnSubagentDirect(...);
      return jsonResult(result);
    },
  };
}
```

### 模式 B：Gateway RPC 代理（通过 Gateway 执行）

很多 tool 不是直接执行，而是通过 Gateway RPC 调用：

```typescript
// cron tool 的 execute
case "status":
  return jsonResult(await callGatewayTool("cron.status", gatewayOpts, {}));
case "list":
  return jsonResult(await callGatewayTool("cron.list", gatewayOpts, { ... }));
```

`callGatewayTool()` 发 RPC 请求到本地 Gateway 进程。这意味着 tool 实际执行在 Gateway 端，agent 只是个 RPC 客户端。

### 模式 C：动态 Schema（根据上下文变化）

最复杂的是 `message` tool，schema 根据频道能力动态构建：

```typescript
function resolveMessageToolSchema(params) {
  const includeButtons = currentChannel
    ? supportsChannelMessageButtonsForChannel({ cfg, channel: currentChannel })
    : supportsChannelMessageButtons(cfg);
  // includeButtons=false 时，buttons 字段从 schema 里 delete 掉
  return buildMessageToolSchemaFromActions(actions, { includeButtons, includeCards, ... });
}
```

## 四、Tool 的生命周期（从定义到执行）

```
1. 工厂创建
   createOpenClawCodingTools() 调用各 create*Tool() 工厂
   ↓
2. Policy 过滤
   applyOwnerOnlyToolPolicy()     → 非 owner 移除敏感工具
   applyToolPolicyPipeline()      → 多层策略(profile/provider/agent/group/sandbox)
   ↓
3. Schema 标准化
   normalizeToolParameters()      → 修复 JSON Schema 兼容性
   cleanToolSchemaForGemini()     → Gemini 特殊处理
   ↓
4. Hook 包装
   wrapToolWithBeforeToolCallHook() → 注入 before_tool_call 钩子
   wrapToolWithAbortSignal()        → 注入 abort signal 支持
   ↓
5. 适配器转换
   toToolDefinitions()            → AgentTool → Pi SDK ToolDefinition
   （处理新旧 API 参数顺序差异）
   ↓
6. 注入 session
   createAgentSession({ tools: builtInTools, customTools: allCustomTools })
   ↓
7. LLM 调用
   Pi SDK 把 tool schema 序列化给 LLM → LLM 返回 tool_call → Pi SDK 调 execute()
   ↓
8. 执行 + Hook
   before_tool_call hook → execute() → after_tool_call hook
   错误时返回 { status: "error", tool: name, error: message }（不 throw，优雅降级）
```

## 五、Tool 返回值的两种格式

### jsonResult — 结构化 JSON（大多数工具）

```typescript
function jsonResult(payload: unknown): AgentToolResult<unknown> {
  return {
    content: [{ type: "text", text: JSON.stringify(payload, null, 2) }],
    details: payload,
  };
}
```

### imageResult — 图片（browser screenshot、canvas snapshot、image tool）

```typescript
function imageResult(params): AgentToolResult<unknown> {
  return {
    content: [
      { type: "text", text: `MEDIA:${params.path}` },
      { type: "image", data: params.base64, mimeType: params.mimeType },
    ],
    details: { path: params.path },
  };
}
```

## 六、错误处理：不 throw，返回错误 JSON

Tool adapter 层（`pi-tool-definition-adapter.ts`）catch 所有异常，转为错误结果返回给 LLM：

```typescript
catch (err) {
  return jsonResult({
    status: "error",
    tool: normalizedName,
    error: described.message,
  });
}
```

这样 LLM 看到的不是 crash，而是 `{ "status": "error", "error": "..." }`，它可以据此调整行为（比如换个参数重试）。

## 七、安全相关的外部内容包装

browser 和 web_fetch 返回的内容可能包含恶意 prompt injection。OpenClaw 用 `wrapExternalContent()` 包装：

```typescript
const wrappedText = wrapExternalContent(extractedText, {
  source: "browser",
  includeWarning: true,
});
```

这会在外部内容前后加上安全标记，提醒 LLM 这是不可信的外部输入。

## 八、Tool 分类总览

| 类别 | 工具 | 执行方式 |
|------|------|---------|
| 文件操作 | read, write, edit, grep, find, ls | Pi SDK 内置，直接文件系统操作 |
| Shell 执行 | exec, process | OpenClaw 自建，支持 PTY/后台/沙箱 |
| 补丁 | apply_patch | OpenClaw 自建，仅 OpenAI/Codex 模型 |
| 网络 | web_search, web_fetch | 直接 HTTP 调用（Brave/Perplexity/Grok API） |
| 浏览器 | browser | Gateway RPC → Playwright |
| UI | canvas | Gateway RPC → WebView/Canvas |
| 设备 | nodes | Gateway RPC → 配对设备 |
| 定时 | cron | Gateway RPC → CronService |
| 消息 | message | Gateway RPC → 频道适配器 |
| 语音 | tts | 直接 TTS API |
| 系统 | gateway | Gateway RPC → 自我管理 |
| Session | sessions_list/history/send/spawn, subagents, agents_list | Gateway RPC / 直接 |
| 状态 | session_status | Gateway RPC |
| 图像 | image | 直接调用图像模型 API |
| 记忆 | memory_search, memory_get | 文件系统 + 向量搜索 |
| 插件 | (动态) | 由插件定义 |

## 讨论记录

（待补充）
