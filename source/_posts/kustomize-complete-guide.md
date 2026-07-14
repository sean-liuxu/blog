---
title: Kustomize 完整掌握 — 配置即代码的艺术
date: 2026-07-14 10:00:00
tags:
  - Kubernetes
  - Kustomize
  - DevOps
categories:
  - 云原生
---

## 前言

[上一篇文章](/2026/07/13/helm-complete-guide/) 我们掌握了 Helm——它解决的是"如何打包和分发应用"。但生产环境中还有一个棘手的问题：**同一套应用，在 dev / test / prod 环境里只有细微差异（副本数、镜像 tag、域名），能不能不复制粘贴整份 YAML？**

Kustomize 就是答案。它从 Kubernetes 1.14 起内置在 `kubectl` 中，核心思想是 **base + overlay**：一份基础配置打底，每个环境只声明差异。

| 对比维度 | Helm | Kustomize |
|---|---|---|
| 核心理念 | 模板 + 值注入 | 基础配置 + 差异化补丁 |
| 工作方式 | `{{ .Values.xxx }}` 占位符 | 原地 patch 修改 YAML |
| 配置复用 | 多 values.yaml 覆盖 | base/overlay 分层 |
| 模板语法 | Go Template（需学习） | 纯 YAML（零学习语法） |
| 原生集成 | 第三方工具 | `kubectl apply -k` 内置 |
| 最佳场景 | 对外分发的通用包 | 内部多环境配置管理 |

> **简单粗暴的区分**：对外分发的 Chart 用 Helm，内部多环境管理用 Kustomize。两者并不对立——后面会看到它们如何**组合使用**。

---

## 一、基础篇：hello world

### 1.1 确认版本

``` bash
kubectl version --client | grep Kustomize
# Kustomize Version: v5.x
```

`kubectl apply -k` 直接用，无需额外安装。

### 1.2 第一个 Kustomize 项目

创建 base 目录：

``` bash
mkdir -p ~/kustomize-demo/base
```

**vim ~/kustomize-demo/base/deployment.yaml**

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/nginx:stable
          ports:
            - containerPort: 80
```

**vim ~/kustomize-demo/base/service.yaml**

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

**vim ~/kustomize-demo/base/kustomization.yaml**

``` yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

### 1.3 运行

``` bash
# 预览渲染结果（不部署）
kubectl kustomize ~/kustomize-demo/base/

# 直接部署
kubectl apply -k ~/kustomize-demo/base/

# 验证
kubectl get all -l app=nginx

# 清理
kubectl delete -k ~/kustomize-demo/base/
```

就这么简单。`kustomization.yaml` 是入口，`resources` 列出要管理的文件。目前它只是"打包"，还没有任何差异化能力。接下来才是重点。

---

## 二、多环境篇：base + overlay

Kustomize 的核心模式：**base 写不变的部分，overlay 只写差异**。

### 2.1 创建 overlay 目录

``` bash
mkdir -p ~/kustomize-demo/overlays/{dev,test,prod}
```

**vim ~/kustomize-demo/overlays/dev/kustomization.yaml**

``` yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# 引用 base
resources:
  - ../../base

# 声明差异：副本数改为 1
replicas:
  - name: nginx
    count: 1

# 加一个 namespace
namespace: dev

# 统一加标签
commonLabels:
  env: dev
```

`replicas` 是 Kustomize 内置的快捷字段，专门用来 **原地修改 Deployment 的 replicas**，不需要写 patch 文件。

**vim ~/kustomize-demo/overlays/test/kustomization.yaml**

``` yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: test

commonLabels:
  env: test

replicas:
  - name: nginx
    count: 2

# patch 是 Kustomize 最强大的功能：精确修改 base 中的任意字段
patches:
  - patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/nginx:stable-alpine
    target:
      kind: Deployment
      name: nginx
```

> **patches 语法**：使用 JSON Patch（RFC 6902）。`op` 可以是 `replace` / `add` / `remove`，`path` 用 `/` 指到目标字段。上面这条语义是："把 nginx Deployment 的容器镜像换成 alpine 版"。

**vim ~/kustomize-demo/overlays/prod/kustomization.yaml**

``` yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: prod

commonLabels:
  env: prod

replicas:
  - name: nginx
    count: 5

patches:
  - patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/nginx:stable
      - op: add
        path: /spec/template/spec/containers/0/resources
        value:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 200m
            memory: 256Mi
    target:
      kind: Deployment
      name: nginx
```

