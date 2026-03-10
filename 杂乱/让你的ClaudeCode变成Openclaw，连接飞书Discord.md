@op7418 (歸藏(guizang.ai)):
📰 让你的 ClaudeCode 变成 Openclaw（龙虾），连接飞书、Discord 远程控制

大家都知道，我这几天都在用 Vibe Coding 做我的 Agent 客户端 Codepilot。

除开春节假期的九天我用 16 天，发布了40 个版本、220 次提交，效率真的拉满。

刚开始这个项目的定位只是 Claude Code 的桌面端，现在已经支持：

- 飞书等 IM 远程连接；

- 可视化配置所有 Code plan 套餐；

- 藏师傅写的设计 Agent 和素材库；

- 多个 Agent 并发分屏；

- Token 使用检测看板；

- 还能一键帮你安装 Claude Code ；

- MacOS 和 Windows 全平台支持。

等一系列的 Agent 功能，可以说完全自己搞了一个 Claude Code 桌面端 + Cowork + OpenClaw。

非常适合小白入门和使用，比 OpenClaw 安全多了：

https://github.com/op7418/CodePilot/releases/tag/v0.26.0

我这几天主要集中在实现一套非常优雅而且全面的 Claude Code 连接各种 IM 工具比如飞书之类的系统。

参考了各个项目的优势同时结合我自己的需求，非常易用，比 OpenClaw 的配置方便非常多。

然后我就想说既然都写了，除了放在 Codepilot 里面不如拆出来开源。

帮大家减轻点开发压力，顺便给那些想使用 Claude Code 本身还想连接 IM 的朋友。

所以就有了两个项目：Claude-to-IM 以及 Claude-to-IM-skill

## Claude-to-IM-skill

这个 Skills 可以直接将你当前的 Claude Code 对话远程连接到飞书、Discord 等 IM 工具，即使你在外面也可以跟你的的 Claude Code 进行交互了。

具体的功能有：

- 三大 IM 平台 — Telegram、Discord、飞书，可任意组合启用

- 交互式配置 — 引导式向导逐步收集 token，附带详细获取说明

- 权限控制 — 工具调用需要在聊天中通过内联按钮明确批准

- 流式预览 — 实时查看 Claude 的输出（Telegram 和 Discord 支持）

- 会话持久化 — 对话在守护进程重启后保留

- 密钥保护 — token 以 chmod 600 存储，日志中自动脱敏

- 无需编写代码 — 安装 Skill 后运行 /claude-to-im setup 即可

其中交互式配置这个非常方便，首次启动的时候 Claude 会通过详细的指引，引导你进行配置。

甚至会告诉你该点哪里，门槛非常低。

设置完就可以跟你的 Claude Code 用 IM 进行远程交互了。

可以用下面这个命令安装：

npx skills add op7418/Claude-to-IM-skill

同时项目也是开源的，可以过来反馈问题和贡献 PR：https://github.com/op7418/Claude-to-IM-skill

## Claude-to-IM

这个主要面向的是开发者朋友们，如果你的产品也是基于 Agent SDK 开发。

你也想要快速的接入多个 IM 远程控制，就可以用这个库。

主要的特点是：

- 多平台适配器：Telegram（长轮询）、Discord（Gateway WebSocket）、飞书（WSClient）

- 流式预览：通过消息编辑实现实时响应草稿，支持按平台定制的节流策略

- 权限管理：通过交互式内联按钮实现 Claude Code 工具审批（允许 / 拒绝 / 本次会话允许）

- 会话绑定：每个 IM 聊天映射到一个持久化的会话，支持工作目录和模型配置

- Markdown 渲染：平台原生格式化——Telegram 用 HTML、Discord 用 Discord 风格 Markdown、飞书用富文本卡片

- 可靠投递：按平台限制自动分块、指数退避重试、HTML 降级、消息去重

- 安全机制：输入验证、令牌桶速率限制（每个聊天 20 条/分钟）、用户授权白名单、完整审计日志

- 宿主无关：所有宿主依赖通过 4 个 DI 接口抽象——不绑定数据库驱动、不绑定 LLM 客户端、不绑定框架

其他详细的接入指南啥的我就不在这里说了，直接去看文档就行：https://github.com/op7418/Claude-to-IM

上面这两个项目都欢迎大家点 Star，同时如果有问题可以在 Github  反馈，也可以提交 PR。

如果觉得今天的内容对你有帮助的话，可以帮我点个赞👍或者转发给需要的朋友们。
📅 Thu Mar 05 14:22:57 +0000 2026
🔗 https://x.com/op7418/status/2029563318973952221
❤️ 859  🔁 161  💬 80