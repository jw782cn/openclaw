# 创新点 7：安全与隔离

## 核心思想

Agent 不再只在终端里由本人使用，而是暴露在公开群聊中、面对多个用户。安全模型比终端 coding agent 复杂得多。

## 安全机制

### Sandbox（Docker 沙箱）
- 工具执行可以在 Docker 容器内运行
- 文件系统通过 bridge 隔离
- 浏览器通过 bridge URL 隔离
- 子 agent 继承沙箱（不能逃逸）

### 工具 Policy Pipeline
多层策略过滤，每一层都可以限制可用工具：
1. Profile policy（用户配置的工具白名单/黑名单）
2. Provider policy（特定 LLM provider 的限制）
3. Global policy（全局限制）
4. Agent policy（特定 agent 的限制）
5. Group policy（特定群聊的限制）
6. Sandbox policy（沙箱内的限制）
7. Subagent policy（子 agent 的限制）

### Owner-Only 工具
- 某些敏感工具（如 gateway 管理）只有 owner 可以触发
- `senderIsOwner` 标记用于鉴权

### 群聊安全
- allowFrom 配置控制谁可以触发 agent
- 群聊中的工具集可以比 DM 更受限

## 关键代码位置

- 沙箱: `src/agents/sandbox.ts`, `src/agents/sandbox/`
- 工具策略: `src/agents/pi-tools.policy.ts`, `src/agents/tool-policy-pipeline.ts`
- Owner 策略: `src/agents/tool-policy.ts` → `applyOwnerOnlyToolPolicy()`
- System prompt 安全段: `src/agents/system-prompt.ts` (Safety section)

## 讨论记录

（待补充）
