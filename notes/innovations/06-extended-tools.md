# 创新点 6：工具生态扩展

## 核心思想

在 Pi SDK 的基础编码工具（read/write/edit/bash）之上，构建了一整套"个人助手"专属工具集。

## 工具清单

### 来自 Pi SDK 的基础工具
| 工具 | 用途 |
|------|------|
| read | 读取文件 |
| write | 创建/覆盖文件 |
| edit | 精确编辑文件 |
| grep | 搜索文件内容 |
| find | 按 glob 找文件 |
| ls | 列目录 |

### OpenClaw 自建的执行工具
| 工具 | 用途 |
|------|------|
| exec | 执行 shell 命令（替代 Pi 的 bash，支持 PTY、后台、沙箱） |
| process | 管理后台 exec 会话 |
| apply_patch | 应用 multi-file patch（仅 OpenAI/Codex） |

### OpenClaw 专属助手工具
| 工具 | 用途 |
|------|------|
| web_search | 网页搜索（Brave API） |
| web_fetch | 抓取 URL 内容 |
| browser | 控制 Playwright 浏览器 |
| canvas | 渲染/执行/截图 Canvas UI |
| nodes | 控制配对的设备 |
| cron | 管理定时任务 |
| message | 跨频道发消息 + 频道动作 |
| tts | 文字转语音 |
| gateway | 管理 OpenClaw 网关进程 |
| agents_list | 列出 agent id |
| sessions_list | 列出 session |
| sessions_history | 获取 session 历史 |
| sessions_send | 跨 session 发消息 |
| sessions_spawn | 生成子 agent |
| subagents | 管理子 agent |
| session_status | 查看 usage/状态 |
| image | 图像分析 |

### 动态工具
- 频道工具（各频道特有操作）
- 插件工具（从已安装插件加载）

## 关键代码位置

- 工具组装: `src/agents/pi-tools.ts` → `createOpenClawCodingTools()`
- OpenClaw 工具创建: `src/agents/openclaw-tools.ts` → `createOpenClawTools()`
- 各工具实现: `src/agents/tools/` 目录

## 工具 Policy Pipeline

所有工具经过多层策略过滤后才暴露给 agent：
profile → provider → global → agent → group → sandbox → subagent

## 讨论记录

（待补充）
