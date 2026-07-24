---
title: HAMi vGPU 完整掌握 — Kubernetes GPU 共享与隔离
date: 2026-07-24 15:00:00
tags:
  - GPU虚拟化
categories:
  - 云原生
---

## 前言

[上一篇文章](/2026/07/14/A100通过MIG虚拟化并进行K8S调用/) 我们讲了 A100 的 MIG 硬件切分——性能最强、隔离最硬，但只有 A100/A30/H100 才支持。那普通的 Tesla T4、V100、A10 怎么办？K8s 原生限制一张 GPU 只能分给一个 Pod，跑个小推理就独占 16GB 显存，太浪费了。

**HAMi**（Heterogeneous AI Computing Virtualization Middleware）就是解决这个问题的开源 vGPU 方案。原项目`k8s-vgpu-scheduler`，这次改名 `HAMi` 的同时也将核心的 `vCUDA` 库 `libvgpu.so` 开源了。它不挑 GPU 型号，能在驱动层拦截 CUDA API 调用，把一张物理 GPU 切分成多份虚拟 GPU，每份独立限制显存和算力，给多个 Pod 同时使用。

> 前置阅读：[GPU 使用指南：如何在裸机、Docker、K8s 等环境中使用 GPU](/2026/07/23/GPU-Basic/)，了解基础 GPU 环境搭建。
  
---

## 一、为什么需要 vGPU？

### 1.1 问题根源

在裸机环境下，多个进程可以自然地共享同一张 GPU——驱动负责调度，各进程通过 CUDA Context 分时使用。但到了 K8s 里，情况就变了：

```
1. Device Plugin 上报物理 GPU 数量 → kube-apiserver
2. Pod 申请 nvidia.com/gpu: 1 → kube-scheduler 标记该 GPU 为"已用"
3. 其他 Pod 再申请 → 资源不足，Pending
```

K8s 把 GPU 当成**不可拆分的整块资源**，一张卡分给一个 Pod 就从可分配池中扣掉了，即使这个 Pod 只用了 2GB 显存跑个小推理。这就是 GPU 利用率低的根因——**不是 GPU 不支持共享，是 K8s 的调度模型不让共享**。

### 1.2 三种方案对比

```
隔离强度
  ▲
  │  MIG (硬件级)      ← A100/A30/H100 专属，硬件电气隔离
  │  vGPU/HAMi (驱动级) ← 不限 GPU 型号，CUDA API 拦截
  │  Time-Slicing (调度级) ← 纯时间片轮转，无显存隔离
  └──────────────────────────────► 适用面
```

| 方案 | 显存隔离 | 算力隔离 | 支持 GPU 型号 | 部署难度 |
|---|---|---|---|---|
| **MIG** | ✅ 硬件级 | ✅ 硬件级 | 仅 A100/A30/H100 | 低（原生支持） |
| **HAMi vGPU** | ✅ 驱动拦截 | ✅ 时间片比例 | **所有 NVIDIA GPU** | 中（需安装 HAMi） |
| **Time-Slicing** | ❌ 无隔离 | ❌ 仅轮转 | 所有 NVIDIA GPU | 极低（一个配置参数） |

> **一句话选型**：有 A100 用 MIG，没有就用 HAMi，临时测试用 Time-Slicing。HAMi 是普适性最强的方案。

---

## 二、HAMi 架构

### 2.1 核心组件

