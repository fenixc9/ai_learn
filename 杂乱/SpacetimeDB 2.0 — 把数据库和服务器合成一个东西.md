> 来源：[sitin (@sitinme)](https://x.com/sitinme/status/2030210093128798482)
> 时间：Sat Mar 07 09:13
> 👍 639  🔁 120  💬 28

SpacetimeDB 2.0 — 号称比传统数据库快 1000 倍的实时数据库

GitHub Trending + Hacker News 同时爆了。

SpacetimeDB 不是普通数据库，它把数据库和服务器合成了一个东西：
• 客户端直接连数据库，没有中间的 API 服务器
• 数据变化实时推送到所有客户端
• 用 Rust 写的，性能拉满

最骚的演示：他们用 SpacetimeDB 做了视频通话——所有数据通过数据库实时同步传输。

适用场景：

• 🎮 多人游戏（取代传统游戏服务器）
• 💬 实时协作应用
• 📊 实时数据看板
• 🤖 AI Agent 之间的实时通信

Hacker News 上有人质疑 benchmark 的公平性（用了不同的客户端和协议），但架构思路确实新颖——"如果数据库本身就是服务器呢？"

开源地址：https://t.co/DLHzK0SffH 

来源：Hacker News, YouTube

![](https://pbs.twimg.com/media/HCsd3BkaUAAoVbz.png)
![](https://pbs.twimg.com/media/HCsd3BcaUAE33Dv.jpg)