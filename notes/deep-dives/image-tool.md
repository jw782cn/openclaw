# Image Tool 设计解析

## 核心问题

当前对话用的模型可能不支持图片输入，或者用户想用更强的视觉模型分析图片。image tool 让 agent 能**绕过主模型，直接调用一个专门的视觉模型**做图片理解。

## 工作流程

```
用户发了一张图片
  │
  ├─ 主模型支持 vision → 图片自动注入 prompt（attempt.ts 里处理）
  │   image tool 仍然可用，但 description 会提示：
  │   "Only use this tool when images were NOT already provided in the user's message"
  │
  └─ 主模型不支持 vision，或 agent 需要额外分析
     → 调用 image(prompt="这张图里有什么？", image="/path/to/photo.jpg")
       │
       ▼
     Step 1: 加载图片
       支持: 本地路径、file:// URL、http(s) URL、data: URL (base64)
       沙箱内禁止远程 URL
       最多 20 张图同时分析
       │
       ▼
     Step 2: 选择视觉模型（有 fallback 链）
       优先级:
       1. 配置的 agents.defaults.imageModel（显式指定）
       2. 同 provider 的视觉模型（如 provider 有的话）
       3. MiniMax → minimax/MiniMax-VL-01
       4. OpenAI → openai/gpt-5-mini
       5. Anthropic → anthropic/claude-opus-4-6 (fallback: claude-opus-4-5)
       │
       ▼
     Step 3: 直接调用视觉模型 API
       用 Pi SDK 的 complete() 发送 image + prompt
       MiniMax 特殊处理（只支持单张图、用专用 API）
       │
       ▼
     Step 4: 返回文本分析结果
       { content: [{ type: "text", text: "图片中是..." }],
         details: { model: "openai/gpt-5-mini", image: "/path/to/photo.jpg" } }
```

## 模型选择逻辑

`resolveImageModelConfigForTool()` 的策略：

| 条件 | 选择的模型 |
|------|-----------|
| 显式配置了 `imageModel` | 用配置的 |
| 主模型是 MiniMax + 有 auth | minimax/MiniMax-VL-01 |
| 主模型的 provider 有视觉模型 | 同 provider 的视觉模型 |
| 主模型是 OpenAI + 有 auth | openai/gpt-5-mini |
| 主模型是 Anthropic + 有 auth | anthropic/claude-opus-4-6 |
| 都没有 auth | 返回 null（tool 不创建） |

支持 fallback 链：主视觉模型失败时自动 fallback 到下一个。

## Schema

```typescript
Type.Object({
  prompt: Type.Optional(Type.String()),     // 分析指令，默认 "Describe the image."
  image: Type.Optional(Type.String()),      // 单张图路径/URL
  images: Type.Optional(Type.Array(Type.String())), // 多张图（最多 20）
  model: Type.Optional(Type.String()),      // 手动覆盖模型
  maxBytesMb: Type.Optional(Type.Number()), // 图片大小限制
  maxImages: Type.Optional(Type.Number()),  // 最大图片数限制
})
```

## 关键设计

- **不是截图工具**：这个 tool 分析已有的图片，不生成图片
- **独立于主模型**：调用的是单独的视觉模型 API，不走主 agent 的 ReAct loop
- **按需创建**：如果没有任何视觉模型的 auth，tool 返回 null 不注册
- **沙箱安全**：沙箱内禁止远程 URL，路径通过 bridge 解析

## 代码位置

- `src/agents/tools/image-tool.ts` — 主实现
- `src/agents/tools/image-tool.helpers.ts` — 辅助函数
- `src/agents/model-fallback.ts` — 模型 fallback 逻辑
