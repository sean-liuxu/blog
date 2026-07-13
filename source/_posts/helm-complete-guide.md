---
title: Helm 完整掌握 — 从零到生产
date: 2026-07-13 21:00:00
tags:
  - Kubernetes
  - Helm
  - DevOps
  - 教程
categories:
  - 云原生
---

## 前言

Helm 是 Kubernetes 的包管理器，就像 apt/yum 之于 Linux、npm 之于 Node.js。它把一组 K8s 资源打包成一个 Chart，让部署、升级、回滚变得简单可控。

本文从零开始，通过一系列**前后关联**的实操任务，带你循序渐进掌握 Helm 的核心用法。建议打开终端跟着敲。

> 前置条件：一个可用的 Kubernetes 集群（minikube / kind / k3s 均可），已安装 `kubectl`。

---

## 一、基础篇：认识 Helm

### 1.1 安装 Helm

``` bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Windows (choco)
choco install kubernetes-helm
```

验证安装：

``` bash
helm version
# version.BuildInfo{Version:"v3.17+", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.24"}
```

### 1.2 搭建本地 Chart 仓库

由于国内访问 Bitnami 等国外仓库很慢，我们**从零搭建一个本地 Chart 仓库**，同时也理解仓库的底层原理。

一个 Helm 仓库本质上就是一个 HTTP 服务器 + `index.yaml` 索引文件 + 打包好的 `.tgz` 文件。

**第一步：创建目录结构**

``` bash
mkdir -p ~/helm-repo/charts
cd ~/helm-repo
```

**第二步：快速准备一个 Chart**

先用 `helm create` 生成一个脚手架 Chart 作为仓库的第一个"货物"：

``` bash
helm create my-nginx
```

这会生成一个标准的 Chart 目录。我们稍作修改，把镜像换成国内可用的：

``` bash
# 修改 values.yaml 中的镜像为华为云 SWR 国内镜像
sed -i 's/repository: nginx/repository: ddn-k8s\/docker.io\/nginx/' my-nginx/values.yaml
sed -i 's/tag: ""/tag: "stable"/' my-nginx/values.yaml

# 修改后检查
grep -A2 'image:' my-nginx/values.yaml
# image:
#   repository: ddn-k8s/docker.io/nginx
#   tag: "stable"
```

> 镜像地址是 `swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/nginx:stable`。Chart 的 `values.yaml` 里，`registry` + `repository` + `tag` 三段组合成完整镜像路径。

**第三步：打包并生成仓库索引**

``` bash
# 将 Chart 打包成 .tgz，输出到 charts 目录
helm package my-nginx -d charts/
# 生成: charts/my-nginx-0.1.0.tgz

# 生成仓库索引文件
helm repo index charts/ --url http://127.0.0.1:8879/charts
# 生成: charts/index.yaml

# 看看索引文件内容
cat charts/index.yaml
```

**第四步：启动本地仓库服务**

``` bash
# 用 Python 起一个简单的 HTTP 服务（新开一个终端窗口）
cd ~/helm-repo
python3 -m http.server 8879
```

**第五步：添加本地仓库并用它安装**

``` bash
# 添加本地仓库
helm repo add local http://127.0.0.1:8879/charts

# 查看已有仓库
helm repo list
# local    http://127.0.0.1:8879/charts

# 搜索本地仓库中的 Chart
helm search repo local/
```

> **仓库原理小结**：`helm repo add` 只是记下了一个 URL，`helm search` 去那个 URL 下载 `index.yaml` 进行检索，`helm install` 则下载对应的 `.tgz` 包并部署到集群。

### 1.3 第一个 Helm 部署：安装本地 Chart

``` bash
# 从本地仓库安装
helm install my-nginx local/my-nginx

# 查看已安装的 Release
helm list

# 查看部署的资源
kubectl get all -l app.kubernetes.io/instance=my-nginx
```

访问验证：

``` bash
# 端口转发
kubectl port-forward svc/my-nginx 8080:80

# 另开终端测试
curl http://localhost:8080
```

### 1.4 升级、回滚与卸载

