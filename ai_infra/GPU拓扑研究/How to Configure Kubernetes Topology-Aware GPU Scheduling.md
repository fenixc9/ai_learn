---
title: "How to Configure Kubernetes Topology-Aware GPU Scheduling"
source: "https://oneuptime.com/blog/post/2026-02-09-topology-aware-gpu-scheduling-nvlink/view"
author:
  - "[[Nawaz Dhandala]]"
published: 2026-02-09
created: 2026-03-15
description: "Configure topology-aware GPU scheduling in Kubernetes to optimize NVLink communication for multi-GPU workloads and maximize training performance."
tags:
  - "clippings"
---
---

在运行多 GPU 工作负载时，并非所有 GPU 配置都相同。通过 NVLink 连接的 GPU 比 PCIe 连接的 GPU 提供更高的带宽。拓扑感知调度可确保每个 pod 都能获得具有最佳连接性的 GPU，从而显著提升分布式训练和推理工作负载的性能。

本指南将向您展示如何配置 Kubernetes 以根据拓扑结构调度 GPU 工作负载。

## 了解 GPU 拓扑结构

GPU系统具有不同的连接拓扑结构：

- **NVLink** ：GPU 间直接连接（300-600 GB/s）
- **NVSwitch** ：全连接 NVLink
- **PCIe** ：CPU 介导的连接（16-32 GB/s）

检查GPU拓扑结构：

```bash
# Run nvidia-smi topo command on GPU node
kubectl debug node/<gpu-node> -it --image=nvidia/cuda:12.0.0-base-ubuntu22.04 -- \
  nsenter -t 1 -m -- nvidia-smi topo -m

# Example output:
#     GPU0  GPU1  GPU2  GPU3  GPU4  GPU5  GPU6  GPU7  CPU Affinity
# GPU0  X    NV12  NV12  NV12  NV12  NV12  NV12  NV12  0-23
# GPU1 NV12   X    NV12  NV12  NV12  NV12  NV12  NV12  0-23
# GPU2 NV12  NV12   X    NV12  NV12  NV12  NV12  NV12  0-23
# ...

# Legend:
# X    = Self
# SYS  = Connection traversing PCIe and NUMA nodes
# NODE = Connection traversing PCIe and single NUMA node
# PHB  = Connection traversing PCIe and PCIe Host Bridge
# PXB  = Connection traversing multiple PCIe bridges
# PIX  = Connection traversing single PCIe bridge
# NV#  = Connection traversing bonded set of # NVLinks
```

## 安装 GPU 功能发现

部署 GPU 特征发现功能，为节点添加拓扑信息标签：

```yaml
# gpu-feature-discovery.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-feature-discovery
  namespace: gpu-operator
spec:
  selector:
    matchLabels:
      app: gpu-feature-discovery
  template:
    metadata:
      labels:
        app: gpu-feature-discovery
    spec:
      nodeSelector:
        nvidia.com/gpu.present: "true"
      containers:
      - name: gpu-feature-discovery
        image: nvcr.io/nvidia/gpu-feature-discovery:v0.8.2
        volumeMounts:
        - name: output-dir
          mountPath: /etc/kubernetes/node-feature-discovery/features.d
        - name: host-sys
          mountPath: /sys
        securityContext:
          privileged: true
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: GFD_FAIL_ON_INIT_ERROR
          value: "false"
      volumes:
      - name: output-dir
        hostPath:
          path: /etc/kubernetes/node-feature-discovery/features.d
      - name: host-sys
        hostPath:
          path: /sys
```

部署并检查标签：

```bash
巴什kubectl apply -f gpu-feature-discovery.yaml

# Check discovered topology labels
kubectl get nodes -l nvidia.com/gpu.present=true -o json | \
  jq '.items[].metadata.labels | with_entries(select(.key | contains("nvidia.com/gpu")))'

# Example labels:
# nvidia.com/gpu.count: "8"
# nvidia.com/gpu.product: "A100-SXM4-40GB"
# nvidia.com/gpu.nvlink: "true"
# nvidia.com/gpu.nvlink.version: "3"
```

基于GPU拓扑结构的节点标签：

```bash
巴什# Label nodes with NVLink topology
kubectl label node gpu-node-1 \
  gpu-topology=nvlink-all-to-all \
  nvlink-bandwidth=600GB

kubectl label node gpu-node-2 \
  gpu-topology=nvlink-paired \
  nvlink-bandwidth=300GB

kubectl label node gpu-node-3 \
  gpu-topology=pcie-only \
  pcie-bandwidth=32GB
```

创建一个脚本，根据拓扑结构自动标注：