### 2.4 渲染对比

``` bash
# 分别查看三个环境的最终 YAML
kubectl kustomize ~/kustomize-demo/overlays/dev/  | head -20
kubectl kustomize ~/kustomize-demo/overlays/test/ | grep -A2 image
kubectl kustomize ~/kustomize-demo/overlays/prod/ | grep replicas
```

你会发现 base 中的 deployment.yaml + service.yaml **原样保留**，overlay 只"覆盖"了差异部分。这是 Kustomize 和 Helm 最大的不同：没有模板语言，没有占位符。

---

## 三、核心资源生成器：ConfigMap / Secret / 镜像替换

Kustomize 不仅能 patch 已有资源，还能**自动生成** ConfigMap、Secret，并优雅地处理镜像 tag 替换。

### 3.1 ConfigMap 生成器 — 从文件生成

创建环境专属的 nginx 配置：

创建 nginx 配置目录：

``` bash
mkdir -p ~/kustomize-demo/overlays/dev/nginx-conf
```

**vim ~/kustomize-demo/overlays/dev/nginx-conf/default.conf**

``` nginx
server {
    listen 80;
    server_name localhost;
    location / {
        root /usr/share/nginx/html;
        index index.html;
        add_header X-Environment "DEV";
    }
}
```

然后在 `~/kustomize-demo/overlays/dev/kustomization.yaml` 末尾追加 configMapGenerator：

``` yaml
# ... 原有内容 ...

configMapGenerator:
  - name: nginx-config
    files:
      - nginx-conf/default.conf
```

渲染出来看看：

``` bash
# 用 yq 精确提取 ConfigMap 部分（推荐）
kubectl kustomize ~/kustomize-demo/overlays/dev/ | yq 'select(.kind=="ConfigMap")'

# 如果没有 yq，grep 用 -B（查匹配行之前）而非 -A（之后），因为 data 在 kind 上面
kubectl kustomize ~/kustomize-demo/overlays/dev/ | grep -B20 'kind: ConfigMap'
```

输出：

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config-7k4m8d5g2f   # ← 自动加了内容 hash！
  namespace: dev
data:
  default.conf: |
    server { ... }
```

**为什么要加 hash？** 假设你不小心改了 nginx 配置中的某个参数，如果 ConfigMap 名字不变（始终叫 `nginx-config`），Deployment 不会知道内容变了，Pod 继续用旧配置运行。加 hash 之后：**配置内容变了 → ConfigMap 名字变了 → Deployment 挂载的 volume 引用也自动更新为新名字 → Pod 必然重建 → 新配置自动生效。** 这也是为什么 Deployment 里可以安全地写 `name: nginx-config` —— Kustomize 渲染时会自动替换。

### 3.2 让 Deployment 挂载生成的 ConfigMap

现在的问题是：ConfigMap 名称带了随机 hash，Deployment 怎么引用？答案是 **namePrefix / nameSuffix 或直接用生成器的 name**。

Kustomize 会自动把模板中的 `nginx-config` 替换为实际带 hash 的名称，只要在 Deployment 中引用同名即可。修改 `base/deployment.yaml`，加入 volume 挂载：

``` yaml
# 在 containers 下增加
        volumeMounts:
          - name: nginx-config
            mountPath: /etc/nginx/conf.d
# 在 spec.template.spec 下增加
      volumes:
        - name: nginx-config
          configMap:
            name: nginx-config
```

**这里有一个关键问题：ConfigMap 名称已经变成 `nginx-config-7k4m8d5g2f`，Deployment 里挂载的还是 `name: nginx-config`，那不就不匹配了吗？**

这就是 Kustomize 最精妙的地方：**它会扫描所有资源，把引用了 `nginx-config` 的地方全部自动替换成带 hash 的名字。** 渲染后的 Deployment 实际上变成了：

``` yaml
# Deployment 渲染结果（关键字段）
spec:
  template:
    spec:
      volumes:
        - name: nginx-config
          configMap:
            name: nginx-config-7k4m8d5g2f   # ← 自动替换了！
