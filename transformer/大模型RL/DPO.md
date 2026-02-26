# 什么是DPO？
直接偏好优化（Direct prefence Optimization）。
* 该算法是基于PPO的RLHF的优化（PPO是近段策略优化， 是OpenAI 2017年提出的： [近端策略优化](https://zhida.zhihu.com/search?content_id=224601754&content_type=Article&match_order=1&q=%E8%BF%91%E7%AB%AF%E7%AD%96%E7%95%A5%E4%BC%98%E5%8C%96&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NzE5NTI1ODksInEiOiLov5Hnq6_nrZbnlaXkvJjljJYiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyMjQ2MDE3NTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.dFrRM1KwrVf7ldK9dt4kD6_KB4av96KfxiiOoCJC_xU&zhida_source=entity)（PPO）算法是[OpenAI](https://zhida.zhihu.com/search?content_id=224601754&content_type=Article&match_order=1&q=OpenAI&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NzE5NTI1ODksInEiOiJPcGVuQUkiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyMjQ2MDE3NTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.LNL_XlqKzvRVcqowLcJtRepsuTxy_GLtd2CkeA4IzYw&zhida_source=entity)在2017提出的一种强化学习算法，被认为是目前强化学习领域的SOTA方法），在PPO基础上进行了大幅简化。

# 对比RLHF和DPO？
### RLHF有什么痛点？
1. **训练不稳定**。为什么不稳定，因为训练的过程中存在4个模型，数十个超参的不同组合也影响着其稳定性。
2. 资源开销极大，因为要同时运行4个模型（策略模型，奖励模型，价值模型，参考模型）。
### DPO怎么解决这些痛点
1. 流程简洁，不需要奖励模型。
2. 稳定性，DPO是一种监督学习算法
3. 资源需求少，只需要策略模型

### 为什么DPO算法不需要奖励模型？
传统RLHF是训练另外一个网络来计算分数，DPO则是：
## 3️⃣ 通俗理解：DPO在干什么？

可以把它理解成一个更直接的训练规则：

传统 RLHF：

> 先学一个老师（奖励模型），再让学生听老师打分。

DPO：

> 不要老师，直接用人类偏好对模型说：  
> “这个回答比那个好，你把前者概率调高就行。”

也就是说：

- 不去学一个“打分函数”
    
- 直接优化“好回答概率 > 坏回答概率”。

DPO 发现奖励模型其实是个"中间商"——你可以从偏好数据直接优化策略模型，数学上完全等价，但省掉了训 RM 和跑 PPO 这两个麻烦步骤。变成了一个简单的分类问题，跟普通 fine-tune 一样用交叉熵就能训。
