---
title: KubeVirt 完整掌握 — 从零到 50 VM 压力测试
date: 2026-07-29 10:00:00
tags:
  - KubeVirt
  - 虚拟化
  - Kubernetes
  - 压力测试
categories:
  - 云原生
---

## 前言

KubeVirt 把虚拟机（VM）变成 K8s 的一等公民——你可以用 `kubectl` 管理 VM，就像管理 Pod 一样。VM 和容器跑在同一个集群里，共享网络、存储和调度能力。

本文在 3 节点混合集群上从零部署 KubeVirt，经历 HTTP 导入镜像 → 单 VM 验证 → 50 并发压力测试的完整流程，记录所有踩坑经验。

> 前置条件：一个 K8s 集群（本文基于 v1.35.4，3 节点），已配置 StorageClass 和私有 Registry。

---

## 1. 环境概览

| 节点 | CPU | 内存 | 磁盘 | 角色 | OS |
|---|---|---|---|---|---|
| 192.168.55.41 | 32C | 64GB | 98GB | control-plane | Ubuntu 22.04 |
| 192.168.55.2 | 64C | 256GB | 295GB | worker | Ubuntu 24.04 |
| 192.168.88.128 | 16C | 32GB | 98GB | worker | Ubuntu 22.04 |

- **StorageClass**：`local-path`（rancher.io/local-path），PV 路径 `/opt/local-path-provisioner/`
- **私有 Registry**：`easzlab.io.local:5000`（HTTP，宿主机 192.168.55.41）

### DNS 前置配置

集群部署了 `node-local-dns` DaemonSet（监听 `169.254.20.10`），会拦截所有 DNS 请求。私有 Registry 域名需要显式绕过 catch-all，转发到 CoreDNS：

``` yaml
# CoreDNS ConfigMap — hosts 插件注册域名
192.168.55.41 easzlab.io.local

# node-local-dns ConfigMap — 区域转发
easzlab.io.local:53 {
    forward . 10.68.0.2
}
```

---

## 2. KubeVirt + CDI 安装

### 2.1 CDI（Containerized Data Importer）

CDI 负责将外部磁盘镜像导入到 PVC 中，是整个 VM 创建的基石。

``` bash
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/download/v1.65.0/cdi-operator.yaml
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/download/v1.65.0/cdi-cr.yaml
```

CDI CR 关键配置：

``` yaml
spec:
  config:
    featureGates:
    - HonorWaitForFirstConsumer   # CDI 等 Pod 消费 PVC 后才启动导入/克隆
    - WebhookPvcRendering
    insecureRegistries:
    - easzlab.io.local:5000        # 私有 Registry 是 HTTP，必须加
```

| 配置 | 作用 | 不配的后果 |
|---|---|---|
| `HonorWaitForFirstConsumer` | CDI 不立即导入，等 Pod 绑定 PVC 再触发 | WFFC 存储类下 PV 可能创建在错误节点 |
| `insecureRegistries` | 允许 HTTP Registry | `http: server gave HTTP response to HTTPS client` |

### 2.2 KubeVirt

``` bash
helm repo add kubevirt https://kubevirt.github.io/kubevirt-charts
helm repo update
helm install kubevirt kubevirt/kubevirt \
  --namespace kubevirt --create-namespace
```

验证：

``` bash
kubectl -n kubevirt get pods
# virt-api-xxxxx          1/1  Running   # API 入口
# virt-controller-xxxxx   1/1  Running   # VM 状态管理
# virt-handler-xxxxx      1/1  Running   # 每节点一个，操作 VM
# virt-operator-xxxxx     1/1  Running   # 生命周期管理
```

> 四个组件缺一不可：`virt-api` 是 REST 网关，`virt-controller` 管状态，`virt-handler` 是节点 agent，`virt-operator` 管部署。

---

## 3. 基础镜像构建

### 3.1 方案选择

| 方案 | 原理 | 适用 |
|---|---|---|
| **HTTP 导入** | CDI 直接下载 qcow2 写入 PVC | 简单高效，推荐 |
| Registry 导入 | 打包成容器镜像 → CDI 拉取提取 | 多一步，适合已有镜像仓库的场景 |

采用 HTTP 导入 + xz 压缩。

### 3.2 搭建 HTTP 源

