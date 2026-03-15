# k8s Event到底是什么语义？
**at-least-once** ，所以幂等必须在消费端自己做。

## 先理解 K8s Watch 的投递语义

K8s 的 Watch 机制基于 etcd 的 watch，每个资源变更会产生一个 event（ADDED / MODIFIED / DELETED），每个 event 对应一个单调递增的 `resourceVersion`。但有几种情况会导致你收到重复或乱序的 event：

- **Watch 断开重连**：网络抖动、apiserver 重启、client 超时，都会导致 watch 断开。重连时你需要从某个 `resourceVersion` 继续，但如果这个 version 已经被 etcd compact 掉了，apiserver 会返回 `410 Gone`，你只能重新 LIST 全量资源再重新 Watch——这时候 LIST 拿到的每个资源都会触发一次 ADDED event，和之前已经处理过的完全重复
- **Informer resync**：如果你用的是 client-go 的 Informer（大多数人都是），它有个 `resyncPeriod` 参数，到期后会对本地 cache 里的所有对象触发一轮 UPDATE event，即使这些对象根本没变过
- **多副本消费**：如果你的 controller 跑多个副本做 HA，同一个 event 可能被多个副本同时处理
## 幂等的核心机制：resourceVersion 去重

每个 K8s 对象都有 `metadata.resourceVersion`，这是你做幂等的锚点。

**方案一：基于 resourceVersion 的状态机**

最常用的模式。核心思想是：不要把 event 当"命令"处理，而是把它当"状态通知"——你不关心收到了几次 event，只关心对象当前的期望状态和实际状态是否一致。

go

```go
func (c *Controller) processItem(key string) error {
    obj, exists, err := c.informer.GetStore().Get(key)
    if err != nil {
        return err
    }
    
    if !exists {
        // 对象已删除，做清理
        return c.handleDelete(key)
    }
    
    // 不看 event 类型，只看当前状态
    // 这天然就是幂等的：不管收到几次 event，
    // 最终都是把实际状态 reconcile 到期望状态
    current := obj.(*v1.Pod)
    return c.reconcile(current)
}

func (c *Controller) reconcile(pod *v1.Pod) error {
    // 检查当前状态 vs 期望状态
    desired := c.computeDesiredState(pod)
    actual, err := c.getActualState(pod)
    if err != nil {
        return err
    }
    
    if reflect.DeepEqual(desired, actual) {
        return nil  // 已经是期望状态，跳过（幂等）
    }
    
    // 执行变更
    return c.applyState(pod, desired)
}
```

这就是 K8s 社区推崇的 **reconciliation loop** 模式。Operator SDK、Kubebuilder 生成的 controller 骨架都是这个范式——你实现的 `Reconcile(req)` 方法只接收一个 object key，不接收 event 类型，天然引导你写出幂等逻辑。

**方案二：显式 resourceVersion 去重**

如果你的场景不适合 reconcile 模式（比如你需要对每次变更触发一个外部动作，而不是做状态同步），就需要显式跟踪已处理的 resourceVersion：

go

```go
type EventProcessor struct {
    // 记录每个对象最后处理的 resourceVersion
    processedVersions sync.Map  // key: namespace/name, value: resourceVersion
}

func (p *EventProcessor) Handle(eventType watch.EventType, obj *unstructured.Unstructured) error {
    key := obj.GetNamespace() + "/" + obj.GetName()
    rv := obj.GetResourceVersion()
    
    // 检查是否已处理过更新的版本
    if lastRV, ok := p.processedVersions.Load(key); ok {
        if compareResourceVersions(lastRV.(string), rv) >= 0 {
            // 已经处理过同版本或更新的版本，跳过
            return nil
        }
    }
    
    // 处理 event
    if err := p.doWork(eventType, obj); err != nil {
        return err  // 不更新 version，下次重试还会处理
    }
    
    // 标记已处理
    p.processedVersions.Store(key, rv)
    return nil
}

// resourceVersion 是数字字符串，直接比较大小
func compareResourceVersions(a, b string) int {
    ai, _ := strconv.ParseInt(a, 10, 64)
    bi, _ := strconv.ParseInt(b, 10, 64)
    if ai < bi { return -1 }
    if ai > bi { return 1 }
    return 0
}
```