```
┌──────────────────────────────────────────────────┐
│                    HAMi 架构                       │
│                                                    │
│  ┌──────────┐  ┌──────────┐  ┌─────────────────┐  │
│  │ Scheduler│  │ Webhook  │  │  Device Plugin   │  │
│  │ (调度器) │  │ (自动注入)│  │  (DaemonSet)     │  │
│  │ 扩展     │  │          │  │  上报 vGPU 资源   │  │
│  └────┬─────┘  └────┬─────┘  └───────┬─────────┘  │
│       │             │                │            │
│       └──────────┬──┘                │            │
│                  ↓                   ↓            │
│  ┌──────────────────────────────────────────────┐ │
│  │             HAMi-Core（libvgpu.so）           │ │
│  │         拦截 CUDA / NVML API 调用             │ │
│  │    显存限制 → cudaMalloc 超限返回 OOM          │ │
│  │    算力限制 → 时间片比例控制 Utilization        │ │
│  └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

| 组件 | 作用 | 部署方式 |
|---|---|---|
| **HAMi-Core** | vCUDA 库，替换容器内原生 `libcuda.so`，拦截显存分配和算力调用 | 以 `.so` 文件挂载进容器 |
| **Device Plugin** | 上报 vGPU 资源（默认 1 张物理 GPU → 10 个 vGPU） | DaemonSet |
| **Scheduler** | 调度器扩展，感知 vGPU 资源，把 Pod 调度到有可用 vGPU 的节点 | Deployment |
| **Webhook** | 自动为 Pod 注入 vGPU 环境变量和 volume 挂载 | MutatingWebhook |

### 2.2 工作原理
```bash
Pod 提交
  -> HAMi Mutating Webhook
  -> HAMi 调度器 filter / score / bind
  -> 设备分配写入 Pod 注解
  -> 设备插件 Allocate()
  -> 容器运行时环境
  -> HAMi 监控和指标
```
![0001.png](/images/hami-vgpu-guide/0001.png)

> **核心机制**：`LD_PRELOAD` 注入 `libvgpu.so`，这个库在真正的 `libcuda.so` 之前拦截所有 CUDA API 调用。显存超了就拒绝，`nvidia-smi` 的输出也帮你修成"看起来只有 3000MB"。

> **注意**： 
> - 安装 HAMi 后，节点上注册的 `nvidia.com/gpu` 值默认为 vGPU 数量。
> - 在 Pod 中申请资源时，`nvidia.com/gpu` 指的是当前 Pod 所需的物理 GPU 数量。

---

## 三、安装部署

### 3.1 前置检查

HAMi 依赖 NVIDIA 驱动栈，如果已经手动配好 NVIDIA 环境，确认以下 3 点：

``` bash
# 1. 驱动版本 ≥ 440
nvidia-smi | head -3

# 2. nvidia-container-toolkit 已配为默认 runtime
docker info | grep -i runtime

# 3. 如果已有原生 nvidia-device-plugin，先卸载（会冲突）
kubectl delete -f nvidia-device-plugin.yml 2>/dev/null
```

### 3.2 安装 HAMi

``` bash
# 给 GPU 节点打标签
kubectl label nodes <gpu-node> gpu=on --overwrite

# 添加 HAMi Helm 仓库
helm repo add hami-charts https://project-hami.github.io/HAMi/
helm repo update

# 安装（替换 <k8s-version> 为你的 K8s 版本，如 v1.27.4）
helm install hami hami-charts/hami \
  --set scheduler.kubeScheduler.imageTag=<k8s-version> \
  -n kube-system
```

查看自己的 K8s 版本：

``` bash
kubectl version --short
# Server Version: v1.27.4
```

### 3.3 验证安装

``` bash
# 看 HAMi 相关 Pod 状态
kubectl -n kube-system get pods | grep hami
# hami-device-plugin-xxxxx   1/1  Running
# hami-scheduler-xxxxx       1/1  Running

# 看节点是否出现 vGPU 资源
kubectl describe node <gpu-node> | grep -A2 "nvidia.com"
# nvidia.com/gpu:     20        ← 2 张物理卡 × 10 = 20 个 vGPU
# nvidia.com/gpumem:  32000     ← 总显存（MB）
```

### 3.4 常用自定义配置

安装时通过 `--set` 可调整参数，完整列表见 [官方配置文档](https://github.com/Project-HAMi/HAMi/blob/master/docs/config_cn.md)：

``` bash
helm install hami hami-charts/hami \
  --set scheduler.kubeScheduler.imageTag=v1.27.4 \
  --set devicePlugin.deviceSplitCount=10 \        # GPU 分割数，默认 10
  --set devicePlugin.deviceMemoryScaling=1 \       # 显存放大比例，>1 启用虚拟显存（实验）
  --set devicePlugin.disablecorelimit=false \      # 是否关闭算力限制
  --set devicePlugin.migStrategy=none \            # MIG 策略：none / mixed
  --set scheduler.defaultMem=5000 \                # 默认显存（MB）
  --set scheduler.defaultCores=0 \                 # 默认算力比例
  -n kube-system