```bash
#!/bin/bash
# label-gpu-topology.sh

for node in $(kubectl get nodes -l nvidia.com/gpu.present=true -o name | cut -d/ -f2); do
    echo "Processing node: $node"

    # Get topology from node
    topology=$(kubectl debug node/$node -q -it --image=nvidia/cuda:12.0.0-base-ubuntu22.04 -- \
      nsenter -t 1 -m -- nvidia-smi topo -m 2>/dev/null | grep -c "NV12")

    if [ "$topology" -gt 50 ]; then
        # All GPUs connected via NVLink
        kubectl label node $node gpu-topology=nvlink-mesh --overwrite
        echo "  Labeled as nvlink-mesh"
    elif [ "$topology" -gt 0 ]; then
        # Some NVLink connections
        kubectl label node $node gpu-topology=nvlink-partial --overwrite
        echo "  Labeled as nvlink-partial"
    else
        # PCIe only
        kubectl label node $node gpu-topology=pcie-only --overwrite
        echo "  Labeled as pcie-only"
    fi
done
```

## 配置拓扑结构的 GPU 设备插件

配置 NVIDIA 设备插件以公开拓扑结构：

```yaml
YAML# device-plugin-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: device-plugin-config
  namespace: gpu-operator
data:
  config.yaml: |
    version: v1
    flags:
      migStrategy: none

    sharing:
      timeSlicing:
        renameByDefault: false

    resources:
      gpus:
      - pattern: "*"
        name: nvidia.com/gpu

    # Enable topology-aware scheduling
    topology:
      strategy: nvlink-preferred
      fallback: any
```

更新 GPU Operator 以使用拓扑配置：

```bash
巴什helm upgrade gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --set devicePlugin.config.name=device-plugin-config \
  --reuse-values
```

## 部署具有拓扑亲和性的工作负载

请求具备 NVLink 连接功能的 GPU：

```yaml
YAML# training-job-nvlink.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: distributed-training-nvlink
  namespace: ml-training
spec:
  template:
    spec:
      # Node affinity for NVLink topology
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              # Prefer nodes with NVLink
              - key: gpu-topology
                operator: In
                values:
                - nvlink-mesh
                - nvlink-all-to-all

              # Require specific GPU model
              - key: nvidia.com/gpu.product
                operator: In
                values:
                - "A100-SXM4-40GB"
                - "H100-SXM5-80GB"

      containers:
      - name: trainer
        image: your-registry/pytorch-distributed:v1
        command: ["python", "train.py"]
        resources:
          limits:
            nvidia.com/gpu: 4  # Request 4 GPUs
            memory: "64Gi"
            cpu: "32"
        env:
        - name: NCCL_TOPO_FILE
          value: "/etc/nccl/topo.xml"
        - name: NCCL_DEBUG
          value: "INFO"
        - name: NCCL_IB_DISABLE
          value: "0"
        volumeMounts:
        - name: nccl-topology
          mountPath: /etc/nccl
      volumes:
      - name: nccl-topology
        configMap:
          name: nccl-topology-config
```

## 创建 NCCL 拓扑文件

生成 NCCL 拓扑文件以实现最佳通信：

```xml
XML<!-- nccl-topology.xml -->
<system version="1">
  <gpu dev="0" sm="80" rank="0" gdr="1">
    <cpu affinity="0-23"/>
    <nvlink target="1" count="12"/>
    <nvlink target="2" count="12"/>
    <nvlink target="3" count="12"/>
  </gpu>

  <gpu dev="1" sm="80" rank="1" gdr="1">
    <cpu affinity="0-23"/>
    <nvlink target="0" count="12"/>
    <nvlink target="2" count="12"/>
    <nvlink target="3" count="12"/>
  </gpu>

  <!-- More GPUs... -->
</system>
```

创建 ConfigMap：

```bash
巴什kubectl create configmap nccl-topology-config \
  --from-file=topo.xml=nccl-topology.xml \
  -n ml-training
```

## 实现拓扑感知 Pod 调度器

创建一个考虑GPU拓扑结构的自定义调度器：

