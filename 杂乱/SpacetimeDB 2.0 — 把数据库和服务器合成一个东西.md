> 来源：[@sitinme](https://x.com/sitinme) · 2026-03-07

## SpacetimeDB 2.0

号称比传统数据库快 **1000 倍**的实时数据库，GitHub Trending + Hacker News 同时爆。

### 核心设计

- **客户端直连数据库**，没有中间的 API 服务器层
- 数据变化**实时推送**到所有客户端（类似 websocket，但不用自己写）
- 用 **Rust** 写，性能极强
- 把数据库逻辑（SQL/索引）和服务器逻辑（业务代码）合并成一个运行时

### 最骚的演示

用 SpacetimeDB 做了一个大规模多人游戏后端，数千玩家实时同步，延迟极低。

### 适合场景

- 实时多人游戏
- 协作应用（类 Notion/Figma 实时同步）
- 任何需要"数据变化立刻推给所有客户端"的场景

### 链接

- GitHub: https://github.com/clockworklabs/SpacetimeDB