```

| 参数 | 默认值 | 说明 |
|---|---|---|
| `devicePlugin.deviceSplitCount` | 10 | 每张物理 GPU 最多同时跑的任务数 |
| `devicePlugin.deviceMemoryScaling` | 1 | 显存放大比例，>1 启用虚拟显存（实验功能） |
| `devicePlugin.disablecorelimit` | false | true=关闭算力限制 |
| `devicePlugin.migStrategy` | none | MIG 策略，mixed 则专用资源名指定 MIG 设备 |
| `scheduler.defaultMem` | 5000 | 不配显存时的默认值（MB） |
| `scheduler.defaultCores` | 0 | 默认算力预留百分比，0=不限制 |
| `scheduler.defaultGPUNum` | 1 | 默认 vGPU 数量，当 Pod 没设 `nvidia.com/gpu` 但有 `gpumem/gpucores` 时自动补 |
| `resourceName` | `nvidia.com/gpu` | 申请 vGPU 数量的资源名 |
| `resourceMem` | `nvidia.com/gpumem` | 申请 vGPU 显存大小的资源名 |
| `resourceMemPercentage` | `nvidia.com/gpumem-percentage` | 申请 vGPU 显存比例的资源名 |
| `resourceCores` | `nvidia.com/gpucores` | 申请 vGPU 算力比例的资源名 |
| `resourcePriority` | `nvidia.com/priority` | 任务优先级资源名 |

**容器级别也有两个重要环境变量：**

| 环境变量 | 可选值 | 说明 |
|---|---|---|
| `GPU_CORE_UTILIZATION_POLICY` | `default` / `force` / `disable` | 算力限制策略：default 空闲可突破，force 强制限制，disable 关闭 |
| `ACTIVE_OOM_KILLER` | `true` / `false` | 显存超用是否杀容器，true=超限直接 OOM Kill |

---

## 四、使用篇

### 4.1 最基本的 vGPU Pod

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
    - name: ubuntu-container
      image: ubuntu:18.04
      command: ["bash", "-c", "sleep 86400"]
      resources:
        limits:
          nvidia.com/gpu: 1            # 请求 1 个 vGPU
          nvidia.com/gpumem: 3000      # 每个 vGPU 申请 3000MB 显存（可选，整数类型）
          nvidia.com/gpucores: 30      # 每个 vGPU 的算力为 30% 实际显卡的算力（可选，整数类型）
```

``` bash
kubectl apply -f gpu-pod.yaml
kubectl exec gpu-pod -- nvidia-smi
```

输出中会看到 `[HAMI-core Msg]` 初始化日志，且显存显示为 `0MiB / 3000MiB`（而非整张卡的 15360MiB）。退出时 `nvidia-smi` 还会打印 HAMi-core 的清理日志：

```
[HAMI-core Msg(16:139711087368000:libvgpu.c:836)]: Initializing.....
Mon Apr 29 06:22:16 2024
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.54.14              Driver Version: 550.54.14      CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Tesla T4                       On  |   00000000:00:07.0 Off |                    0 |
| N/A   33C    P8             15W /   70W |       0MiB /   3000MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
[HAMI-core Msg(16:139711087368000:multiprocess_memory_limit.c:434)]: Calling exit handler 16
```

> 最后一行 exit handler 日志就是 HAMi 的 vCUDA 库在容器退出时打印的，说明 `libvgpu.so` 确实接管了 CUDA 调用。`nvidia-smi` 显示的 3000MiB 上限也正是 Pod 中 `nvidia.com/gpumem: 3000` 所申请的。

### 4.2 资源字段速查

