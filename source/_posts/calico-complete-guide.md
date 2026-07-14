---
title: Calico 完整掌握 — 容器网络从入门到生产
date: 2026-07-14 18:00:00
tags:
  - Kubernetes
  - Calico
  - 网络
  - CNI
categories:
  - 云原生
---

## 前言

如果把 K8s 集群比作一座城市，Calico 就是这座城市的**交通系统**——它决定了 Pod 之间怎么通信、哪些路能走、哪些路被封、跨城区（跨节点）走隧道还是走高速。

本文从网络基础开始，逐步深入到 BGP、网络策略、生产排障。每个阶段都有原理图解 + 实战命令。

> 前置条件：一个 K8s 集群（minikube / kind / k3s 均可），已安装 `kubectl`。

---

## 一、容器网络基础铺垫

### 1.1 K8s 网络的四大"通信需求"

一个 K8s 集群里同时存在四种通信场景，每种走的路都不一样：

```
┌──────────┐   ④外网   ┌──────────┐
│ Internet │ ←───────→ │ Ingress  │
└──────────┘           └────┬─────┘
                            │
┌──────┐  ①Pod间  ┌──────┐│③Service
│Pod A │←────────→│Pod B ││  ↓
│10.0.1│          │10.0.2││10.96.x
└──┬───┘          └──┬───┘└───────┘
   │                 │
┌──┴──────┐  ②节点  ┌┴───────┐
│ Node 1  │←──────→│ Node 2 │
│10.0.0.1 │         │10.0.0.2│
└─────────┘         └────────┘
```

| 序号 | 通信 | 说明 | 谁负责 |
|---|---|---|---|
| ① | Pod ↔ Pod | 同节点或跨节点 | CNI（Calico） |
| ② | Pod ↔ 节点 | Pod 访问宿主机 | CNI + 路由表 |
| ③ | Pod ↔ Service | 虚拟 IP，靠 kube-proxy 转 | kube-proxy |
| ④ | 外部 ↔ Pod | 南北向流量 | Ingress + CNI |

> **核心问题**：Pod IP 是集群内部虚拟 IP，跨节点时物理网络不认识。CNI 的使命就是解决"Pod IP 怎么跨节点路由"。

### 1.2 CNI 是什么？怎么工作的？

CNI（Container Network Interface）是 K8s 定义的一套**标准接口**，规定了"创建 Pod 时调什么、传什么参数、返回什么结果"。

调用流程：

```
kubelet 创建 Pod
  │
  ├─ 读 /etc/cni/net.d/*.conf  → 找到用哪个 CNI 插件
  │
  ├─ 调 CNI ADD（传 Pod 名称、NS、网卡名）
  │     │
  │     ├─ 分配 IP
  │     ├─ 创建 veth pair（一头插 Pod，一头插宿主机）
  │     ├─ 配路由表
  │     └─ 返回结果：IP、网关、DNS
  │
  └─ Pod 网络就绪
```

几个 CNI 插件对比，秒懂选型：

| 插件 | 原理 | 性能 | 适用 |
|---|---|---|---|
| **Flannel** | 所有 Pod 走 overlay 隧道（VXLAN） | 一般 | 简单场景 |
| **Calico** | BGP 路由直连 / IPIP 隧道可选 | 高 | 生产、大集群 |
| **Cilium** | eBPF 数据面，内核态转发 | 极高 | 超大规模、可观测 |

> **关键区分**：Flannel 只负责"通"，Calico 同时负责"通 + 安全策略"。所以生产环境要么直接上 Calico，要么 Flannel + Calico 策略层。

---

## 二、Calico 整体架构

### 2.1 四大组件一张图