在控制平面节点上启动 nginx 提供镜像下载：

``` bash
apt install -y nginx

# /etc/nginx/sites-available/imageserver:
#   server_name imageserver;
#   root /root/kubevirt-cdi;
#   autoindex on;

ln -sf /etc/nginx/sites-available/imageserver /etc/nginx/sites-enabled/
systemctl restart nginx

curl -I http://imageserver/openEuler-22.03-LTS-SP4-x86_64.qcow2.xz
# Content-Length: 477MB （原始 qcow2 1.5GB，xz 压缩后 477MB）
```

> CDI 支持自动检测压缩格式并**流式解压**：下载 477MB → 解压为 1.5GB → 写入 PVC sparse 格式（实际 ~2.5GB）。

### 3.3 创建基础 DataVolume

``` yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: openeuler-disk
  namespace: default
spec:
  source:
    http:
      url: "http://imageserver/openEuler-22.03-LTS-SP4-x86_64.qcow2.xz"
  storage:
    accessModes:
      - ReadWriteOnce
    storageClassName: "local-path"
    resources:
      requests:
        storage: 40Gi
```

``` bash
kubectl apply -f datavolume-openeuler.yaml

# 观察导入进度
kubectl get dv openeuler-disk -w
# ImportScheduled → ImportInProgress → Succeeded

kubectl get pvc openeuler-disk
# NAME             STATUS   VOLUME        CAPACITY
# openeuler-disk   Bound    pvc-4870e...  43418Mi
```

这个 PVC 作为后续所有 VM 的**克隆源**——基础镜像只下载一次，后续 VM 通过 PVC 克隆秒级获得系统盘。

---

## 4. Cloud-Init 配置

``` yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudinit
type: Opaque
stringData:
  userdata: |
    #cloud-config
    ssh_authorized_keys:
      - ssh-rsa AAAAB3Nza... root@ops
    runcmd:
      - [sh, -c, "mkdir -p /ddhome"]
      - [sh, -c, "timeout 25 sh -c 'until [ -b /dev/vdb ]; do udevadm settle --timeout=3; sleep 0.5; done'"]
      - [sh, -c, "blkid /dev/vdb || mkfs.ext4 /dev/vdb"]
      - [sh, -c, "UUID=$(blkid -s UUID -o value /dev/vdb); sed -i '/\\/ddhome/d' /etc/fstab; echo \"UUID=$UUID /ddhome ext4 defaults,nofail 0 0\" >> /etc/fstab"]
      - [sh, -c, "mount -a"]
```

| 设计点 | 说明 |
|---|---|
| **UUID 写入 fstab** | 多磁盘环境设备名（`/dev/vdb`）可能漂移，UUID 永远唯一 |
| **`nofail` 挂载** | 数据盘暂不可用不影响系统启动 |
| **先 `blkid` 再 `mkfs`** | 避免重复格式化已用磁盘 |

``` bash
kubectl apply -f cloudinit-secret.yaml
```

---

## 5. 单 VM 验证

在跑 50 并发之前，先创建 1 个 VM 验证整条链路。

### 5.1 磁盘准备

``` yaml
# 系统盘：从基础镜像 PVC 克隆
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: vm1-system-disk
spec:
  source:
    pvc:
      name: openeuler-disk
      namespace: default
  storage:
    accessModes: [ReadWriteOnce]
    storageClassName: local-path
    resources:
      requests:
        storage: 40Gi
---
# 数据盘：空白磁盘
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: data-disk-vm1
spec:
  source:
    blank: {}
  storage:
    accessModes: [ReadWriteOnce]
    storageClassName: local-path
    resources:
      requests:
        storage: 40Gi
```

### 5.2 VM 定义

``` yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm1
  labels:
    kubevirt.io/os: linux
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        cpu:
          cores: 4
        firmware:
          bootloader:
            efi:
              secureBoot: false
        devices:
          disks:
          - bootOrder: 1
            disk:
              bus: virtio
            name: system-disk
          - disk:
              bus: virtio
            name: data-disk
          - disk:
              bus: virtio
              readonly: true
            name: cloudinitdisk
          interfaces:
          - name: default
            masquerade: {}
        resources:
          requests:
            memory: 4Gi
      networks:
      - name: default
        pod: {}
      volumes:
      - name: system-disk
        persistentVolumeClaim:
          claimName: vm1-system-disk
      - name: data-disk
        persistentVolumeClaim:
          claimName: data-disk-vm1
      - name: cloudinitdisk
        cloudInitNoCloud:
          secretRef:
            name: cloudinit
```

