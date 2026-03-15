核心思想很直接：既然 K8s 默认的 device plugin 不懂拓扑，那就自己写一个，在 `Allocate()` 的时候按照预先构建的 NVLink 亲和矩阵来分配 GPU，而不是随机分。

整个方案分三层：

**第一层：拓扑发现（离线或启动时）**

节点上的 daemon 在启动时通过 NVML API 构建 GPU 亲和矩阵。`nvidia-smi topo -m` 做的事情：
go

```go
// 用 NVML 获取两张卡之间的互联类型
nvmlDeviceGetTopologyCommonAncestor(deviceA, deviceB, &pathInfo)
```

NVML 返回的 `pathInfo` 就是你之前看到的那些枚举值——`NVML_TOPOLOGY_INTERNAL`（同一张卡）、`NVML_TOPOLOGY_SINGLE`（同一 PCIe switch）、`NVML_TOPOLOGY_MULTIPLE`（跨 PCIe switch 同 NUMA）、`NVML_TOPOLOGY_HOSTBRIDGE`、`NVML_TOPOLOGY_NODE`（跨 NUMA）、`NVML_TOPOLOGY_SYSTEM`。

拿到矩阵后，预处理成分组。对于典型的 8 卡机器，分组策略一般是这样：

- 请求 8 卡：唯一解，整机分配
- 请求 4 卡：`[0,1,2,3]` 或 `[4,5,6,7]`（同一个 NVSwitch domain / NVLink 全互联子图）
- 请求 2 卡：优先同 NVSwitch domain 内的对，再优先 NV2 > NV1 > PIX > NODE > SYS
- 请求 1 卡：无所谓拓扑，但优先 NUMA 亲和

**第二层：自定义 Device Plugin**

K8s device plugin 的接口很简单，核心就三个 gRPC 方法：`ListAndWatch`（报告有哪些设备）、`Allocate`（分配设备）、`GetPreferredAllocation`（推荐分配方案）。关键改造在后两个。

`GetPreferredAllocation` 是 K8s 1.19+ 加的，专门给 device plugin 一个机会说"如果你要从这些可用卡里选 N 张，我推荐这几张"。这就是注入拓扑感知逻辑的地方：
```go
func (p *NvlinkPlugin) GetPreferredAllocation(
    req *pluginapi.PreferredAllocationRequest,
) (*pluginapi.PreferredAllocationResponse, error) {
    resp := &pluginapi.PreferredAllocationResponse{}
    
    for _, creq := range req.ContainerRequests {
        available := creq.AvailableDeviceIDs  // 当前空闲的 GPU
        needed := int(creq.AllocationSize)     // 需要几张
        
        // 核心：从可用卡里选出拓扑最优的 N 张
        best := p.findBestTopologyGroup(available, needed)
        
        resp.ContainerResponses = append(resp.ContainerResponses,
            &pluginapi.ContainerPreferredAllocationResponse{
                DeviceIDs: best,
            })
    }
    return resp, nil
}
```


`findBestTopologyGroup` 的实现思路：

go

```go
func (p *NvlinkPlugin) findBestTopologyGroup(
    available []string, needed int,
) []string {
    // 1. 先查预计算的分组表
    //    比如 needed=4，直接看 [0,1,2,3] 或 [4,5,6,7] 哪组可用卡够
    for _, group := range p.precomputedGroups[needed] {
        if isSubset(group, available) {
            return group
        }
    }
    
    // 2. 预定义分组都不满足（有卡被占了），回退到贪心
    //    从可用卡里选出 "组内最差链路最好" 的组合
    return p.greedyTopologySelect(available, needed)
}
```

贪心选择的逻辑：

```go
func (p *NvlinkPlugin) greedyTopologySelect(
    available []string, needed int,
) []string {
    // 枚举所有 C(len(available), needed) 组合
    // 对每个组合，计算 "组内最差互联类型" 作为 score
    // 返回 score 最高的组合
    
    bestScore := -1
    var bestCombo []string
    
    forEachCombination(available, needed, func(combo []string) {
        score := math.MaxInt
        for i := 0; i < len(combo); i++ {
            for j := i + 1; j < len(combo); j++ {
                link := p.topoMatrix[combo[i]][combo[j]]
                if link < score {
                    score = link  // 瓶颈 = 最差的那条链路
                }
            }
        }
        if score > bestScore {
            bestScore = score
            bestCombo = combo
        }
    })
    
    return bestCombo
}
```

