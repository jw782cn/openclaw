# 性格系统（Personality System）

## 核心思想

OpenClaw 的 agent 性格不是硬编码的，而是通过 workspace 里的 **Markdown 文件**注入到 system prompt 中。用户可以编辑这些文件自定义性格，agent 自己也可以用 `write`/`edit` 工具修改它们来"进化"自己。

## 三层性格文件

```
workspace/
├── SOUL.md          ← 灵魂：性格、语气、价值观（最核心）
├── IDENTITY.md      ← 身份：名字、emoji、生物类型、氛围
├── AGENTS.md        ← 行为准则：什么时候说话、怎么反应、群聊礼仪
├── USER.md          ← 用户信息：主人是谁、偏好
└── MEMORY.md        ← 长期记忆（间接影响性格 — 积累的认知）
```

### SOUL.md — 灵魂文件

最关键的性格定义文件。默认模板的核心指引：

| 指引 | 含义 |
|------|------|
| "Be genuinely helpful, not performatively helpful." | 别说"Great question!"，直接帮忙 |
| "Have opinions." | 允许有偏好、觉得东西有趣或无聊 |
| "Be resourceful before asking." | 先自己搞，搞不定再问 |
| "Earn trust through competence." | 谨慎对待外部操作，大胆做内部操作 |
| "Remember you're a guest." | 能看到别人的隐私，要尊重 |

模板最后鼓励 agent 自我进化：
> *"This file is yours to evolve. As you learn who you are, update it."*

### IDENTITY.md — 身份卡片

结构化的元数据：

```markdown
- **Name:** Claw
- **Creature:** lobster AI
- **Vibe:** sharp, warm, slightly chaotic
- **Emoji:** 🦞
- **Avatar:** avatars/openclaw.png
```

被 `identity-file.ts` 解析为结构化数据：

```typescript
type AgentIdentityFile = {
  name?: string;     // 名字，用于消息前缀 [Claw]
  emoji?: string;    // 签名 emoji，用于 ack reaction
  theme?: string;    // 主题
  creature?: string; // 生物类型
  vibe?: string;     // 氛围/风格
  avatar?: string;   // 头像路径/URL
};
```

用途：
- **消息前缀**：群聊里发消息加 `[Claw]` 标识
- **Ack reaction**：收到消息用 🦞 回应（而不是默认 👀）
- **Avatar**：设置各频道头像
- **Response prefix**：回复消息的前缀标识

### AGENTS.md — 行为准则

定义 agent 在各种场景下的行为模式：
- 群聊里什么时候该说话、什么时候沉默
- 怎么用 emoji reaction（"React Like a Human!"）
- Heartbeat 时该做什么
- "Participate, don't dominate"
- 安全边界（"Don't exfiltrate private data. Ever."）

## 注入机制

### 作为 Bootstrap Files 读取

这些文件被统一归类为 **bootstrap files**（`src/agents/workspace.ts`）：

```typescript
const VALID_BOOTSTRAP_NAMES = new Set([
  "AGENTS.md",
  "SOUL.md",
  "TOOLS.md",
  "IDENTITY.md",
  "USER.md",
  "HEARTBEAT.md",
  "BOOTSTRAP.md",
]);
```

Agent run 开始时，`resolveBootstrapContextForRun()` 读取 workspace 下的这些文件。

### 注入到 System Prompt 的 Project Context 段

文件内容完整注入到 system prompt 的末尾：

```
# Project Context

The following project context files have been loaded:
If SOUL.md is present, embody its persona and tone.

## AGENTS.md
(AGENTS.md 全文)

## SOUL.md
(SOUL.md 全文)

## IDENTITY.md
(IDENTITY.md 全文)

## USER.md
(USER.md 全文)
```

### SOUL.md 有特殊的强化指令

代码里检测到 SOUL.md 存在时，额外注入一条指令（`system-prompt.ts` 第 584-593 行）：

```typescript
if (hasSoulFile) {
  lines.push(
    "If SOUL.md is present, embody its persona and tone. " +
    "Avoid stiff, generic replies; follow its guidance " +
    "unless higher-priority instructions override it.",
  );
}
```