``` bash
kubectl apply -f vm1-disks.yaml
kubectl apply -f vm1.yaml

kubectl get vmi vm1 -w
# Pending → Scheduling → Scheduled → Running

# 登录验证
virtctl console vm1
```

---

## 6. 50 并发压力测试

### 6.1 CDI Clone 流程

开启 `HonorWaitForFirstConsumer` 后，每个 VM 的启动经过以下步骤：

```
1. DataVolume 创建 PVC (WFFC) → PVC: Pending
2. VM controller 创建 virt-launcher Pod → Pod: Pending
3. Scheduler 调度 Pod → local-path 创建 PV → PVC: Bound
4. CDI 检测到 Pod 消费 PVC → 创建 clone Pod → 流式拷贝
   → DataVolume: CloneScheduled → CloneInProgress → Succeeded
5. Pod 启动 virt-launcher → VMI: Scheduled → Running
```

### 6.2 批量生成 VM

``` bash
#!/bin/bash
# generate-vms.sh
N=${1:-50}

for i in $(seq -w 1 $N); do
  VM_NAME="stress-vm-${i}"
  DV_NAME="${VM_NAME}-disk"

  # DataVolume
  cat > vms/${VM_NAME}-dv.yaml <<YAML
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: ${DV_NAME}
spec:
  source:
    pvc:
      name: openeuler-disk
      namespace: default
  storage:
    accessModes: [ReadWriteOnce]
    storageClassName: local-path
    resources:
      requests:
        storage: 40Gi
YAML

  # VM: 2C4G
  cat > vms/${VM_NAME}-vm.yaml <<YAML
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: ${VM_NAME}
  labels:
    stress-test: "true"
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        stress-test: "true"
    spec:
      domain:
        cpu:
          cores: 2
        firmware:
          bootloader:
            efi:
              secureBoot: false
        devices:
          disks:
          - bootOrder: 1
            disk:
              bus: virtio
            name: system-disk
          - disk:
              bus: virtio
              readonly: true
            name: cloudinitdisk
          interfaces:
          - name: default
            masquerade: {}
        resources:
          requests:
            memory: 4Gi
      networks:
      - name: default
        pod: {}
      volumes:
      - name: system-disk
        persistentVolumeClaim:
          claimName: ${DV_NAME}
      - name: cloudinitdisk
        cloudInitNoCloud:
          secretRef:
            name: cloudinit
YAML
done
```

``` bash
bash generate-vms.sh 50
```

### 6.3 运行测试

``` bash
#!/bin/bash
# run-test.sh
N=${1:-50}

# 记录开始时间
START_TS=$(date +%s)
kubectl apply -f vms/

# 轮询等待所有 VM Running
while true; do
  ELAPSED=$(( $(date +%s) - START_TS ))
  RUNNING=$(kubectl get vmi -l stress-test=true --no-headers 2>/dev/null | grep Running | wc -l)

  echo "[${ELAPSED}s] Running: ${RUNNING}/${N}"

  [ "$RUNNING" -ge "$N" ] && break
  [ "$ELAPSED" -ge 1800 ] && echo "Timeout!" && break
  sleep 15
done

echo "All VMs Running! Total: $(( $(date +%s) - START_TS ))s"
```

测试过程中可观察中间状态：

``` bash
# DataVolume 克隆进度
kubectl get dv -l stress-test=true
# stress-vm-01-disk   CloneInProgress   50.61%
# stress-vm-02-disk   Succeeded
# ...

# CDI clone pod
kubectl get pods | grep cdi-upload
# cdi-upload-stress-vm-31-disk   1/1   Running
# cdi-upload-stress-vm-32-disk   1/1   Running
```

---

## 7. 测试结果

