> 来源：[@pritipatelfgoo](https://x.com/pritipatelfgoo) · 2026-03-07
> 👍 1758 转发 381

## 案例：AI Agent 做 Polymarket 量化套利

### 故事背景

一个从量化基金辞职的人，分享了他做 Polymarket 套利的 2 页手写公式，声称年赚 **40 万美元**。

### 复现过程

1. 把 2 页公式图纸丢进 **OpenClaw（Claude）**
2. 提示词：`"给Polymarket建个机器人"`
3. AI 生成了可运行的 Agent，通过 **Telegram** 发送状态通知
4. 结果：连续 4 天，**每天稳定盈利 150 美元**

### 核心思路

- Polymarket 是预测市场（押注某事件是否发生）
- 套利点在于：不同市场对同一事件的定价存在偏差
- 量化公式 → LLM 理解逻辑 → Agent 自动执行

### 启发

- LLM 可以直接"读懂"量化逻辑并生成可运行代码
- 预测市场 + AI Agent 是一个值得深入研究的方向