| 资源名 | 含义 | 示例 |
|---|---|---|
| `nvidia.com/gpu` | vGPU 数量 | `1` |
| `nvidia.com/gpumem` | 显存限制（MB） | `3000` |
| `nvidia.com/gpumem-percentage` | 显存百分比 | `20`（表示 20%） |
| `nvidia.com/gpucores` | 算力百分比（0-100） | `30` |
| `nvidia.com/gpumem-percentage` | 显存百分比（如 50 = 50%） | `50` |
| `nvidia.com/priority` | 优先级：0=高 / 1=低（默认 1） | `0` |

### 4.3 优先级策略

**高优先级（priority=0）** 和 **低优先级（priority=1）** 的行为差异：

| 场景 | 高优先级 Pod | 低优先级 Pod |
|---|---|---|
| GPU 上只有同优先级任务 | `gpucores` 限制**不生效**，可吃满 | `gpucores` 限制**不生效**，可吃满 |
| GPU 上有高优先级任务 | 正常受 `gpucores` 限制 | 严格受 `gpucores` 限制，给高优任务让路 |

> **设计意图**：高优任务独占 GPU 时别限制它，低优任务在 GPU 繁忙时自动被限速。这样既保证吞吐，又不浪费空闲算力。

---

## 五、进阶篇

### 5.1 查看 vGPU 资源分配详情

``` bash
# 列出所有使用 vGPU 的 Pod
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].resources.limits["nvidia.com/gpu"] != null) | {name: .metadata.name, ns: .metadata.namespace, node: .spec.nodeName}'

# 在节点上查看 HAMi 分配记录
kubectl -n kube-system exec -it $(kubectl -n kube-system get pod -l app=hami-device-plugin -o jsonpath='{.items[0].metadata.name}') -- ls /k8s-vgpu/
```

### 5.2 集成 Volcano 调度器

如果集群已有 Volcano，可以不用独立装 HAMi scheduler，直接用 Volcano vGPU 插件：

``` bash
# 1. 安装 Volcano
helm repo add volcano-sh https://volcano-sh.github.io/helm-charts
helm install volcano volcano-sh/volcano -n volcano-system --create-namespace

# 2. 启用 vGPU（在 volcano-scheduler-configmap 中）
kubectl edit cm -n volcano-system volcano-scheduler-configmap
# 确认 deviceshare 插件配置中 VGPUEnable: true

# 3. 部署 Volcano vGPU Device Plugin
kubectl apply -f https://raw.githubusercontent.com/Project-HAMi/volcano-vgpu-device-plugin/main/volcano-vgpu-device-plugin.yml

# 4. Pod 用 Volcano 资源名
# volcano.sh/vgpu-number: 1
# volcano.sh/vgpu-memory: 3000
# volcano.sh/vgpu-cores: 30
```

### 5.3 多个 vGPU Pod 的显存隔离验证

同时跑两个 Pod，分别验证显存限制生效：

``` yaml
# pod-a.yaml — 申请 2000MB
spec:
  containers:
    - name: cuda-a
      image: nvidia/cuda:12.0.1-runtime-ubuntu22.04
      resources:
        limits:
          nvidia.com/gpu: 1
          nvidia.com/gpumem: 2000

# pod-b.yaml — 申请 4000MB
spec:
  containers:
    - name: cuda-b
      image: nvidia/cuda:12.0.1-runtime-ubuntu22.04
      resources:
        limits:
          nvidia.com/gpu: 1
          nvidia.com/gpumem: 4000
```

两个 Pod 部署到同一节点后，分别 `nvidia-smi` 看到不同的显存上限——物理上是同一张卡，HAMi 让每个 Pod 以为自己独占一张"小卡"。

---

## 六、生产排障

### 6.1 vGPU Pod 一直 Pending：`Insufficient nvidia.com/gpu`

``` bash
# 1. 看节点还有多少 vGPU
kubectl describe node <gpu-node> | grep -A3 "nvidia.com/gpu"

# 2. 看哪些 Pod 占着 vGPU
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].resources.limits["nvidia.com/gpu"] != null) | "\(.metadata.namespace)/\(.metadata.name)"'

# 3. 如果 vGPU 没释放（Pod 已删但 vGPU 还标记在用）
kubectl -n kube-system delete pod -l app=hami-device-plugin
# DaemonSet 会自动重建，重置 vGPU 状态
```