```

所以 Deployment 挂载的一定能对上 ConfigMap 的名字。**写的时候用固定名，渲染后自动变成 hash 名**，既保证了引用正确，又保证了内容变更即滚动更新。

### 3.3 Secret 生成器 — 从文字面值生成

``` yaml
# 在 kustomization.yaml 中添加
secretGenerator:
  - name: db-secret
    literals:
      - DB_PASSWORD=MyS3cret!Dev
      - API_KEY=sk-dev-abc123
    # type: Opaque  # 默认值，可不写
```

渲染结果：

``` yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret-9a8b7c6d
  namespace: dev
data:
  DB_PASSWORD: TXlTM2NZXQhRGV2   # base64 编码
  API_KEY: c2stZGV2LWFiYzEyMw==
```

> **安全提醒**：这里的值是明文写在 kustomization.yaml 中的，**不要直接提交 Git**。后面会介绍如何搭配 sops 加密。

### 3.4 images — 统一替换镜像 tag

假设 CI 构建的镜像 tag 每次不同（如 git commit hash），总不能去改 deployment.yaml 吧？Kustomize 的 `images` 字段就是干这个的：

``` yaml
# 在 kustomization.yaml 中添加
images:
  - name: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/nginx
    newTag: "abc1234"   # CI 动态注入的 commit hash
```

`images` 会找到所有使用该镜像的 Deployment，把 tag 替换成 `abc1234`。这个是**声明式的镜像替换**，比 `sed` 靠谱一百倍。

prod overlay 完整示例：

``` yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: prod
commonLabels:
  env: prod

replicas:
  - name: nginx
    count: 5

images:
  - name: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/nginx
    newTag: "v1.2.3"

configMapGenerator:
  - name: nginx-config
    files:
      - nginx-conf/default.conf

patches:
  - patch: |-
      - op: add
        path: /spec/template/spec/containers/0/resources
        value:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 200m
            memory: 256Mi
    target:
      kind: Deployment
      name: nginx
```

---

## 四、进阶篇：Kustomize + Helm 混合使用

前面说过二者不互斥。企业里最典型的模式：**用 Helm 拉取外部 Chart，用 Kustomize 做环境化定制**。

### 4.1 helmChartInflator — 引入外部 Helm Chart

假设我们要引入 Bitnami 的 Redis Chart，用 Kustomize 定制。

> 注意：这个功能在 Kustomize 5+ 中通过 `helmCharts` 字段原生支持。

**overlays/prod/kustomization.yaml** 中添加：

``` yaml
# 引入外部 Helm Chart（Kustomize 会自动 helm pull + 渲染）
helmCharts:
  - name: redis
    repo: https://charts.bitnami.com/bitnami
    version: "20.0.0"
    releaseName: my-redis
    namespace: prod
    valuesFile: redis-values.yaml
```

> **国内用户注意**：如果 Bitnami 仓库访问慢，可以先用 `helm pull` 拉下来放到本地，改用 `chartHome` 指定本地路径。或者参考上篇 Helm 文章搭建本地仓库。

**overlays/prod/redis-values.yaml**

``` yaml
# Helm values 覆盖
architecture: standalone
auth:
  enabled: true
  password: "RedisP@ss"    # 演示用，生产请配合 sops 加密
master:
  persistence:
    enabled: false
    size: 8Gi
```

### 4.2 用 Kustomize patch 修改 Helm 原生资源

Helm Chart 渲染出的资源可能不完全满足需求，用 Kustomize 的 `patches` 做二次加工：

``` yaml
# 继续在 kustomization.yaml 中
patches:
  # ... 之前的 nginx patch ...

  # 给 Redis 加上 nodeSelector（Helm Chart 的 values 可能不支持）
  - patch: |-
      - op: add
        path: /spec/template/spec/nodeSelector
        value:
          disktype: ssd
    target:
      kind: Deployment
      name: my-redis-master   # Bitnami Chart 渲染出的 Deployment 名称
```

### 4.3 完整混合 overlay 示例

把 nginx（自己的 Chart）+ Redis（外部 Helm Chart）组合在一起，这是企业里最常见的模式：

```
overlays/prod/
├── kustomization.yaml       # 主入口：引用 base + helmCharts + patches
├── redis-values.yaml        # Redis 的 Helm values 覆盖
├── nginx-conf/
│   └── default.conf         # nginx 环境配置
└── secrets/                 # 加密密钥（见第六章）
```

**overlays/prod/kustomization.yaml** 完整版：

``` yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# 1. 引用自己的 base
resources:
  - ../../base