``` bash
# 查看 Release 的历史版本
helm history my-nginx

# 修改配置后升级（比如调整副本数）
helm upgrade my-nginx local/my-nginx --set replicaCount=3

# 再次查看历史，多了一条记录
helm history my-nginx

# 回滚到上一个版本
helm rollback my-nginx 1

# 卸载
helm uninstall my-nginx
```

> **理解关键概念**：
> - **Chart**：应用包（模板 + 默认值）
> - **Release**：Chart 在集群中的一个部署实例
> - **Repository**：Chart 的存储和分发仓库
> - **Revision**：每次 upgrade/rollback 产生的新版本号

---

## 二、Chart 结构篇：解剖一个 Chart

### 2.1 创建一个空白 Chart

``` bash
helm create demo-chart
tree demo-chart
```

输出：

```
demo-chart/
├── Chart.yaml          # Chart 元信息
├── values.yaml         # 默认配置值
├── charts/             # 依赖的子 Chart
├── templates/          # 模板文件
│   ├── NOTES.txt       # 安装后提示信息
│   ├── _helpers.tpl    # 模板辅助函数
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   └── tests/          # 测试
└── .helmignore
```

### 2.2 Chart.yaml — Chart 的身份证

``` yaml
apiVersion: v2
name: demo-chart
description: A Helm chart for Kubernetes
type: application          # application | library
version: 0.1.0             # Chart 自身版本
appVersion: "1.16.0"       # 应用版本
```

关键字段都在这，`version` 是 Chart 版本号，每次打包发布时递增。

### 2.3 values.yaml — 配置的入口

这是 Helm 的核心设计：**配置与模板分离**。所有可变参数都定义在 `values.yaml` 中，模板文件引用这些值。

``` yaml
replicaCount: 1

image:
  repository: nginx
  tag: ""

service:
  type: ClusterIP
  port: 80
```

### 2.4 _helpers.tpl — 模板复用

``` tpl
{% raw %}{{/*
Create a default fully qualified app name.
*/}}
{{- define "demo-chart.fullname" -}}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "demo-chart.labels" -}}
app.kubernetes.io/name: {{ include "demo-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}{% endraw %}
```

`define` 定义可复用的模板片段，`include` 在别处调用。`{{-` 和 `-}}` 用于去除模板输出前后的空白。

### 2.5 内置对象速查

模板中可用的关键对象：

| 对象 | 含义 | 示例 |
|---|---|---|
| `{% raw %}{{ .Release.Name }}{% endraw %}` | Release 名称 | `my-release` |
| `{% raw %}{{ .Release.Namespace }}{% endraw %}` | 命名空间 | `default` |
| `{% raw %}{{ .Chart.Name }}{% endraw %}` | Chart 名称 | `demo-chart` |
| `{% raw %}{{ .Chart.Version }}{% endraw %}` | Chart 版本 | `0.1.0` |
| `{% raw %}{{ .Values.replicaCount }}{% endraw %}` | values 中的值 | `1` |
| `{% raw %}{{ .Files.Get "config.json" }}{% endraw %}` | 读取 Chart 内文件 | 文件内容 |
| `{% raw %}{{ .Capabilities.KubeVersion }}{% endraw %}` | K8s 版本信息 | `v1.30` |

### 2.6 流程控制：if / range / with

**if 条件判断：**

``` yaml
{% raw %}{{- if .Values.serviceAccount.create }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "demo-chart.serviceAccountName" . }}
{{- end }}{% endraw %}
```

**range 循环遍历：**

``` yaml
{% raw %}env:
{{- range .Values.env }}
  - name: {{ .name }}
    value: {{ .value }}
{{- end }}{% endraw %}
```

对应的 values：

``` yaml
env:
  - name: DB_HOST
    value: mysql.default.svc
  - name: DB_PORT
    value: "3306"
```

**with 切换作用域：**

``` yaml
{% raw %}{{- with .Values.resources }}
resources:
  {{- toYaml . | nindent 2 }}
{{- end }}{% endraw %}
```

`with` 将当前作用域切换到指定对象，内部直接 `.` 即可访问。

