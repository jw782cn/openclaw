# Memory Tool 设计解析

## 核心问题

LLM 本身没有长期记忆 —— 上下文窗口之外的东西就忘了。Memory tool 通过 **RAG（检索增强生成）** 模式，让 agent 能从 Markdown 文件中语义搜索历史信息。

## 两个工具配合使用

### memory_search — 语义搜索

```
memory_search(query="上次部署用的什么配置？", maxResults=5)
→ [
    { path: "MEMORY.md", startLine: 42, endLine: 48, snippet: "...", score: 0.87 },
    { path: "memory/devops.md", startLine: 10, endLine: 15, snippet: "...", score: 0.72 },
  ]
```

输入一段自然语言 query，返回语义最相关的片段列表（带路径、行号、相似度分数）。

### memory_get — 精确读取

```
memory_get(path="MEMORY.md", from=40, lines=20)
→ { path: "MEMORY.md", text: "第40-60行的原文内容..." }
```

根据 search 返回的位置，精确读取那些行的完整内容。

## 强制使用模式（System Prompt 要求）

System prompt 里写死了使用流程：

> "Before answering anything about prior work, decisions, dates, people, preferences, or todos:
> run memory_search on MEMORY.md + memory/*.md;
> then use memory_get to pull only the needed lines.
> If low confidence after search, say you checked."

这是 prompt 层面的"强制"，不是代码层面的。没有任何代码拦截机制确保 agent 真的去搜 — 完全依赖 LLM 的 instruction following。LLM 可能忘了搜、判断错不需要搜、或者"觉得自己知道"就直接编了。

### 为什么不自动注入而是让 agent 主动搜？

因为记忆可能很大（几千行 Markdown）。每次全量注入会浪费 token、占满上下文，大部分内容和当前问题无关。按需检索（RAG 模式）让 agent 只拉相关片段，更高效。

### 代码层面唯一的"硬逻辑"

如果 memory_search 和 memory_get 都不在可用工具列表里，这段 prompt 就不会注入（`system-prompt.ts` 第 49 行检查）。但只要工具可用，执不执行完全看 LLM。

## 存储方式

**就是普通 Markdown 文件**，没有专门的数据库：

```
workspace/
├── MEMORY.md          ← 主记忆文件
└── memory/
    ├── work.md        ← 工作相关记忆
    ├── people.md      ← 人物信息
    └── devops.md      ← 运维记录
```

Agent 自己也可以用 `write`/`edit` 工具修改这些文件 = "写入记忆"。

这个设计很巧妙：
- **用户可以直接编辑**（就是 Markdown，任何编辑器都能改）
- **Git 友好**（纯文本，可以 diff / version control）
- **Agent 可读可写**（不需要专门的 write API）

## 搜索后端

`getMemorySearchManager()` 抽象了搜索实现：

| 后端 | 描述 |
|------|------|
| **qmd** | 内置轻量后端，基于 sqlite-vec 向量存储 |
| **memory-core** | 扩展插件（`extensions/memory-core`） |
| **memory-lancedb** | LanceDB 后端插件（`extensions/memory-lancedb`） |

向量化过程：Markdown 文件被分块（chunk）→ 用 embedding 模型向量化 → 存入向量数据库 → search 时 query 也向量化 → 做余弦相似度匹配。

## Citations（引用）

搜索结果可以带引用（`Source: MEMORY.md#L42-L48`），方便验证。

三种模式：
| 模式 | 行为 |
|------|------|
| `on` | 始终在回复中显示引用 |
| `off` | 不显示引用 |
| `auto`（默认） | DM 里显示，群聊里不显示（避免刷屏） |

引用显示还会根据 session key 自动判断 chat type（direct / group / channel）。

## 结果裁剪

qmd 后端支持 `maxInjectedChars` 限制，防止搜索结果过大撑爆上下文：

```typescript
function clampResultsByInjectedChars(results, budget) {
  let remaining = budget;
  for (const entry of results) {
    if (remaining <= 0) break;
    if (snippet.length <= remaining) {
      clamped.push(entry);
      remaining -= snippet.length;
    } else {
      clamped.push({ ...entry, snippet: snippet.slice(0, remaining) });
      break;
    }
  }
}
```

## 和 Session History 的区别

| | Memory Tool | Session History |
|---|---|---|
| 存什么 | 用户主动整理的结构化知识 | 对话原始记录（JSONL） |
| 格式 | Markdown 文件 | Session JSONL |
| 搜索方式 | 语义搜索（向量） | 按时间顺序读取 |
| 谁写入 | Agent 用 write/edit，或用户手动编辑 | 自动记录对话 |
| 持久性 | 永久（直到删除） | 可能被 compaction 压缩 |
| 用途 | "我记得你喜欢..." | "刚才我们聊了什么" |

## 代码位置

| 文件 | 作用 |
|------|------|
| `src/agents/tools/memory-tool.ts` | memory_search + memory_get 实现 |
| `src/agents/memory-search.ts` | 搜索配置解析 |
| `src/memory/index.ts` | 搜索管理器入口 |
| `src/memory/backend-config.ts` | 后端配置 |
| `src/memory/types.ts` | MemorySearchResult 类型定义 |
| `extensions/memory-core/` | 核心记忆插件 |
| `extensions/memory-lancedb/` | LanceDB 后端插件 |