但这里有个坑：`resourceVersion` 在 K8s 文档里被明确标注为 **opaque value**，官方不保证它是数字（虽然在 etcd 后端下它确实是 etcd 的 modRevision，是单调递增整数）。如果你的集群用了其他存储后端，数字比较可能不成立。更安全的做法是用 `metadata.generation`（只在 spec 变更时递增）或者自己在 annotation 里维护版本号。

**方案三：外部去重表（持久化）**

如果你的 controller 可能重启、丢失内存状态，需要把去重信息持久化：

go

```go
// 用对象本身的 annotation 或 status 记录处理状态
func (c *Controller) reconcile(pod *v1.Pod) error {
    lastProcessedRV := pod.Annotations["my-controller/last-processed-rv"]
    currentRV := pod.ResourceVersion
    
    if lastProcessedRV == currentRV {
        return nil  // 已处理过，幂等跳过
    }
    
    // 做业务逻辑
    if err := c.doWork(pod); err != nil {
        return err
    }
    
    // 回写 annotation 标记已处理
    // 注意：这本身又会触发一次 MODIFIED event，
    // 但下次进来时 lastProcessedRV == currentRV 会直接跳过
    patch := fmt.Sprintf(
        `{"metadata":{"annotations":{"my-controller/last-processed-rv":"%s"}}}`,
        currentRV,
    )
    _, err := c.clientset.CoreV1().Pods(pod.Namespace).Patch(
        ctx, pod.Name, types.MergePatchType, []byte(patch), metav1.PatchOptions{},
    )
    return err
}
```

这个方案的代价是每次处理都会多一次 API 调用（patch annotation），而且 patch 本身触发的 MODIFIED event 需要额外处理（上面代码已经通过 RV 比较覆盖了）。

## 对外部系统操作的幂等

如果你的 controller 需要调用外部系统（创建云资源、发通知、写数据库），光在 K8s 侧去重还不够——你的外部操作本身也需要幂等：

go

```go
func (c *Controller) reconcile(obj *MyResource) error {
    // 用 K8s 对象的 UID 作为外部操作的幂等 key
    idempotencyKey := string(obj.UID) + "-" + obj.ResourceVersion
    
    // 先检查外部系统是否已经处理过
    exists, err := c.externalClient.CheckExists(idempotencyKey)
    if err != nil {
        return err
    }
    if exists {
        return nil  // 外部系统已处理，跳过
    }
    
    // 执行外部操作，带幂等 key
    return c.externalClient.Create(idempotencyKey, obj.Spec)
}
```

`UID + resourceVersion` 的组合可以唯一标识一个对象的一个特定版本，适合作为幂等 key。单用 UID 不够（对象被多次更新，每次都需要触发外部操作），单用 resourceVersion 也不够（不同对象可能有相同的 RV 数字）。

## Informer 层面减少重复

在消费端做幂等之外，也可以在 Informer 层面减少不必要的触发：

go

```go
informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
    UpdateFunc: func(oldObj, newObj interface{}) {
        oldPod := oldObj.(*v1.Pod)
        newPod := newObj.(*v1.Pod)
        
        // 过滤掉 resync 触发的假 update（RV 没变）
        if oldPod.ResourceVersion == newPod.ResourceVersion {
            return
        }
        
        // 只关心 spec 变更，忽略 status 变更
        if reflect.DeepEqual(oldPod.Spec, newPod.Spec) {
            return
        }
        
        c.enqueue(newPod)
    },
})
```

或者更简洁地，设 `resyncPeriod` 为 0 关掉定期 resync（如果你不需要它作为兜底机制的话）：

go

```go
factory := informers.NewSharedInformerFactory(clientset, 0)  // 0 = 不 resync
```

## 总结一下

最推荐的是 reconcile 模式——不关心 event 类型和次数，只做状态收敛。这不是偷懒，而是 K8s 的 watch 语义决定的最优解。只有当你的场景必须对"每次变更"做 exactly-once 的外部动作时，才需要上显式的 resourceVersion 去重加外部幂等 key。