### 2.7 管道与函数

管道 `|` 将前一个输出作为最后一个参数传给下一个函数：

``` yaml
{% raw %}image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"{% endraw %}
```

常用函数：

| 函数 | 用途 | 示例 |
|---|---|---|
| `quote` | 加引号 | `{% raw %}{{ .Values.name | quote }}{% endraw %}` |
| `upper` / `lower` | 大小写转换 | `{% raw %}{{ .Values.env | upper }}{% endraw %}` |
| `default` | 默认值 | `{% raw %}{{ .Values.port | default 8080 }}{% endraw %}` |
| `indent` / `nindent` | 缩进 | `{% raw %}{{ toYaml . | nindent 2 }}{% endraw %}` |
| `toYaml` / `toJson` | 格式转换 | `{% raw %}{{ .Values.config | toYaml }}{% endraw %}` |
| `required` | 必填校验 | `{% raw %}{{ required "image is required" .Values.image }}{% endraw %}` |

---

## 三、实操篇：手写 nginx Chart

学完基础，现在**从零写一个 nginx Chart**，不依赖 `helm create`。

### 3.1 创建 Chart 目录结构

``` bash
mkdir -p nginx-chart/templates
cd nginx-chart
```

### 3.2 Chart.yaml

``` yaml
apiVersion: v2
name: nginx-chart
description: A simple nginx chart for learning Helm
type: application
version: 0.1.0
appVersion: "1.25"
```

### 3.3 values.yaml

``` yaml
replicaCount: 1

image:
  registry: swr.cn-north-4.myhuaweicloud.com
  repository: ddn-k8s/docker.io/nginx
  tag: "stable"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  host: nginx.local

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

# 自定义 nginx 配置
nginxConf: |
  server {
      listen 80;
      server_name localhost;
      location / {
          root /usr/share/nginx/html;
          index index.html;
      }
      location /health {
          return 200 'OK';
      }
  }
```

> **设计思路**：这里故意加了一个 `nginxConf` 参数，后面会用它演示 ConfigMap 的用法。

### 3.4 _helpers.tpl

``` tpl
{% raw %}{{- define "nginx-chart.name" -}}
{{- .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "nginx-chart.labels" -}}
app.kubernetes.io/name: {{ include "nginx-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
{{- end }}

{{- define "nginx-chart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "nginx-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}{% endraw %}
```

### 3.5 templates/configmap.yaml

先创建 ConfigMap 来存储 nginx 配置：

``` yaml
{% raw %}apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "nginx-chart.name" . }}-config
  labels:
    {{- include "nginx-chart.labels" . | nindent 4 }}
data:
  default.conf: |
    {{- .Values.nginxConf | nindent 4 }}{% endraw %}
```

`nindent 4` 让整个 nginxConf 内容相对于 `default.conf: |` 缩进 4 个空格。

### 3.6 templates/deployment.yaml

``` yaml
{% raw %}apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "nginx-chart.name" . }}
  labels:
    {{- include "nginx-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "nginx-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "nginx-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: nginx
          image: "{{ .Values.image.registry }}/{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d
      volumes:
        - name: nginx-config
          configMap:
            name: {{ include "nginx-chart.name" . }}-config{% endraw %}
```

> **关键点**：注意到 Deployment 通过 ConfigMap Volume 挂载了 nginx 配置。values.yaml 中的 `nginxConf` 最终注入到了 Pod 的 `/etc/nginx/conf.d/default.conf`。

### 3.7 templates/service.yaml

``` yaml
{% raw %}apiVersion: v1
kind: Service
metadata:
  name: {{ include "nginx-chart.name" . }}
  labels:
    {{- include "nginx-chart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
  selector:
    {{- include "nginx-chart.selectorLabels" . | nindent 4 }}{% endraw %}
```

### 3.8 templates/ingress.yaml（可选）

``` yaml
{% raw %}{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "nginx-chart.name" . }}
  labels:
    {{- include "nginx-chart.labels" . | nindent 4 }}
spec:
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "nginx-chart.name" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}{% endraw %}
```

