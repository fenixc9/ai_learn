> 来源：[马天翼 (@fkysly)](https://x.com/fkysly/status/2030518029424136367)
> 时间：Sun Mar 08 05:36
> 👍 65  🔁 10  💬 9

实测 rtk (Rust Token Killer)，这是一个给 Claude Code 省 token 的 CLI 工具。我让 Claude Code 在一个中型 Next.js 项目上跑了对比：
* grep 的压缩率99%，直接从 527KB 干到 102 字节。
* ls 和 git status 压缩也很不错，去掉了权限、日期、提示语等信息。

https://t.co/wEryby5U4l

不过只能节省输入 token，而且有信息丢失的问题，如果重度命令用户，可以试试。

![](https://pbs.twimg.com/media/HC3Y8-EaYAAdAXa.png)