# 创新点 5：Gateway 即服务

## 核心思想

OpenClaw 不是一个"跑完就退出"的 CLI 脚本，而是一个本地常驻的服务（daemon），可以开机自启、崩溃重启，长期运行。

## 运行形态

- **macOS**: menubar app（SwiftUI），通过 launchd 管理生命周期
- **Linux**: systemd user service
- **开发模式**: 直接 `openclaw gateway run`

## 客户端应用

- `apps/macos/` — SwiftUI macOS 菜单栏应用（也是 Gateway 的主要宿主）
- `apps/ios/` — SwiftUI 移动端
- `apps/android/` — Kotlin 移动端
- `apps/shared/` — 共享的 OpenClawKit

## 关键代码位置

- Gateway 核心: `src/gateway/`
- Daemon 管理: `src/daemon/`
- CLI 入口: `src/commands/`
- macOS 打包: `scripts/package-mac-app.sh`

## 设计要点

- Gateway 是控制面，负责消息路由、频道管理、AI 模型调度
- 通过 RPC 与 CLI 和客户端通信
- `openclaw onboard --install-daemon` 一键安装为系统服务
- `openclaw gateway status/start/stop/restart` 管理生命周期

## 讨论记录

（待补充）