### 3.9 安装并验证

``` bash
# 回到 nginx-chart 上级目录，安装
helm install nginx-demo ./nginx-chart

# 查看生成的资源
helm get manifest nginx-demo | head -40

# 查看 values
helm get values nginx-demo

# 端口转发，测试 /health 端点
kubectl port-forward svc/nginx-chart 8080:80
curl http://localhost:8080/health
# 输出：OK

# 测试修改配置后升级（改副本数）
helm upgrade nginx-demo ./nginx-chart --set replicaCount=3
kubectl get pods -l app.kubernetes.io/instance=nginx-demo
# 应看到 3 个 Pod
```

---

## 四、多环境篇：dev 与 prod 环境区分

同一套 Chart，不同环境用不同的 values。这是 Helm 在生产中最常用的模式。

### 4.1 创建环境 values 文件

```
nginx-chart/
├── values.yaml          # 公共默认值
├── values-dev.yaml      # 开发环境覆盖
├── values-prod.yaml     # 生产环境覆盖
└── ...
```

### 4.2 values-dev.yaml

``` yaml
replicaCount: 1

image:
  tag: "stable"

service:
  type: NodePort

ingress:
  enabled: true
  host: nginx-dev.local

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

nginxConf: |
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

### 4.3 values-prod.yaml

``` yaml
replicaCount: 3

image:
  tag: "stable"

service:
  type: ClusterIP

ingress:
  enabled: true
  host: nginx.liuxu.site

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 200m
    memory: 256Mi

nginxConf: |
  server {
      listen 80;
      server_name localhost;
      location / {
          root /usr/share/nginx/html;
          index index.html;
          add_header X-Environment "PRODUCTION";
      }
  }
```

### 4.4 安装到不同命名空间

``` bash
# 创建命名空间
kubectl create ns dev
kubectl create ns prod

# 安装到 dev
helm install nginx-demo ./nginx-chart \
  -f values-dev.yaml \
  -n dev

# 安装到 prod（用不同的 Release 名）
helm install nginx-prod ./nginx-chart \
  -f values-prod.yaml \
  -n prod

# 验证：dev 环境
kubectl -n dev get pods
kubectl -n dev port-forward svc/nginx-chart 8081:80 &
curl -I http://localhost:8081 2>&1 | grep X-Environment
# X-Environment: DEV

# 验证：prod 环境
kubectl -n prod get pods
kubectl -n prod port-forward svc/nginx-chart 8082:80 &
curl -I http://localhost:8082 2>&1 | grep X-Environment
# X-Environment: PRODUCTION
```

> **values 合并规则**：Helm 按顺序合并 — 后面的覆盖前面的。默认 `values.yaml` → `-f` 指定的文件 → `--set` 命令行。更深层的合并是 merge 而非 replace，所以 `resources` 中只改了一部分值也是安全的。

---

## 五、进阶篇：helm-diff 发布前预览

升级前想知道**到底变了什么**？`helm-diff` 插件解决这个问题。

### 5.1 安装插件

``` bash
helm plugin install https://github.com/databus23/helm-diff
```

### 5.2 预览变更

``` bash
# 假设我们改了 dev 环境的副本数
# 将 values-dev.yaml 中 replicaCount 改为 2，然后：

helm diff upgrade nginx-demo ./nginx-chart \
  -f values-dev.yaml \
  -n dev
```

输出类似：

``` diff
default, nginx-chart, Deployment (apps) has changed:
  # Source: nginx-chart/templates/deployment.yaml
...
-   replicas: 1
+   replicas: 2
...
```

用颜色高亮增删改的行，一目了然。

### 5.3 常用命令

``` bash
# 对比已安装的 Release 与新 values
helm diff upgrade <release> <chart> -f values.yaml

# 对比两个 Release
helm diff release <release-a> <release-b>