```python
Python# topology_scheduler.py
from kubernetes import client, config, watch
import logging

logging.basicConfig(level=logging.INFO)

class TopologyAwareScheduler:
    def __init__(self):
        config.load_incluster_config()
        self.v1 = client.CoreV1Api()
        self.scheduler_name = "topology-aware-scheduler"

    def get_gpu_topology_score(self, node_name):
        """Score nodes based on GPU topology quality"""
        node = self.v1.read_node(node_name)
        labels = node.metadata.labels

        # Score based on topology type
        topology = labels.get("gpu-topology", "unknown")

        scores = {
            "nvlink-mesh": 100,
            "nvlink-all-to-all": 100,
            "nvlink-partial": 70,
            "pcie-only": 30,
            "unknown": 0
        }

        return scores.get(topology, 0)

    def schedule_pod(self, pod):
        """Schedule pod to best node based on topology"""
        namespace = pod.metadata.namespace
        pod_name = pod.metadata.name

        # Get GPU request
        gpu_request = 0
        for container in pod.spec.containers:
            if container.resources and container.resources.limits:
                gpu_request = container.resources.limits.get("nvidia.com/gpu", 0)

        # Only consider topology for multi-GPU pods
        if gpu_request < 2:
            logging.info(f"Single GPU pod, using default scheduling")
            return None

        # Get candidate nodes
        nodes = self.v1.list_node(label_selector="nvidia.com/gpu.present=true")

        # Score nodes
        node_scores = []
        for node in nodes.items:
            # Check if node has enough GPUs
            gpu_capacity = int(node.status.allocatable.get("nvidia.com/gpu", 0))

            if gpu_capacity >= gpu_request:
                score = self.get_gpu_topology_score(node.metadata.name)
                node_scores.append((node.metadata.name, score))

        # Sort by score
        node_scores.sort(key=lambda x: x[1], reverse=True)

        if not node_scores:
            logging.error(f"No suitable nodes found for pod {pod_name}")
            return None

        # Select best node
        best_node = node_scores[0][0]
        logging.info(f"Scheduling pod {pod_name} to node {best_node} (score: {node_scores[0][1]})")

        # Create binding
        binding = client.V1Binding(
            metadata=client.V1ObjectMeta(name=pod_name),
            target=client.V1ObjectReference(kind="Node", name=best_node)
        )

        self.v1.create_namespaced_pod_binding(namespace, binding)

        return best_node

    def run(self):
        """Watch for pods that need scheduling"""
        w = watch.Watch()

        for event in w.stream(
            self.v1.list_pod_for_all_namespaces,
            field_selector=f"spec.schedulerName={self.scheduler_name}"
        ):
            pod = event['object']

            if event['type'] == 'ADDED' and pod.status.phase == 'Pending':
                try:
                    self.schedule_pod(pod)
                except Exception as e:
                    logging.error(f"Failed to schedule pod: {e}")

if __name__ == "__main__":
    scheduler = TopologyAwareScheduler()
    scheduler.run()
```

部署自定义调度程序：

```yaml
YAML# topology-scheduler.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: topology-scheduler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: topology-scheduler
  template:
    metadata:
      labels:
        app: topology-scheduler
    spec:
      serviceAccountName: topology-scheduler
      containers:
      - name: scheduler
        image: your-registry/topology-scheduler:v1
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: topology-scheduler
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: topology-scheduler
rules:
- apiGroups: [""]
  resources: ["nodes", "pods", "pods/binding"]
  verbs: ["get", "list", "watch", "create", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: topology-scheduler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: topology-scheduler
subjects:
- kind: ServiceAccount
  name: topology-scheduler
  namespace: kube-system
```

使用自定义调度程序：

```yaml
YAMLspec:
  schedulerName: topology-aware-scheduler
  containers:
  - name: trainer
    resources:
      limits:
        nvidia.com/gpu: 4
```

## 监控 GPU 通信性能

监控 NVLink 使用情况：

```bash
巴什# Check NVLink stats
kubectl exec <gpu-pod> -- nvidia-smi nvlink -s

# Monitor during training
watch -n 1 'kubectl exec <gpu-pod> -- nvidia-smi nvlink --status'
```

查询 DCGM 指标：

```bash
Promql# NVLink throughput
DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL

# NVLink errors
DCGM_FI_DEV_NVLINK_CRC_FLIT_ERROR_COUNT_TOTAL
```

## 结论

拓扑感知型 GPU 调度可确保多 GPU 工作负载获得最佳的 GPU 部署位置，并利用高带宽的 NVLink 连接。这可以显著提升训练性能，尤其对于 GPU 间通信成为瓶颈的大型模型而言更是如此。通过为节点添加拓扑信息标签，并使用亲和性规则或自定义调度器，您可以确保昂贵的 GPU 集群得到最高效的利用。

### 纳瓦兹·丹达拉

作者

@nawazdhandala • 2026年2月9日 •

Nawaz 怀着对构建可靠系统和提高可观测性的热情，打造了 OneUptime。

[GitHub](https://github.com/nawazdhandala)

### 改进这篇博客文章

我们所有的博客文章都是开源的。如果您发现拼写错误、想要添加更多细节或有更好的解释，欢迎随时贡献内容，让这篇文章对大家都有好处。

[在 GitHub 上编辑此帖子](https://github.com/oneuptime/blog/tree/master/posts/2026-02-09-topology-aware-gpu-scheduling-nvlink) [投稿指南](https://github.com/oneuptime/blog)