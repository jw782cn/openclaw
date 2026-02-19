# 创新点 1：多频道路由层

## 核心思想

把一个只能在终端里跑的 coding agent 接入十几个 IM 平台，让用户在任何聊天工具里都能跟同一个 AI 助手对话。

## 支持的频道

### 内置频道（`src/` 下）
- WhatsApp (`src/whatsapp`, `src/web`)
- Telegram (`src/telegram`)
- Slack (`src/slack`)
- Discord (`src/discord`)
- Signal (`src/signal`)
- iMessage (`src/imessage`)
- WebChat (`src/channel-web.ts`)

### 扩展频道（`extensions/` 下，共 37 个）
- Microsoft Teams, Matrix, Google Chat, Zalo, Line, Feishu (飞书), IRC, Mattermost, Nostr, Twitch, BlueBubbles 等

## 关键代码位置

- 频道适配器: `src/telegram/`, `src/discord/`, `src/slack/` 等
- 路由核心: `src/routing/`
- 频道抽象: `src/channels/`
- 消息投递: `src/infra/outbound/`

## 设计要点

- 每个频道有自己的适配器，处理该平台特有的消息格式、API 调用、webhook 等
- 统一的路由层将不同频道的消息映射到统一的 session key
- 回复时通过 delivery pipeline 路由回消息来源的频道

## 讨论记录

### 频道层并非纯外部壳，与 agent 核心有三处耦合

频道信息不只是在"最外层"做消息收发，它渗透到了 agent 核心的三个地方：

1. **System Prompt 依赖频道**：当前频道名、频道能力（inlineButtons/reactions 等）、频道特有的 message tool 提示都会写入 system prompt。Agent 知道自己"在哪个平台上"。

2. **工具集依赖频道**：每个频道可以注册专属工具（channel agent tools），群聊和 DM 有不同的工具 policy。工具可用性因频道而异。

3. **工具行为依赖频道**：message tool、cron tool 等需要 messageProvider/groupId/accountId 等信息来正确路由消息。这些参数从频道适配器一路透传到工具执行层。

但 Pi SDK 本身完全不感知频道。所有频道信息都是 OpenClaw 在调用 `session.prompt()` 之前通过 system prompt 和 tool 配置注入的。对 Pi SDK 来说，所有频道的消息都只是"一段文字 + 一组工具"。

### 为什么频道会导致工具不同？

核心原因是**安全**。Agent 暴露在群聊中时，群里任何人都可能触发它。不能让陌生人通过群聊让 agent 执行 `exec rm -rf /`。所以群聊可以通过 `resolveGroupToolPolicy()` 配置工具白名单，只暴露安全的只读工具（read/grep/web_search 等），而 DM 里 owner 拥有完整工具集。

### 为什么 agent 需要在 system prompt 中感知频道？

让 agent 了解当前频道的能力，避免浪费 tool call 做不支持的事。例如：
- Telegram 支持 inline buttons，Signal 不支持 → prompt 里告知是否可用
- Telegram 支持 emoji reaction（minimal/extensive 模式），其他频道可能不支持 → prompt 里注入 reaction 指导
- 不同频道的 message tool 用法有差异 → prompt 里注入频道特有的 hints

### 频道能力限制是两层机制（不只是 prompt 提示）

1. **Tool Schema 层（硬限制）**：message tool 的 JSON Schema 是动态构建的。如果当前频道不支持 inline buttons，`buttons` 参数直接从 schema 里 `delete` 掉，LLM 看不到这个字段，无法生成。同理 `card`（Teams 专属）和 `components`（Discord 专属）。代码：`message-tool.ts` 的 `buildSendSchema()` + `supportsChannelMessageButtonsForChannel()`。

2. **System Prompt 层（软提示/兜底）**：额外在 prompt 中说明 "inline buttons not enabled for <channel>"，作为双保险，防止 LLM hallucinate 出不存在的参数。

### Cache 影响（设计 trade-off）

频道信息写入 system prompt 确实会影响 Anthropic prompt cache。但影响可控：
1. 同一 session 始终在同一频道（session key 编码了频道），不会频繁切换
2. 频道相关内容在 system prompt 靠后位置，前缀的大部分（Safety/Tooling/Skills/Workspace/Context）不变，Anthropic 前缀匹配 cache 仍能命中大部分
3. 这是一个 trade-off：用少量 cache 效率换取 agent 在不同频道上的行为正确性