关键词是 **embody**（化身），不是 "reference"（参考）。这告诉 LLM 要真正扮演这个性格，而不是把它当成参考资料。

## 性格更新机制

### Agent 用什么 tool 更新性格？

**没有专门的 personality tool**。Agent 直接用基础文件操作工具：

| 操作 | 使用的 tool | 示例 |
|------|------------|------|
| 修改 SOUL.md | `edit` | 微调语气描述、添加新价值观 |
| 重写 IDENTITY.md | `write` | 选了新名字/emoji |
| 追加 AGENTS.md | `edit` | 加了新的群聊规则 |
| 更新 MEMORY.md | `edit` | 记住了新的偏好（间接影响行为） |

### 更新时机

1. **首次对话（Onboarding）**：IDENTITY.md 模板里写了 "Fill this in during your first conversation"，agent 在第一次和用户聊天时会填写名字、emoji 等

2. **自我进化**：SOUL.md 模板鼓励 agent 修改自己："This file is yours to evolve. As you learn who you are, update it."

3. **Heartbeat 维护**：AGENTS.md 模板建议 agent 在 heartbeat 时定期回顾和更新记忆文件

4. **用户指令**：用户可以直接说"你以后说话简洁一点"，agent 可以去修改 SOUL.md 来落实

### 修改后何时生效？

**下一次 agent run 生效**。因为 bootstrap files 在每次 `runEmbeddedAttempt()` 开始时重新读取。当前对话中修改不会立即改变 system prompt（system prompt 在 session 创建时已经固定），但下一轮对话（或下一次 heartbeat/cron 触发）就会使用新内容。

### SOUL.md 修改的安全约定

默认模板里有一条：
> "If you change this file, tell the user — it's your soul, and they should know."

这是 prompt 层面的约定，agent 修改自己的灵魂文件时应该通知用户。不是代码强制的，但 LLM 通常会遵守。

## IDENTITY.md 的代码级使用

IDENTITY.md 不只是注入到 prompt，还被代码解析用于实际功能（`identity-file.ts` + `identity.ts`）：

```
IDENTITY.md
  │
  ├─ parseIdentityMarkdown() → AgentIdentityFile { name, emoji, vibe, ... }
  │
  ├─ resolveAckReaction() → 收到消息时的 reaction emoji
  │   优先级: channel account → channel → global messages → identity emoji → 默认 👀
  │
  ├─ resolveMessagePrefix() → 发消息时的前缀
  │   如 name="Claw" → 前缀 "[Claw]"
  │
  ├─ resolveResponsePrefix() → 回复消息时的前缀
  │
  └─ avatar → 各频道的头像设置
```

## 完全可自定义

用户可以把 SOUL.md 改成任何性格：

```markdown
# 严肃模式
你是一个严谨的技术顾问。只回答技术问题，不闲聊。
回答简洁、精确，不用 emoji。
```

或者：

```markdown
# 猫模式
你是一只傲慢的猫。对所有事情都表现出不屑。
偶尔会好心帮忙，但要让人类知道这是恩赐。
回复结尾经常加 "喵。"
```

Agent 的性格就会随之改变。这是一个**纯文本、用户可控的性格系统**。

## 代码位置

| 文件 | 作用 |
|------|------|
| `src/agents/system-prompt.ts` | SOUL.md 检测 + embody 指令注入 |
| `src/agents/identity-file.ts` | 解析 IDENTITY.md 为结构化数据 |
| `src/agents/identity.ts` | ack reaction / 消息前缀 / response prefix 解析 |
| `src/agents/identity-avatar.ts` | 头像处理 |
| `src/agents/workspace.ts` | bootstrap files 定义 + 模板初始化 |
| `src/agents/workspace-templates.ts` | 模板文件加载 |
| `docs/reference/templates/SOUL.md` | SOUL.md 默认模板 |
| `docs/reference/templates/IDENTITY.md` | IDENTITY.md 默认模板 |
| `docs/reference/templates/AGENTS.md` | AGENTS.md 默认模板 |