```
┌───────────────────────────────────────────────┐
│                  Calico 架构                    │
│                                                │
│  ┌──────────┐    ┌──────────────────────┐      │
│  │  etcd /  │←──│ calico-kube-         │      │
│  │ K8s CRD  │    │ controllers          │      │
│  │ (配置存储)│    │ (策略控制器)          │      │
│  └──────────┘    └──────────────────────┘      │
│        ↑                                      │
│  ┌─────┴──────────────────────────────┐       │
│  │         calico-node（每节点一个）     │       │
│  │  ┌─────────┐  ┌──────────────────┐ │       │
│  │  │  Felix   │  │      BIRD        │ │       │
│  │  │ (策略执行)│  │   (路由广播)     │ │       │
│  │  │ iptables │  │   BGP 协议       │ │       │
│  │  └─────────┘  └──────────────────┘ │       │
│  └────────────────────────────────────┘       │
└───────────────────────────────────────────────┘
```

| 组件 | 作用 | 类比 |
|---|---|---|
| **Felix** | 读策略 → 翻译成 iptables 规则 → 写到内核 | 交警，管"谁不能走" |
| **BIRD** | 跑 BGP 协议，把本节点 Pod 网段广播给所有节点 | 导航，告诉别人"我这有条路" |
| **calico-node** | Felix + BIRD 的容器壳，每节点一个 DaemonSet Pod | 每台机器上的交通局 |
| **calico-kube-controllers** | 监控集群状态，同步 NetworkPolicy、IPAM | 交通管理局 |

### 2.2 数据存储：etcd vs K8s CRD

Calico 3.x 以后默认用 **K8s CRD** 存储配置，不再需要独立 etcd。你 `kubectl get ippool` 看到的资源就是它存的：

``` bash
kubectl get crd | grep calico
# bgpconfigurations.crd.projectcalico.org
# felixconfigurations.crd.projectcalico.org
# ippools.crd.projectcalico.org
# networkpolicies.crd.projectcalico.org
```

### 2.3 安装 Calico

``` bash
# 官方最新 manifest（替换 <VERSION> 为实际版本）
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/calico.yaml

# 等待所有 calico-node Pod 就绪
kubectl -n kube-system get pods -l k8s-app=calico-node -w
# 每节点一个，都 Running 即安装完成

# 安装 calicoctl 命令行工具
curl -L https://github.com/projectcalico/calico/releases/latest/download/calicoctl-linux-amd64 -o calicoctl
chmod +x calicoctl
sudo mv calicoctl /usr/local/bin/
```

---

## 三、Calico 三种通信模式

这是 Calico 最重要的概念，三种模式决定了 Pod 跨节点通信走什么路径。

### 3.1 IPIP 隧道模式（默认）

**原理**：Pod 数据包外面再套一层 IP 头，封装成宿主机 IP 之间的包，走物理网络传输。

```
原始包：[Pod A IP → Pod B IP] [TCP DATA]
          ↓ 封装
隧道包：[Node1 IP → Node2 IP] [Pod A IP → Pod B IP] [TCP DATA]
          ↑ 外层（物理网络可见）  ↑ 内层（隧道解封后还原）
```

**优缺点**：
- 优：物理网络不需要认识 Pod IP，能通就行
- 缺：多了 20 字节 IP 头，CPU 做封装/解封，性能损耗约 10%-15%

**适用**：节点跨子网、物理网络不可控的云环境。

### 3.2 BGP 直连模式

**原理**：不走隧道，Calico 把 Pod 网段通过 BGP 协议广播给所有节点。节点直接往目标 Pod IP 发包，物理路由器认识这个路由。

```
Node1 的 BIRD 广播："10.244.1.0/24 的 Pod 在我这台机器上"
Node2 的 BIRD 收到 → 写路由表 → 往 10.244.1.x 的包直接发给 Node1
```

**优缺点**：
- 优：无封装开销，性能最高，接近裸金属
- 缺：要求物理网络支持 BGP 或二层可达（同一子网）

**适用**：机房自建集群、物理网络可控。

### 3.3 VXLAN 模式

**原理**和 IPIP 类似，都是隧道封装，但 VXLAN 用 MAC-in-UDP 代替 IP-in-IP。某些云环境禁止 IPIP 协议但允许 UDP。

### 3.4 三种模式对比 & 如何切换