### 6.2 `nvidia-smi` 在容器内不显示 HAMi-core 信息

``` bash
# 确认 libvgpu.so 挂载成功
kubectl exec <pod> -- ls /usr/local/vgpu/libvgpu.so

# 确认 LD_PRELOAD 已设
kubectl exec <pod> -- env | grep LD_PRELOAD
# LD_PRELOAD=/usr/local/vgpu/libvgpu.so

# 如果没挂载 → 检查 Webhook 是否正常运行
kubectl -n kube-system get mutatingwebhookconfigurations | grep hami
```

### 6.3 显存限制不生效（Pod 用了超过申请的显存）

- 确认 `nvidia.com/gpumem` 在 `limits` 段而非 `requests` 段（HAMi 读的是 limits）
- 确认 `hami-device-plugin` 启动参数中 `--device-memory-scaling=1`（关闭虚拟显存，默认值）
- 部分旧版 CUDA 程序使用 `cuMemAlloc` 而非 `cudaMalloc`，HAMi 需确认是否拦截了该 API

### 6.4 算力限制感觉没效果

- 算力（`gpucores`）限制是**时间片比例**，不是硬上限。如果 GPU 空闲，即使设了 30% 也能跑到 100%
- 要想**强制上限**，需要设置 `priority=1`（低优先级），且 GPU 上有高优先级 Pod 在跑时才会严格限速

---

## 七、面试高频 5 题

**1. HAMi 和 MIG 的本质区别？**

MIG 是硬件级切分（A100 物理上切成 7 块），静态分区不灵活；HAMi 是软件级通过 `LD_PRELOAD` 注入拦截库，动态限制显存和算力，不限 GPU 型号。MIG 隔离性完胜，HAMi 灵活性完胜。

**2. HAMi 怎么实现显存隔离？**

通过 `libvgpu.so` 拦截 CUDA 的显存分配 API（`cudaMalloc` 等），记录每个进程已分配的显存总量，超过 `nvidia.com/gpumem` 限制时直接返回 `cudaErrorMemoryAllocation`，不给显存。同时 Hook `nvidia-smi` 的输出，让它"看起来"只有申请的大小。

**3. HAMi Device Plugin 为什么把 1 张 GPU 上报成 10 个 vGPU？**

`deviceSplitCount` 默认值 10，用于控制**并发度**——一张物理 GPU 上最多同时跑 10 个 vGPU Pod。这个值和性能无关，只影响调度并发。调大了可以多塞 Pod，但 GPU 竞争更激烈；调小了 Pod 排队等更久。

**4. `nvidia.com/gpucores: 30` 是怎么限制算力的？**

HAMi 在 CUDA kernel launch 时插入时间片控制：根据 `gpucores` 百分比计算每个时间窗口内该进程能占用的 SM（流处理器）时间比例。但它不是硬上限——如果 GPU 空闲，低利用率任务仍然可以跑到 100%。要强制限速，需要配合优先级策略。

**5. HAMi 和原生 Time-Slicing 方案的核心差异？**

Time-Slicing 只分调度时间，**不隔离显存**——Pod A 可能吃满 16GB 导致 Pod B OOM。HAMi 的核心价值就是**显存硬隔离**：每个 Pod 分到独立的显存配额，超了就报错，不会互相影响。这是"能用"和"能用好"的区别。

---

## 总结

```
MIG 硬件切分 ──→ A100/A30/H100 专属（最强隔离）
     ↓
HAMi vGPU ──→ 不限 GPU 型号（显存+算力隔离）
     ↓
Time-Slicing ──→ 临时测试（无隔离，仅并发）
```

HAMi 填补了"没有 A100 但想共享 GPU"的缺口。核心就三个组件：Device Plugin 上报虚拟资源、Scheduler 感知调度、libvgpu.so 拦截隔离。安装 3 条命令，Pod 加 3 个 resource limits，搞定。
