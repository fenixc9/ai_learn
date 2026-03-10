> 来源：[@fkysly](https://x.com/fkysly) 马天翼 · 2026-03-08
> 👍 65 转发 10

## rtk（Rust Token Killer）

给 Claude Code 省 token 的 CLI 代理工具，通过过滤命令输出中的冗余信息来压缩输入 token。

### 实测压缩率（在中型 Next.js 项目）

| 命令 | 原始大小 | 压缩后 | 压缩率 |
|------|---------|--------|--------|
| grep | 527 KB | 102 字节 | **99%** |
| ls | 较好 | — | 去掉权限、日期等 |
| git status | 较好 | — | 去掉提示语等冗余 |

### 工作原理

`git status` → 自动重写为 `rtk git status`（透明代理，0 token overhead）

### 局限

- 只能节省**输入 token**，不影响输出 token
- 有信息丢失的问题（过滤了部分细节）
- 适合重度命令行用户

### 链接

- GitHub: https://github.com/reachingforthejack/rtk（注意：同名包，需确认是 Rust Token Killer 而非 Rust Type Kit）
