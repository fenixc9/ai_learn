做大模型推理网关时，**不要把后端模型负载均衡当成普通 HTTP 服务的轮询问题**。\
LLM 推理的瓶颈主要在：
- **prefill** 和 **decode** 阶段负载完全不同
- **KV cache 命中率**会直接影响吞吐和时延
- 长短请求混跑会互相干扰
- 真正决定容量的往往不是 QPS，而是 **token rate、并发序列数、KV 内存占用、TTFT/ITL/SLO**

所以网关层更适合做的是：**LLM-aware routing（面向推理特性的路由）**，而不是 round-robin。

vLLM 官方文档强调了其高吞吐来自 **PagedAttention、continuous batching** 等机制；这意味着后端实例并不是“有空闲连接就能接”，而是要看 token 和 KV cache 压力。
NVIDIA Dynamo 的官方文档也明确把 **KV cache aware routing**、**disaggregated prefill/decode**、动态调度列为分布式 LLM serving 的关键能力。citeturn136078search1turn136078search14turn136078search5

## 一、先给结论

网关后端模型负载均衡，建议按这个层次做：

1. **第一阶段：实例分组**
   按模型名、版本、量化方式、上下文长度、GPU 类型分池

2. **第二阶段：池内路由**
   不用轮询，改成 **最小成本路由**
   成本函数至少考虑：
   - 预计 prefill token 数
   - 当前 decode 压力
   - 已占用 KV cache
   - 排队长度
   - 最近 TTFT / ITL
   - prefix/KV cache 命中概率

3. **第三阶段：过载保护**
   做 admission control，超预算直接拒绝、降级或排队

4. **第四阶段：弹性伸缩**
   按 token throughput、KV 占用、P95 TTFT，而不是只看 CPU/QPS

如果你的平台还在第一版，最实用的落地路线是：

> **“模型分池 + 池内 least-loaded + prefix-aware/KV-aware 路由 + token 预算限流”**

这是比“随机/轮询”高一个数量级的正确方向。

---

## 二、为什么普通负载均衡不适合 LLM 网关

普通服务里，一个请求的成本差异通常没那么大。\
但 LLM 请求的成本差异可能非常夸张：

- 输入 100 tokens 和 100k tokens，不是一个量级
- `max_tokens=64` 和 `max_tokens=8192`，decode 压力差很大
- 命中 prefix cache / KV cache 的请求，成本可能显著下降
- 长上下文请求会吃掉大量 KV 内存，直接影响实例还能不能接新活

所以你如果做：

- round-robin
- least-connections
- 随机
- 按请求数均摊

结果通常都会很差。因为它们都没回答最关键的问题：

> 这条请求会给这个实例增加多少“真实推理成本”？

---

## 三、推荐的后端负载均衡架构

### 1. 先按“可运行能力”分池

不要让所有副本混在一个大池子里。先做硬隔离：

- `model = DeepSeek-R1 / Qwen / Llama`
- `revision = 某个具体权重版本`
- `engine = vLLM / SGLang / TensorRT-LLM`
- `precision = fp16 / fp8 / int4`
- `context_window = 32k / 128k / 1M`
- `gpu_class = H100 / H20 / A800 / 4090`
- `feature = function calling / structured output / multimodal`

这样做的目的，是先保证**正确性和性能同质性**。\
同一个路由池里，实例能力和成本模型尽量一致，否则负载分配会失真。

---

### 2. 池内不要按“请求数”，要按“成本”路由

你可以给每个实例维护一个实时状态：

- `running_seqs`
- `waiting_seqs`
- `estimated_decode_tokens_in_flight`
- `used_kv_blocks / kv_cache_usage`
- `gpu_mem_usage`
- `recent_ttft_p50/p95`
- `recent_itl_p50/p95`
- `prefix_cache_hit_ratio`
- `health_score`

然后对每个候选实例算一个分数：

```text
score =
  a * normalized_waiting_queue
+ b * normalized_decode_load
+ c * normalized_kv_usage
+ d * normalized_ttft_p95
- e * prefix_cache_hit_bonus
- f * locality_bonus
+ g * penalty_if_near_oom
```

选分数最低的实例。

这里最关键的不是公式多复杂，而是**别用单一指标**。

### 最少要有的三个核心量

#### ① 预计 prefill 成本

近似用输入 token 数衡量。

```text
prefill_cost ≈ prompt_tokens
```

长 prompt 请求不要都打到同一台。

#### ② 预计 decode 成本

近似用：

```text
decode_cost ≈ expected_output_tokens
```

如果拿不到精确值，可以用：

- `max_tokens`
- 历史平均 completion tokens
- 按业务类型估计（chat / reasoning / code / embedding）

#### ③ KV cache 压力

这是 LLM 路由里最容易被忽略、但最重要的一项。\
KV cache 压力高时，实例虽然“请求数不多”，也可能已经接近不能再接请求。