# 对比两个 revision
helm diff revision <release> <rev1> <rev2>
```

> **最佳实践**：CI/CD 流水线中在 `helm upgrade` 前先跑 `helm diff`，输出 diff 到 PR comment，审核通过后再执行真正的升级。

---

## 六、安全篇：helm-secrets + sops 加密敏感配置

生产环境必然涉及密码、Token、证书等敏感信息。**绝不能把明文密钥提交到 Git！**

### 6.1 安装工具

``` bash
# 安装 sops（加密工具，支持 AWS KMS / GCP KMS / Azure Key Vault / PGP）
# macOS
brew install sops

# Linux
curl -LO https://github.com/getsops/sops/releases/download/v3.10.2/sops-v3.10.2.linux.amd64
sudo mv sops-v3.10.2.linux.amd64 /usr/local/bin/sops
sudo chmod +x /usr/local/bin/sops

# 安装 helm-secrets 插件
helm plugin install https://github.com/jkroepke/helm-secrets
```

### 6.2 生成 PGP 密钥（演示用）

``` bash
# 生成密钥对
gpg --batch --gen-key <<EOF
Key-Type: RSA
Key-Length: 4096
Name-Real: Helm Secrets Demo
Name-Email: helm@liuxu.site
Expire-Date: 1y
%no-protection
%commit
EOF

# 获取 Key ID
gpg --list-secret-keys --keyid-format LONG | grep sec
# sec   rsa4096/XXXXXXXX 2026-07-13 [SC] [expires: 2027-07-13]

# 记下这个 ID，后面用
export SOPS_PGP_FP="XXXXXXXX"
```

> **生产环境建议使用云 KMS**（如 AWS KMS / GCP KMS），而不是本地 PGP 密钥。

### 6.3 创建加密的 secrets 文件

先准备一个包含敏感值的文件，**不要直接写入 values.yaml**：

``` yaml
# secrets-dev.yaml（加密前的原始内容）
db:
  password: "MyS3cret!Dev"
  apiKey: "sk-dev-abc123"

redis:
  password: "RedisP@ss"
```

用 sops 加密：

``` bash
# 加密文件
sops --encrypt \
  --pgp $SOPS_PGP_FP \
  --encrypted-regex '^(password|apiKey|secret)$' \
  secrets-dev.yaml > secrets-dev.yaml.enc

# 查看加密后的内容
cat secrets-dev.yaml.enc
```

加密后的内容类似：

``` yaml
db:
    password: ENC[AES256_GCM,data:xxxx,iv:xxxx,tag:xxxx,type:str]
    apiKey: ENC[AES256_GCM,data:xxxx,iv:xxxx,tag:xxxx,type:str]
redis:
    password: ENC[AES256_GCM,data:xxxx,iv:xxxx,tag:xxxx,type:str]
sops:
    pgp:
        -   created_at: "2026-07-13T13:00:00Z"
            enc: |-
                -----BEGIN PGP MESSAGE-----
                ...
                -----END PGP MESSAGE-----
    ...
```

> **`.gitignore` 中保留原始明文文件**，只把 `.enc` 加密文件提交 Git：
> ```
> # .gitignore
> secrets-*.yaml
> !secrets-*.yaml.enc
> ```

### 6.4 安装时使用加密文件

``` bash
# helm-secrets 自动识别 .enc 后缀，安装时解密
helm secrets install nginx-demo ./nginx-chart \
  -f values-dev.yaml \
  -f secrets-dev.yaml.enc \
  -n dev

# 升级同理
helm secrets upgrade nginx-demo ./nginx-chart \
  -f values-dev.yaml \
  -f secrets-dev.yaml.enc \
  -n dev

# 配合 helm-diff 预览
helm secrets diff upgrade nginx-demo ./nginx-chart \
  -f values-dev.yaml \
  -f secrets-dev.yaml.enc \
  -n dev
```

### 6.5 在模板中使用敏感值

修改 `templates/deployment.yaml`，添加环境变量：

``` yaml
{% raw %}        env:
        {{- if .Values.db }}
        - name: DB_PASSWORD
          value: {{ .Values.db.password | quote }}
        {{- end }}
        {{- if .Values.redis }}
        - name: REDIS_PASSWORD
          value: {{ .Values.redis.password | quote }}
        {{- end }}{% endraw %}
