**HPA v1 比较“简单”，基本只适合按 CPU 做扩缩；HPA v2 更完整，支持多指标、更多指标类型，以及扩缩容行为控制。** `autoscaling/v2` 已在 Kubernetes 1.23 成为稳定版；`v2beta1` 已在 1.25 停止提供，`v2beta2` 已在 1.26 停止提供。

具体差异：

1. **指标能力**
    
    - **v1** 主要是 `targetCPUUtilizationPercentage`，也就是按 CPU 利用率扩缩。
        
    - **v2** 支持更丰富的 `metrics` 配置，不仅能看 CPU / 内存这类资源指标，还支持自定义指标和外部指标；而且可以同时配置**多个指标**。Kubernetes 会分别计算每个指标建议的副本数，最终取需要扩得**最多**的那个。
        
2. **API 结构**
    
    - **v1** 的写法比较扁平，例如：  
        `targetCPUUtilizationPercentage: 50`
        
    - **v2** 改成结构化的 `metrics` 数组，每个指标都有 `type`、`target` 等字段。官方迁移指南还特别指出，旧字段 `targetAverageUtilization` 在 v2 中改成了 `target.averageUtilization`，并配合 `target.type: Utilization` 使用。
        
3. **多指标支持**
    
    - **v1** 不支持真正意义上的多指标扩缩。
        
    - **v2** 支持多个指标一起参与决策，这通常是大家升级到 v2 的最主要原因。
        
4. **扩缩容行为控制**
    
    - **v1** 基本没有细粒度行为配置。
        
    - **v2** 可以配置 `behavior`，对 scale up / scale down 的策略、稳定窗口等做更细控制；另外 v2 还能看到更明确的 HPA `status conditions`。
        
5. **兼容性与现状**
    
    - 现在应优先使用 **`autoscaling/v2`**。
        
    - 官方也说明：v2 新引入的字段，在与 `autoscaling/v1` 交互时会以 annotation 的形式保留，但从维护和表达能力上看，生产里新配置基本都建议直接写 v2。
        

一个最直观的对比：

**v1**

apiVersion: autoscaling/v1  
kind: HorizontalPodAutoscaler  
spec:  
  minReplicas: 1  
  maxReplicas: 10  
  scaleTargetRef:  
    apiVersion: apps/v1  
    kind: Deployment  
    name: my-app  
  targetCPUUtilizationPercentage: 50

**v2**

apiVersion: autoscaling/v2  
kind: HorizontalPodAutoscaler  
spec:  
  minReplicas: 1  
  maxReplicas: 10  
  scaleTargetRef:  
    apiVersion: apps/v1  
    kind: Deployment  
    name: my-app  
  metrics:  
    - type: Resource  
      resource:  
        name: cpu  
        target:  
          type: Utilization  
          averageUtilization: 50

实战上可以这样选：

- **只看 CPU，配置非常简单**：v1 能用，但不推荐新项目继续写 v1。
    
- **要看内存、自定义指标、外部指标，或者要多指标组合、控制缩容节奏**：直接用 v2。
    

你要的话，我可以直接给你一份 **“HPA v1 升级到 v2 的对照表”**，把字段怎么改一一列出来。