---

## 四、最值得做的：KV-aware / Prefix-aware 路由

这点对大模型推理平台尤其关键。

如果你的请求存在明显前缀复用，比如：

- 系统提示词固定
- RAG 模板固定
- 同租户模板接近
- 多轮会话持续在同一个 backend 上

那么把相关请求尽量路由到同一个后端，可以提高 prefix/KV cache 命中率，减少重复 prefill。

NVIDIA Dynamo 官方把 **KV Cache Aware Routing** 作为核心能力：它会综合已激活 block 的 decode 成本、待计算 block 的 prefill 成本，以及 KV cache overlap 来选择 worker，以减少重复计算。citeturn136078search1

### 你自己实现时可怎么做

#### 方案 A：会话粘性

最简单。

- 同一个 `session_id / conversation_id` 固定路由到同一后端
- 后端失效时再迁移

适合多轮对话。

#### 方案 B：prefix hash

对标准化后的 prompt 前缀做 hash：

- 去掉用户变量位
- 保留 system + instruction + 模板骨架
- 用 hash 决定优先候选实例

适合模板化请求。

#### 方案 C：显式前缀目录

更高级。

- 网关维护 `prefix_fingerprint -> backend_set`
- 后端定期上报“我缓存了哪些 prefix block / session”
- 新请求优先命中已有 cache 的实例

这个复杂，但收益通常最大。

---

## 五、把 prefill 和 decode 分开看

对于长上下文、推理模型、reasoning workload，**prefill 和 decode 应该被视为两种不同资源消耗**。

- **prefill**：更偏计算密集
- **decode**：更偏内存带宽 / KV cache 压力

vLLM 的 chunked prefill 文档就提到，chunked prefill 的意义之一是把 compute-bound 的 prefill 和 memory-bound 的 decode 更好地混合调度，以提升吞吐和延迟。citeturn389992search8\
Dynamo 官方则进一步把 **disaggregated prefill/decode** 作为关键架构能力。citeturn136078search14turn136078search12

### 网关上的建议

#### 中小规模阶段

先不必真的做物理拆分，但至少要：

- 对长 prompt 请求单独打标签
- 估算 prefill tokens
- 避免把多个超长 prefill 同时塞给一台机器

#### 更大规模阶段

可以做两级架构：

- **prefill pool**
- **decode pool**

典型流程：

1. 请求先进入 prefill 实例
2. 生成初始 KV
3. KV 通过高速通道传给 decode 实例
4. decode 实例负责持续生成

这更复杂，但在长上下文和高吞吐场景下收益明显。官方资料显示，Dynamo 就是围绕这种拆分、动态 GPU 调度和路由优化设计的。citeturn136078search5turn136078search14

---

## 六、不要只看“实例忙不忙”，要看“还能不能收”

建议给每个实例设计一个 **admission budget**。例如：

### 可接单预算

- 最大并发序列数
- 最大 batched tokens
- 最大 KV 占用率
- 最大等待队列
- 最大预计 TTFT

当某台实例超过阈值时：

- 不再选它
- 或只允许短请求进入
- 或只允许命中 cache 的请求进入

### 一个很实用的做法：分级准入

例如：

- `kv_usage < 70%`：可接任何请求
- `70% ~ 85%`：只接短 prompt / 低 max\_tokens 请求
- `85% ~ 95%`：只接 cache-hit 概率高的请求
- `>95%`：停止接新请求

这样能显著减少尾延迟和 OOM。

---

## 七、建议的路由算法演进路线

### 方案 1：least-loaded

适合第一版上线。

指标可用：

- running requests
- waiting queue
- KV usage
- recent TTFT

比 round-robin 强很多。

---

### 方案 2：weighted least-cost

推荐你尽快做到这一版。

对每个请求计算：

```text
request_cost =
  α * prompt_tokens
+ β * expected_output_tokens
+ γ * is_long_context
+ δ * is_reasoning_model
```

然后每个实例维护：

```text
instance_load =
  current_prefill_tokens_in_flight
+ current_decode_tokens_in_flight
+ λ * kv_cache_usage
+ μ * waiting_tokens
```

最终路由：

```text
pick backend with minimal (instance_load + request_cost_adjustment - cache_hit_bonus)
```

这已经很接近实战可用方案了。

---

### 方案 3：shortest-expected-delay

更进一步，用“预计排队完成时间”做目标。

即：

```text
score = estimated_queueing_delay + estimated_service_time - cache_hit_bonus
```

这种思路比“当前负载最低”更准确，因为有些实例虽然当前忙，但很快释放；有些实例虽然请求少，却被长请求卡死。

---

### 方案 4：KV-aware + disaggregated routing

这是更高阶版本，适合大规模集群。\
如果你们是自研平台，建议最终往这个方向靠。

