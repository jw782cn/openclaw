# 创新点 8：插件系统

## 核心思想

通过插件机制让社区可以扩展 OpenClaw 的能力，添加新频道、新工具、新功能，而不需要修改核心代码。

## 插件结构

插件位于 `extensions/` 目录，每个插件是一个独立的 workspace package，有自己的 `package.json`。

### 当前扩展列表（37 个）

频道类:
- bluebubbles, discord, googlechat, feishu, imessage, irc, line
- matrix, mattermost, msteams, nextcloud-talk, nostr
- signal, slack, telegram, tlon, twitch
- whatsapp, zalo, zalouser

功能类:
- copilot-proxy, device-pair, diagnostics-otel
- google-antigravity-auth, google-gemini-cli-auth
- llm-task, lobster, memory-core, memory-lancedb
- minimax-portal-auth, open-prose, phone-control
- qwen-portal-auth, talk-voice, thread-ownership, voice-call

共享:
- shared (公共工具库)

## Plugin SDK

- 导出路径: `openclaw/plugin-sdk`
- 类型定义: `dist/plugin-sdk/index.d.ts`
- 插件的 `dependencies` 不能用 `workspace:*`（npm install 会 break）
- `openclaw` 放在 `devDependencies` 或 `peerDependencies`

## 关键代码位置

- Plugin SDK: `src/plugin-sdk/`
- 插件加载: `src/plugins/`
- 插件工具解析: `src/plugins/tools.ts` → `resolvePluginTools()`
- 扩展目录: `extensions/`

## 设计要点

- 插件安装: `npm install --omit=dev` 在插件目录
- 运行时通过 jiti 别名解析 `openclaw/plugin-sdk`
- 插件工具受 policy pipeline 过滤（pluginToolAllowlist）

## 讨论记录

（待补充）