**互联类型的打分**大致是：NVSwitch full mesh (100) > NV2 (80) > NV1 (60) > PIX (40) > PHB (30) > NODE (20) > SYS (10)。

**第三层：Scheduler Extender / Scheduling Framework Plugin**

Device plugin 解决的是"单节点内怎么选卡"，但还有一个问题：**K8s 调度器先选节点，再让 device plugin 分配卡**。如果调度器把 pod 调到一台已经占了 `[0,1,4,5]` 的机器上，你请求 4 卡，剩下的 `[2,3,6,7]` 跨了两个 NVSwitch domain——device plugin 只能在烂牌里选最不烂的。

所以需要在调度层面也注入拓扑感知。做法是写一个 Scheduling Framework Plugin：

```go
// Filter 阶段：排除无法提供连续拓扑组的节点
func (pl *TopoPlugin) Filter(
    ctx context.Context, 
    state *framework.CycleState,
    pod *v1.Pod, 
    nodeInfo *framework.NodeInfo,
) *framework.Status {
    needed := getGPURequest(pod)  // 从 pod spec 读取 GPU 数量
    available := getAvailableGPUs(nodeInfo)
    
    // 检查这个节点上，能否从可用卡中凑出一个拓扑合格的组
    if !canFormTopologyGroup(available, needed, MinLinkType_NV) {
        return framework.NewStatus(framework.Unschedulable,
            "no topology-qualified GPU group available")
    }
    return framework.NewStatus(framework.Success)
}

// Score 阶段：有多个候选节点时，拓扑质量更好的得分更高
func (pl *TopoPlugin) Score(
    ctx context.Context,
    state *framework.CycleState,
    pod *v1.Pod,
    nodeName string,
) (int64, *framework.Status) {
    needed := getGPURequest(pod)
    available := getAvailableGPUs(nodeInfoMap[nodeName])
    
    bestGroup := findBestTopologyGroup(available, needed)
    score := evaluateGroupQuality(bestGroup)  // 同样的打分逻辑
    
    return score, framework.NewStatus(framework.Success)
}
```

这样调度器就会**优先把 pod 放到能提供完整 NVLink domain 的节点上**，而不是傻傻地只看"哪台机器空闲 GPU 数量够"。

## 实际部署的注意点

**NUMA pinning 配合**：光选对 GPU 还不够，还需要把 pod 的 CPU 和内存 pin 到对应的 NUMA node 上。kubelet 的 Topology Manager 设成 `single-numa-node` 可以帮你做这件事，但前提是你的 device plugin 正确实现了 `GetPreferredAllocation`，让 Topology Manager 知道你倾向哪些卡，它才能协调 CPU 分配。

**碎片化问题**：时间长了，节点上的 GPU 会碎片化——比如 8 卡机被分成了 2+2+1+1+2（空闲），你来了一个 4 卡请求，怎么办？这时候要么接受次优分配，要么做 pod preemption/migration。一些大厂会跑定期的碎片整理（drain + reschedule），但这在生产环境里很重（需要 checkpoint + 恢复）。

**和 NCCL 的配合**：分配完 GPU 后，NCCL 运行时也需要知道拓扑才能选最优的通信算法。设置 `NCCL_TOPO_FILE` 环境变量指向一个拓扑 XML 文件，或者让 NCCL 自己通过 NVML 发现（默认行为）。但如果你的容器里 NVML 看到的 device index 和物理拓扑不一致（比如用了 `CUDA_VISIBLE_DEVICES` 重映射），NCCL 可能选错算法，所以要确保 `CUDA_VISIBLE_DEVICES` 的顺序和物理拓扑一致。

总结一下，这个方案的核心就是三件事：用 NVML 建矩阵，在 device plugin 里按矩阵分卡，在 scheduler plugin 里按矩阵选节点。逻辑不复杂，但工程细节（碎片整理、NUMA 协调、NCCL 配合）是真正花时间的地方。