| 模式 | 封装开销 | 性能 | 适用场景 |
|---|---|---|---|
| IPIP | 20B IP 头 | 中 | 跨子网、云环境、默认 |
| BGP | 无 | 高 | 同子网、自建机房 |
| VXLAN | 50B VXLAN头 | 中 | IPIP 被禁用的云 |

切换方法（改为 BGP 模式）：

``` bash
# 方法一：修改 calico-node 环境变量
kubectl set env daemonset/calico-node -n kube-system \
  CALICO_IPV4POOL_IPIP=Never

# 方法二：改 IPPool
calicoctl get ippool -o yaml | sed 's/ipipMode: Always/ipipMode: Never/' | calicoctl apply -f -

# 查看当前模式
calicoctl get ippool -o wide
# IPIP-MODE: Always/CrossSubnet/Never
```

> `CrossSubnet` 是聪明模式：同子网走 BGP 直连，跨子网才封 IPIP。兼顾性能和兼容性。

---

## 四、Calico IP 地址管理（IPAM）

### 4.1 IPPool 原理

Calico 为每个节点分配一个 `/26` 子网（64 个 IP），节点内部的 Pod 从这个子网分配 IP。默认从 `192.168.0.0/16` 这个大池子切分。

``` bash
# 查看 IP 池
calicoctl get ippool -o wide
# NAME        CIDR              NAT    IPIP-MODE
# default    192.168.0.0/16    true   Always
```

### 4.2 Pod 固定 IP

有些场景需要 Pod IP 不变（如数据库主从），在 Pod 注解中指定：

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  annotations:
    cni.projectcalico.org/ipAddrs: "[\"192.168.1.100\"]"
spec:
  containers:
    - name: nginx
      image: nginx
```

> **坑**：指定 IP 必须在 IPPool 范围内且未被占用，否则 Pod 起不来。

### 4.3 IP 池扩容

``` bash
# 新增一个 IPPool
cat <<EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: big-pool
spec:
  cidr: 10.100.0.0/16
  ipipMode: Always
  natOutgoing: true
EOF

# 查看现有 IPPool
calicoctl get ippool
```

### 4.4 IP 耗尽排查

``` bash
# 查看各节点 IP 占用
calicoctl ipam show

# 查看某个 IP 块的使用情况
calicoctl ipam show --show-blocks

# 释放泄漏的 IP 块（⚠ 危险操作，确认该块无 Pod 使用）
calicoctl ipam release --cidr=192.168.1.64/26
```

---

## 五、Calico 网络策略（NetworkPolicy）

网络策略是 Calico 的杀手锏——**精确控制哪个 Pod 能访问哪个 Pod**。

### 5.1 快速实验：拒绝所有入站

``` yaml
# deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  podSelector: {}          # 选中所有 Pod
  policyTypes:
    - Ingress
  ingress: []              # 空 = 拒绝所有入站
```

``` bash
kubectl apply -f deny-all.yaml

# 验证：两个 Pod 之间 ping 不通了
kubectl exec pod-a -- ping -c 2 <pod-b-ip>
# 超时
```

### 5.2 白名单：只允许特定来源

``` yaml
# allow-from-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: backend       # 目标：带 backend 标签的 Pod
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend   # 来源：仅 frontend 标签的 Pod
      ports:
        - protocol: TCP
          port: 8080          # 端口：仅 8080
```

### 5.3 命名空间级隔离

``` yaml
# deny-from-other-ns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-other-ns
  namespace: production
spec:
  podSelector: {}
  ingress:
    - from:
        - podSelector: {}     # ✅ 同 namespace 内可互访
        # 没有 namespaceSelector = 拒绝所有其他 namespace
```

### 5.4 Calico 全局网络策略（GlobalNetworkPolicy）

K8s 原生 NetworkPolicy 是 namespace 级别的。Calico 的 GlobalNetworkPolicy 是**集群级别**的，优先级更高：

``` yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: block-external-ssh
spec:
  order: 10            # 优先级，数字越小越高
  selector: all()
  types:
    - Egress
  egress:
    - action: Deny
      protocol: TCP
      destination:
        nets:
          - 0.0.0.0/0
        ports:
          - 22           # 禁止所有 Pod SSH 出外网
    - action: Allow      # 其他放行