---

## 八、网关必须拿到哪些后端指标

如果后端是 vLLM，官方提供了 `/metrics` 暴露 Prometheus 指标；这非常适合你做网关调度和容量治理。citeturn389992search1turn389992search9

你至少需要采集：

### 请求侧

- request rate
- success / error / timeout / cancel
- prompt tokens
- completion tokens
- total tokens
- streaming duration

### 时延侧

- TTFT p50/p95/p99
- ITL p50/p95/p99
- end-to-end latency p50/p95/p99
- queue wait time

### 资源侧

- running seqs
- waiting seqs
- batch size
- GPU memory usage
- KV cache usage / block usage
- tokens/sec
- prefill tokens/sec
- decode tokens/sec

### 健康侧

- OOM 次数
- CUDA error
- engine restart
- scheduler backlog

如果拿不到这些指标，网关调度就只能盲飞。

---

## 九、和 Kubernetes / 服务网格的关系

KServe 官方文档强调它提供 load balancing、autoscaling、canary 等通用能力。citeturn136078search2turn136078search6\
但对 LLM 来说，**K8s Service / Ingress 只适合做基础流量入口，不适合做最终推理路由决策**。

原因是：

- 它不懂 token
- 不懂 KV cache
- 不懂 prefix 命中
- 不懂 prefill/decode 差异

所以推荐架构是：

- **L4/L7 入口层**：K8s Service / Gateway / Ingress
- **真正的模型路由层**：你自己的 inference gateway
- **执行层**：vLLM / SGLang / TensorRT-LLM / Dynamo 等后端

也就是说，**模型负载均衡要收回到推理网关内部做**。

---

## 十、生产上最容易踩的坑

### 1. 只按请求数均衡

会把长上下文和短请求混在一起，尾延迟暴涨。

### 2. 不做粘性路由

多轮对话每轮都换 backend，KV cache 复用几乎归零。

### 3. 只看 GPU utilization

GPU util 对 decode-heavy workload 常常误导性很强。你更该看 TTFT、ITL、KV usage、tokens/sec。

### 4. 没有 admission control

实例到 95% KV 了还继续塞，最后不是时延爆炸就是 OOM。

### 5. autoscaling 只按 QPS

LLM 容量更应该按 token 速率和尾延迟扩缩容。

### 6. 不区分模型池

不同 GPU、不同量化、不同上下文窗口混在一起，导致路由和容量预测失真。

---

## 十一、一个实用的落地方案

如果你现在就要做，建议直接用下面这套。

### 路由键

先按以下键选池：

```text
(model, revision, engine, context_class, feature_flags, tenant_priority)
```

### 每个实例上报状态

每 1\~2 秒上报：

```text
backend_id
healthy
running_seqs
waiting_seqs
prompt_tokens_in_flight
decode_tokens_in_flight
kv_usage_ratio
gpu_mem_ratio
ttft_p95_30s
itl_p95_30s
cacheable_prefix_fingerprints_topk
```

### 请求进入网关后

提取特征：

```text
prompt_tokens
max_tokens
session_id
prefix_fingerprint
tenant
priority
is_stream
is_reasoning
deadline / timeout budget
```

### 打分

```text
score =
  0.30 * queue_score
+ 0.25 * decode_load_score
+ 0.20 * kv_usage_score
+ 0.15 * ttft_score
+ 0.10 * mem_score
- 0.25 * prefix_hit_bonus
- 0.15 * session_sticky_bonus
```

### 准入规则

- `kv_usage > 0.92` 不接
- `waiting_seqs > threshold` 不接
- `ttft_p95 > SLO * 1.5` 降权
- `health != healthy` 剔除

### 回退策略

- 首选实例失败：重试同池其他实例
- 重试次数严格限制
- 已开始流式输出后不要透明重试
- 超时则按租户策略返回 429 / 503

---

## 十二、如果你问“最推荐哪一种”

对于大模型推理平台网关，我最推荐的是：

> **按模型分池 + KV-aware 的 weighted least-cost 路由 + 会话粘性 + token/KV 准入控制**

这是当前最均衡、最容易落地、效果也最稳定的一条路。\
如果规模继续扩大，再往：

> **disaggregated prefill/decode + 更强的 KV 目录路由**

演进。

---

## 十三、给你的一个简单判断标准

如果你的网关当前还是下面这样：

- nginx / envoy round-robin 到多个 vLLM
- autoscaling 看 CPU / QPS
- 不跟踪 prompt token / completion token / KV usage
- 没有 session stickiness

那基本可以判断：

> 这个网关还没有进入“LLM 原生调度”阶段。

---

如果你愿意，我可以下一步直接给你一版：

**“推理网关后端路由器设计稿”**，包括\
**数据结构、打分公式、缓存目录、健康检查、重试和伪代码**。