# 2. 引入外部 Helm Chart
helmCharts:
  - name: redis
    repo: https://charts.bitnami.com/bitnami
    version: "20.0.0"
    releaseName: my-redis
    namespace: prod
    valuesFile: redis-values.yaml

# 3. 环境标识
namespace: prod
commonLabels:
  env: prod

# 4. 副本数
replicas:
  - name: nginx
    count: 5

# 5. 镜像
images:
  - name: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/nginx
    newTag: "v1.2.3"

# 6. 配置生成
configMapGenerator:
  - name: nginx-config
    files:
      - nginx-conf/default.conf

# 7. 精细 patch
patches:
  - patch: |-
      - op: add
        path: /spec/template/spec/containers/0/resources
        value:
          limits: { cpu: "500m", memory: "512Mi" }
          requests: { cpu: "200m", memory: "256Mi" }
    target:
      kind: Deployment
      name: nginx
  - patch: |-
      - op: add
        path: /spec/template/spec/nodeSelector
        value:
          disktype: ssd
    target:
      kind: Deployment
      name: my-redis-master
```

渲染看看整个 prod 环境到底生成了哪些资源：

``` bash
kubectl kustomize overlays/prod/ --enable-helm
# 会看到 nginx Deployment + Service + ConfigMap + Redis 所有资源的完整 YAML
```

> `--enable-helm` 是启用 helmCharts 处理的开关。若不加这个 flag，helmCharts 部分会被跳过。

---

## 五、实操篇：用 Kustomize 部署 dev/test/prod 到集群

### 5.1 创建命名空间

``` bash
kubectl create ns dev
kubectl create ns test
kubectl create ns prod
```

### 5.2 部署三个环境

``` bash
# Dev
kubectl apply -k overlays/dev/

# Test
kubectl apply -k overlays/test/

# Prod（含 Helm Chart，需要 --enable-helm）
kubectl apply -k overlays/prod/ --enable-helm
```

### 5.3 验证环境差异

``` bash
# Dev 环境：1 个副本，NodePort
kubectl -n dev get pods
kubectl -n dev get svc

# Prod 环境：5 个副本，集群内有 Redis
kubectl -n prod get pods
kubectl -n prod get all
```

### 5.4 查看实际生效的配置

``` bash
# 对比 base 和某环境 overlay 的差异
diff <(kubectl kustomize base/) <(kubectl kustomize overlays/prod/ --enable-helm) | head -40
```

---

## 六、安全篇：SOPS 加密 Secret

和 Helm 那篇一样，Secret 的明文不能提交 Git。Kustomize 原生支持 SOPS 加密。

### 6.1 安装 SOPS + Kustomize SOPS 插件

``` bash
# 安装 sops（参考 Helm 文章 6.1 节）
brew install sops

# 安装 kustomize-sops 插件（Go 写的 Kustomize 执行器插件）
# 方式一：使用 kustomize-sops-exec
GOBIN=~/bin go install github.com/goabout/kustomize-sops-exec@latest

# 方式二：更简单 —— 直接用 kustomize 的 generators + sops
# 这里推荐方式二：用 sops 先加密，再用 Kustomize 内置的 envs/files 生成器引用
```

### 6.2 创建并加密 Secrets 文件

``` bash
# 准备明文 secret（不提交 Git）
cat > overlays/prod/secrets/.env <<EOF
DB_PASSWORD=MyP@ssw0rd!Prod
REDIS_PASSWORD=RedisS3cret
API_KEY=sk-prod-xyz789
EOF

# 加密（使用 PGP，前提是你已按 Helm 那篇文章生成过密钥）
sops --encrypt \
  --pgp YOUR_GPG_KEY_FINGERPRINT \
  overlays/prod/secrets/.env > overlays/prod/secrets/.env.enc

# 删除明文
rm overlays/prod/secrets/.env
```

加密后的 `.env.enc` 内容：

```
DB_PASSWORD=ENC[AES256_GCM,data:xxxx,iv:xxxx,tag:xxxx,type:str]
REDIS_PASSWORD=ENC[AES256_GCM,data:xxxx,iv:xxxx,tag:xxxx,type:str]
...
sops:
  pgp:
    - created_at: "2026-07-14T..."
      enc: |-
        -----BEGIN PGP MESSAGE-----
        ...