```

> `order` 决定了多条策略的执行顺序。值小的先执行，`Deny` + 低 order = 黑名单优先。

### 5.5 策略调试三步法

``` bash
# 1. 看 Pod 被哪些策略选中
kubectl describe networkpolicy -n <ns>

# 2. 抓包定位（Pod 内）
kubectl exec <pod> -- tcpdump -i eth0 -nn

# 3. 从宿主机看 iptables（Calico 策略最终落到这里）
iptables -L -n -v | grep cali
```

---

## 六、BGP 深度篇

### 6.1 BGP 是什么？

BGP 本质上就是**路由器之间互相通报"我这边有哪些网段"**。Calico 的 BIRD 组件让每台宿主机变成一台"BGP 路由器"。

```
Node1 说："192.168.1.0/26 的 Pod 都在我这"
Node2 说："192.168.2.0/26 的 Pod 都在我这"
Node3 说："192.168.3.0/26 的 Pod 都在我这"
```

每台机器都记下全网 Pod 路由，发数据直连过去，不走隧道，不封包。

### 6.2 全节点 BGP（Full Mesh）的问题

默认每两个节点之间建 BGP 邻居，节点数多时连接数爆炸：

```
3 节点：3 条连接
10 节点：45 条连接
100 节点：4950 条连接  ← BIRD 吃不消
```

### 6.3 路由反射器（Route Reflector）

**思路**：指定 2-3 个节点当"传话筒"，其他节点只跟它们建邻居。

```
     RR1 ←────────→ RR2
   ↗  ↑  ↖        ↗  ↑  ↖
 Node  Node       Node  Node
```

Node 只需跟 RR 建一条连接，RR 负责把路由转发给所有 Node。100 节点只需 200 条连接。

**配置路由反射器**：

``` bash
# 给 2 个节点打标签，作为 RR
kubectl label node node1.calico-rr=true
kubectl label node node2.calico-rr=true

# 在 BGPConfiguration 中指定
cat <<EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  nodeToNodeMeshEnabled: false   # 关掉全节点 Mesh
  asNumber: 64512
EOF

# 配置每个节点的 BGP Peer（指向 RR）
cat <<EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: rr-peers
spec:
  nodeSelector: "!has(calico-rr)"   # 非 RR 节点
  peerSelector: has(calico-rr)      # 连接 RR 节点
EOF
```

---

## 七、生产排障

### 7.1 Pod 跨节点不通 — 排障流程

``` bash
# Step 1：Pod 本身网络正常吗？
kubectl exec <pod> -- ip addr show eth0   # 有 IP 吗？

# Step 2：同节点其他 Pod 能通吗？
kubectl exec <pod-a> -- ping <pod-b-ip>   # 同节点：不通说明 CNI 挂了

# Step 3：跨节点能 ping 通对方 Pod 吗？
kubectl exec <pod-a> -- ping <pod-c-ip>   # 跨节点：不通看路由

# Step 4：宿主机路由表正确吗？
ip route | grep cali
# 应该有到目标 Pod 网段的路由

# Step 5：隧道（IPIP）封包正常吗？
tcpdump -i tunl0 -nn   # 抓 IPIP 隧道流量
# 看不到包 → 路由问题；看到包但对方没响应 → 策略 Drop

# Step 6：Calico 策略拦截了吗？
calicoctl get networkpolicy -A
calicoctl get globalnetworkpolicy

# Step 7：Calico 组件正常吗？
kubectl -n kube-system get pods -l k8s-app=calico-node
kubectl -n kube-system logs <calico-node-pod> -c calico-node | tail -50
```

### 7.2 常见故障速查

| 现象 | 可能原因 | 检查命令 |
|---|---|---|
| Pod 没有 IP | CNI 插件未就绪 | `kubectl -n kube-system get pods -l k8s-app=calico-node` |
| 同节点 Pod 通，跨节点不通 | IPIP 隧道被封 / 路由错误 | `ip route`、`tcpdump -i tunl0` |
| 时通时不通 | IP 冲突 | `calicoctl ipam check` |
| 新 Pod 一直 Pending | IP 池耗尽 | `calicoctl ipam show` |
| Service 不通 | kube-proxy / iptables | `iptables -t nat -L \| grep <svc-name>` |

### 7.3 抓包分析 IPIP 隧道

``` bash
# 在源节点抓 tunl0（IPIP 隧道接口）
tcpdump -i tunl0 -nn -w /tmp/ipip.pcap