```
最终状态:
  50/50 VM Running      ✅
  50/50 DataVolume Succeeded ✅

节点分布:
  全部运行在 192.168.55.2 (64C/256GB worker)

资源使用:
  CPU:     3523m / 64000m (5%)
  Memory:  39813Mi / 256Gi (15%)
  Disk:    103G / 295G (37%)

启动时间分布:
  第一批 13 个:  4-6 min
  第二批 12 个:  8-12 min
  第三批 10 个:  14-17 min
  尾部 15 个:   18-26 min
```

CDI clone 串行执行，后续 VM 的克隆排队等待，尾部 VM 等待时间最长。

### 系统负载峰值

| 指标 | 值 | 评价 |
|---|---|---|
| run queue | 55 | 64 核，合理 |
| blocked (I/O) | 50 | clone I/O 等待正常 |
| iowait | 35% | 有压力但未过载 |
| swap | 0 | ✅ |
| free memory | 173GB | 余量充足 |

---

## 8. 关键设计决策

| 决策 | 为什么 | 坑点 |
|---|---|---|
| **local-path + WFFC** | 不做复杂存储，PV 在 Pod 节点本地创建，最简单可靠 | PV 和 Pod 必须同节点 |
| **HonorWaitForFirstConsumer** | CDI 等 Pod 绑定 PVC 后再开始克隆，避免跨节点挂载 | 需要理解 WFFC 语义 |
| **HTTP 导入 + xz** | 传输量小（477MB vs 1.5GB），CDI 原生支持流式解压 | xz 解压 CPU 开销 |
| **PVC 克隆模式** | 基础镜像只下载一次，后续 VM 秒级获得系统盘 | 克隆是串行的 |
| **UUID fstab** | 多磁盘设备名可能漂移，UUID 永远唯一 | `blkid` 必须在磁盘就绪后 |
| **`nofail` 挂载** | 数据盘不可用不影响系统启动 | — |

---

## 9. 面试高频 5 题

**1. KubeVirt 和传统 KVM 虚拟化的区别？**

KubeVirt 在 KVM 之上加了一层 K8s 抽象：VM 由 K8s API 管理（`kubectl get vm`），调度走 kube-scheduler，网络走 CNI（Pod 网络），存储走 PVC。底层还是 QEMU/KVM，但运维体验和容器一致。

**2. virt-handler、virt-launcher、virt-controller 的职责？**

`virt-handler` 是每节点的 DaemonSet agent，负责操作本节点 VM；`virt-launcher` 是每个 VM 的 Pod，内嵌 QEMU 进程；`virt-controller` 是集中控制器，管理 VM 生命周期状态机。

**3. DataVolume 的 WFFC（WaitForFirstConsumer）机制？**

WFFC = 延迟绑定。PVC 创建后不立即分配 PV，等到有 Pod 真正使用时才创建 PV。好处是 PV 一定创建在 Pod 所在节点，避免跨节点挂载。CDI 的 `HonorWaitForFirstConsumer` 就是配合这个机制——Pod 绑定 PVC 后才开始导入/克隆镜像。

**4. PVC 克隆 vs 容器镜像导入，哪个好？**

PVC 克隆更快（本地文件拷贝），但源 PVC 和目标 PVC 必须在同一节点。容器镜像导入适用跨节点场景，但需要 Registry 中转。本实验用 local-path 存储，PV 在同节点，PVC 克隆是最优解。

**5. 50 个 VM 为什么全调度到一个节点？**

`local-path` 存储的 PV 必须在 Pod 所在节点创建。基础镜像 PVC（openeuler-disk）在 192.168.55.2 上，克隆 PVC 必须在同一节点才能 `dd` 拷贝。调度器感知 WFFC 约束后，把所有 VM 都调度到了 55.2。如果要分散，需要把基础镜像复制到每个节点，或者用共享存储（Ceph/NFS）。

---

## 总结

```
KubeVirt + CDI 安装
     ↓
HTTP 导入基础镜像 (xz 压缩, 477MB)
     ↓
PVC 克隆创建 VM 系统盘
     ↓
单 VM 验证 (Cloud-Init SSH + 数据盘 UUID fstab)
     ↓
50 并发压力测试 (全 Running, 26 分钟内)
     ↓
节点资源: CPU 5%, Mem 15%, Disk 37%
```

KubeVirt 让 VM 管理变得像 Pod 一样简单。local-path + PVC 克隆模式对于中小规模 VM 完全够用，50 并发轻松扛住。