```

### 6.3 在 kustomization.yaml 中引用加密文件

``` yaml
# 加在 overlays/prod/kustomization.yaml 中
secretGenerator:
  - name: app-secrets
    envs:
      - secrets/.env.enc    # Kustomize 调用 sops 解密后生成 Secret
```

> Kustomize 会自动检测 `.enc` 后缀，调用 `sops` 解密后再生成 Secret。CI 机器上需要安装 `sops` 并有对应的解密密钥。

### 6.4 .gitignore 规则

```
# .gitignore
secrets/.env
!secrets/*.enc
```

---

## 七、交付篇：同时输出 Helm 包 + Kustomize 清单

企业级交付的终极形态：**对外给 Helm Chart、对内给 Kustomize overlay**。

```
my-app/
├── chart/                    # Helm Chart（对外分发）
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       └── ...
├── kustomize/                # Kustomize 配置（内部多环境）
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── dev/
│       ├── test/
│       └── prod/
├── .gitlab-ci.yml
└── Makefile
```

**Makefile** 统一构建入口：

``` makefile
# 打包 Helm Chart
helm-package:
	helm package chart/ -d dist/

# 渲染各环境 Kustomize 清单
kustomize-dev:
	kubectl kustomize kustomize/overlays/dev/ > dist/dev.yaml

kustomize-test:
	kubectl kustomize kustomize/overlays/test/ > dist/test.yaml

kustomize-prod:
	kubectl kustomize kustomize/overlays/prod/ --enable-helm > dist/prod.yaml

# 一键构建全部
build: helm-package kustomize-dev kustomize-test kustomize-prod
```

``` bash
make build
# dist/
# ├── my-app-0.1.0.tgz    ← Helm 包
# ├── dev.yaml             ← Kustomize 渲染后的 dev 环境 YAML
# ├── test.yaml
# └── prod.yaml
```

> 两种产物各有用处：`.tgz` 扔到 Harbor 供其他团队 `helm install`；`*.yaml` 给 ArgoCD 做 GitOps 部署。

---

## 八、总结：Helm vs Kustomize 选型指南

| 场景 | 推荐工具 |
|---|---|
| 开发一个要分发给社区/其他团队的通用应用 | Helm |
| 同一个应用部署到 3+ 环境，每个环境差异小 | Kustomize |
| 需要引入开源组件（如 Redis、PostgreSQL） | Helm（或 Kustomize + helmCharts） |
| GitOps 工作流（ArgoCD / Flux） | **两者皆可**，Kustomize 更原生 |
| 团队不熟悉 Go Template | Kustomize（纯 YAML） |
| 需要模板化的复杂逻辑 | Helm |

**最终推荐的企业架构：**

```
Helm Chart（制作通用包）
    ↓
Kustomize overlay（每个环境差异化）
    ↓
ArgoCD / Flux（GitOps 自动同步）
    ↓
Kubernetes 集群
```

把 Helm 当作"基础包"，Kustomize 做"环境适配层"，二者各司其职，互不替代。

回头看你的学习路径：

```
Helm: install/upgrade  →  Chart 结构  →  手写 Chart  →  多环境  →  secrets  →  Harbor
                                    ↓ 对比理解 ↓
Kustomize: base/overlay  →  configMap/secret 生成器  →  patches  →  helmCharts  →  sops  →  交付
```

---

## 常用命令速查

| 场景 | 命令 |
|---|---|
| 渲染预览 | `kubectl kustomize <dir>` |
| 部署 | `kubectl apply -k <dir>` |
| 删除 | `kubectl delete -k <dir>` |
| 渲染含 Helm Chart | `kubectl kustomize <dir> --enable-helm` |
| 修改副本数 | `replicas:` 字段（kustomization.yaml 内） |
| 替换镜像 tag | `images:` 字段 |
| JSON Patch | `patches:` + `op: replace/add/remove` |
| 生成 ConfigMap | `configMapGenerator:` + `files:` / `literals:` |
| 生成 Secret | `secretGenerator:` + `literals:` / `envs:` |
| 加密 Secret | `sops --encrypt` → `.env.enc` → `secretGenerator.envs` |
| 引入 Helm Chart | `helmCharts:` + `repo:` + `valuesFile:` |

---

希望这篇教程帮你彻底搞懂 Kustomize，并能和 Helm 搭配使用。两篇一起，覆盖了 Kubernetes 应用配置管理的核心能力。