# 在物理网卡抓封装后的包
tcpdump -i eth0 proto 4 -nn   # IPIP 是 IP 协议号 4
```

---

## 八、Calico 面试高频 15 题

**1. Calico 和 Flannel 的区别？**
Flannel 只做网络连通（overlay 隧道），Calico 做连通 + 网络策略。Flannel 简单但性能差，Calico 支持 BGP 直连无隧道损耗。

**2. Calico 三种模式怎么选？**
同子网/机房 → BGP；跨子网/云环境 → IPIP CrossSubnet；IPIP 被禁 → VXLAN。

**3. IPIP 隧道原理？**
Pod 包外再封一层宿主机 IP 头，物理网络只看到宿主机 IP。封装 20 字节开销。

**4. BIRD 是什么？**
BIRD 是开源的 BGP 路由守护进程，Calico 用它来广播 Pod 网段路由。

**5. 路由反射器解决什么问题？**
全节点 BGP Mesh 连接数 O(n²)，RR 模式降到 O(n)，大规模集群必备。

**6. Calico 策略怎么落到 iptables？**
Felix 读 NetworkPolicy → 翻译成 iptables 规则 → 写入内核 netfilter。每个 Pod 的 veth 接口链上都有对应规则。

**7. Pod 固定 IP 怎么做？**
Pod annotation `cni.projectcalico.org/ipAddrs` 指定，IP 必须在 IPPool 内且未被占用。

**8. IP 池耗尽了怎么办？**
新增 IPPool 或扩容现有 Pool 的 CIDR。`calicoctl ipam show` 查看使用量。

**9. Calico 支持哪些数据存储？**
K8s CRD（默认，3.x+）和 etcd。CRD 模式更简单，直接用 kubectl 管理。

**10. Felix 挂了影响已有 Pod 通信吗？**
不影响。Felix 只管策略更新，已有的 iptables 规则继续生效。新 Pod 的策略会延迟生效。

**11. Calico 为什么需要 etcd？**
3.x 之前默认用 etcd；3.x 之后默认用 K8s CRD，不再需要独立 etcd。

**12. BGP 和 IPIP 能混用吗？**
能。`ipipMode: CrossSubnet` 就是混用——同子网 BGP 直连，跨子网 IPIP 隧道。

**13. NetworkPolicy 和 Calico GlobalNetworkPolicy 什么区别？**
前者 namespace 级别，后者集群级别。后者 order 可以控制优先级。

**14. Pod 重启 IP 会变吗？**
默认会变（从 IPAM 重新分配）。如需固定，用 annotation 指定。

**15. Calico 在生产上最常见的坑？**
IPIP 隧道 MTU 问题（封装后包变大，需调小 Pod MTU 或调大物理 MTU）；全节点 BGP 路由爆炸（忘配 RR）；IP 池用完没监控。

---

## 总结

从通信模型到 BGP、从 IPAM 到排障，Calico 的路由表、iptables 规则和 BGP peer 是三个最核心的检查点。遇到网络问题时，依次排查这三层，基本都能定位到根因。

```
通信模型 ──→ CNI 原理 ──→ Calico 架构（Felix + BIRD）
     ↓
IPIP / BGP / VXLAN 三模式
     ↓
IPAM（IPPool / 固定 IP / 扩容排障）
     ↓
NetworkPolicy（入站 / 出站 / 命名空间隔离）
     ↓
BGP 深度（Full Mesh → Route Reflector）
     ↓
生产排障（抓包 / iptables / 路由 / IPAM）
```