```

> 更推荐使用 **Secret 资源 + envFrom** 的方式注入敏感值，但从 helm-secrets 解密后通过 values 注入已能满足多数场景。

---

## 七、私有仓库篇：Harbor Chart 仓库

团队协作需要私有 Chart 仓库。Harbor 提供了 OCI 兼容的 Helm Chart 存储。

### 7.1 配置 Harbor（假设已有 Harbor 实例）

登录 Harbor（示例地址 `harbor.liuxu.site`）：

``` bash
helm registry login harbor.liuxu.site \
  --username admin \
  --password-stdin
```

### 7.2 打包 Chart

``` bash
# 回到 nginx-chart 上级目录
helm package ./nginx-chart
# 生成 nginx-chart-0.1.0.tgz
```

### 7.3 推送到 Harbor

``` bash
# 推送 OCI 格式的 Chart（Helm 3.8+ 支持）
helm push nginx-chart-0.1.0.tgz oci://harbor.liuxu.site/helm-charts

# 或者用传统 ChartMuseum 方式：
# helm repo add myrepo https://harbor.liuxu.site/chartrepo/helm-charts
# helm cm-push nginx-chart-0.1.0.tgz myrepo
```

### 7.4 拉取并使用

``` bash
# 从 OCI 仓库拉取
helm pull oci://harbor.liuxu.site/helm-charts/nginx-chart \
  --version 0.1.0

# 直接安装
helm install my-nginx oci://harbor.liuxu.site/helm-charts/nginx-chart \
  --version 0.1.0 \
  -n prod

# 或者添加为 repo 后安装
helm repo add harbor https://harbor.liuxu.site/chartrepo/helm-charts
helm repo update
helm install my-nginx harbor/nginx-chart -n prod
```

### 7.5 CI/CD 流水线集成

典型 GitLab CI 片段：

``` yaml
# .gitlab-ci.yml
helm-push:
  stage: deploy
  image: alpine/helm:3.17
  script:
    - helm registry login $HARBOR_URL -u $HARBOR_USER -p $HARBOR_PASS
    - helm package ./nginx-chart
    - helm push nginx-chart-*.tgz oci://$HARBOR_URL/helm-charts
  only:
    - tags  # 打 tag 时发布新版本
```

---

## 八、总结：从入门到生产的学习路径

回头看，你已完成了这样一条路径：

```
helm install/uninstall/upgrade/list  ← 基本命令
       ↓
Chart 结构 + 内置对象 + 流程控制     ← 理解模板
       ↓
手写 nginx Chart                    ← 从零构建
       ↓
多 values 文件 → dev/prod 环境       ← 环境管理
       ↓
helm-diff → 发布前预览变更           ← 安全升级
       ↓
helm-secrets + sops → 加密敏感配置   ← 生产安全
       ↓
Harbor 私有仓库 → 团队共享分发       ← 协作交付
```

这套能力覆盖了 Helm 从开发到生产的核心场景，足以应对日常工作中 90% 的需求。更深入的用法（如 Library Chart、Helm Hooks、Helm Test）可以在实践中逐步探索。

---

## 常用命令速查

| 场景 | 命令 |
|---|---|
| 添加仓库 | `helm repo add <name> <url>` |
| 搜索 Chart | `helm search repo <keyword>` |
| 安装 | `helm install <name> <chart> -n <ns> -f values.yaml` |
| 升级 | `helm upgrade <name> <chart> -f values.yaml` |
| 回滚 | `helm rollback <name> <rev>` |
| 查看历史 | `helm history <name>` |
| 查看 values | `helm get values <name>` |
| 查看渲染结果 | `helm template <name> <chart> -f values.yaml` |
| 打包 | `helm package <chart-dir>` |
| 推送 OCI | `helm push <file.tgz> oci://<registry>/<path>` |
| Diff 预览 | `helm diff upgrade <name> <chart> -f values.yaml` |
| Secrets 安装 | `helm secrets install <name> <chart> -f secrets.yaml.enc` |

---

希望这篇教程能帮你扎实地掌握 Helm，从"会用"到"能写"，再到"敢上生产"。
