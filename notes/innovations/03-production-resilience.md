# 创新点 3：生产级容错（外层循环）

## 核心思想

在 Pi SDK 的 `session.prompt()` 外面包了一层 `while (true)` 循环，处理各种生产环境中的故障，让 agent 能 7x24 稳定运行。

## 容错机制

### Auth Profile 轮转
- 一个 provider 可配多个认证 profile（OAuth、API key 等）
- 当前 profile 失败 → 自动切换到下一个
- 失败的 profile 进入冷却期（cooldown），一段时间后再尝试

### 模型 Failover
- 主模型不可用（超时、限流、服务端错误）→ 自动 fallback 到备选模型
- 支持配置 fallback 链

### Context Overflow → 自动 Compaction
- 上下文超出模型窗口 → 自动压缩旧消息为摘要 → 重试
- 最多尝试 3 次 compaction
- 也支持 tool result 截断（过大的工具输出被裁剪）

### Thinking Level 降级
- high thinking 失败 → 自动降级到 medium → low
- 某些 provider 不支持 extended thinking 时自动适配

## 核心代码

- 外层循环: `src/agents/pi-embedded-runner/run.ts` (第 480 行 `while (true)`)
- 单次尝试: `src/agents/pi-embedded-runner/run/attempt.ts`
- Compaction: `src/agents/pi-embedded-runner/compact.ts`
- Auth profile: `src/agents/auth-profiles.ts`
- Failover: `src/agents/failover-error.ts`, `src/agents/model-fallback.ts`
- Tool result 截断: `src/agents/pi-embedded-runner/tool-result-truncation.ts`

## 讨论记录

（待补充）
