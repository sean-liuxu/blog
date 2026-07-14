---
title: MinIO 完整掌握 — 从零搭建对象存储
date: 2026-07-14 15:00:00
tags:
  - MinIO
categories:
  - 云原生
---

## 前言

MinIO 是一款开源的高性能对象存储，**完全兼容 AWS S3 API**。你可以把它理解为一个轻量级的"自建 S3"——不用绑定云厂商，一个二进制文件就能跑起来，特别适合私有化部署、开发测试环境和边缘场景。

本文从单机部署开始，逐步深入到分布式集群、安全配置、生命周期管理，最后用 Python SDK 编程操作。每个环节前后关联，建议跟着敲。

> 前置条件：一台 Linux 机器（虚拟机即可），至少有 1GB 空闲磁盘。

---

## 一、基础篇：单机部署

### 1.1 二进制安装

``` bash
# 下载 MinIO 二进制
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/

# 验证
minio --version
# minio version RELEASE.2026-07-10T...
```

### 1.2 启动第一个 MinIO 服务

``` bash
# 创建数据目录
mkdir -p ~/minio-data

# 直接启动（前台运行）
minio server ~/minio-data
```

启动后看输出：

```
API: http://192.168.1.100:9000
RootUser: minioadmin
RootPass: minioadmin
Console: http://192.168.1.100:35541
```

- **API 端口 9000**：S3 协议接口，客户端和 SDK 连这个
- **Console 端口（随机）**：Web 管理界面
- **默认账号密码**：`minioadmin / minioadmin`

### 1.3 后台运行 + 自定义端口

``` bash
# 自定义账号密码和数据目录
export MINIO_ROOT_USER=admin
export MINIO_ROOT_PASSWORD=MyP@ssw0rd

mkdir -p ~/minio-data

# 后台运行，固定 Console 端口
nohup minio server ~/minio-data \
  --address ":9000" \
  --console-address ":9001" \
  > ~/minio.log 2>&1 &

# 确认启动
tail -f ~/minio.log
```

浏览器打开 `http://<IP>:9001`，用 `admin / MyP@ssw0rd` 登录，就能看到 Web 管理界面。

> **地址说明**：`--address` 是 S3 API 端口，客户端用；`--console-address` 是 Web 管理界面。

---

## 二、操作篇：Bucket 和对象管理

先安装命令行客户端 `mc`（MinIO Client）：

``` bash
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/
```

### 2.1 添加 MinIO 别名

``` bash
# 把 MinIO 服务注册为一个别名，后面操作就不用每次写地址和密钥
mc alias set myminio http://127.0.0.1:9000 admin MyP@ssw0rd

# 查看已注册的别名
mc alias list
# myminio
#   URL       : http://127.0.0.1:9000
#   AccessKey : admin
```

### 2.2 创建 Bucket 并上传文件

``` bash
# 创建一个测试文件
echo "Hello MinIO!" > hello.txt

# 创建 Bucket
mc mb myminio/my-bucket

# 上传文件
mc cp hello.txt myminio/my-bucket/

# 查看 Bucket 内容
mc ls myminio/my-bucket/
# [2026-07-14 15:00:00 CST]    13B STANDARD hello.txt

# 查看文件内容
mc cat myminio/my-bucket/hello.txt
# Hello MinIO!
```

### 2.3 下载和删除

``` bash
# 下载文件
mc cp myminio/my-bucket/hello.txt ./hello-downloaded.txt

# 删除文件
mc rm myminio/my-bucket/hello.txt

# 删除 Bucket（Bucket 必须为空）
mc rb myminio/my-bucket
```

### 2.4 常用 mc 命令速查

| 命令 | 说明 |
|---|---|
| `mc mb <alias>/<bucket>` | 创建 Bucket |
| `mc rb <alias>/<bucket>` | 删除空 Bucket |
| `mc ls <alias>/<bucket>/` | 列出对象 |
| `mc cp <src> <dst>` | 上传/下载/拷贝 |
| `mc mv <src> <dst>` | 移动/重命名 |
| `mc rm <alias>/<bucket>/<obj>` | 删除对象 |
| `mc cat <alias>/<bucket>/<obj>` | 查看文件内容 |
| `mc stat <alias>/<bucket>/<obj>` | 查看文件元信息 |
| `mc mirror <src> <dst>` | 镜像同步目录 |

---

## 三、安全篇：TLS 加密 + 访问策略

### 3.1 生成自签名 TLS 证书

MinIO 支持 HTTPS，先搞一个自签名证书用于测试：

``` bash
mkdir -p ~/certs
cd ~/certs

# 生成私钥
openssl genrsa -out private.key 2048

# 生成带IP SAN自签证书，包含127.0.0.1
openssl req -new -x509 -days 365 -key private.key \
  -out public.crt \
  -subj "/C=CN/ST=Beijing/L=Beijing/O=MinIO/CN=*.minio.local" \
  -addext "subjectAltName=IP:127.0.0.1,DNS:localhost,DNS:*.minio.local"

# 查看生成的证书
ls -la
# private.key  public.crt
```

### 3.2 MinIO 自动识别证书

MinIO 有约定优于配置的机制：把证书放到 `~/.minio/certs/` 下，启动时自动启用 HTTPS。

``` bash
mkdir -p ~/.minio/certs
cp ~/certs/public.crt ~/.minio/certs/
cp ~/certs/private.key ~/.minio/certs/

# 重启 MinIO（先停掉旧进程）
pkill minio
nohup minio server ~/minio-data \
  --address ":9000" \
  --console-address ":9001" \
  > ~/minio.log 2>&1 &

# mc 连接 HTTPS 需要加 --insecure（自签名证书）
mc alias set myminio-https https://127.0.0.1:9000 admin MyP@ssw0rd --insecure
```

> **生产环境**：用 Let's Encrypt 或公司 CA 签发的证书，放在 `~/.minio/certs/` 下，MinIO 会自动续期。

### 3.3 创建只读用户和访问策略

生产环境不能所有人都用 root 账号。我们来创建一个**只读用户**，只能下载 `my-bucket` 的文件。

``` bash
# 1. 创建 Bucket
mc mb myminio/my-bucket
mc cp hello.txt myminio/my-bucket/

# 2. 定义一个只读策略
cat > /tmp/readonly-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
EOF

# 3. 在 MinIO 中创建策略
mc admin policy create myminio readonly /tmp/readonly-policy.json

# 4. 创建一个用户
mc admin user add myminio reader ReadOnlyP@ss

# 5. 把策略绑定到用户
mc admin policy attach myminio readonly --user reader

# 6. 验证：用 reader 身份连接
mc alias set myreader http://127.0.0.1:9000 reader ReadOnlyP@ss

# 可以读
mc cat myreader/my-bucket/hello.txt
# Hello MinIO!

# 无法写（预期报错）
mc cp hello.txt myreader/my-bucket/hello2.txt
# 报错：Access Denied
```

| 组件 | 说明 |
|---|---|
| `mc admin user add` | 创建用户（AccessKey / SecretKey） |
| `mc admin policy create` | 上传策略 JSON |
| `mc admin policy attach` | 绑定策略到用户 |

---

## 四、高级特性篇：版本控制与生命周期

### 4.1 开启版本控制

版本控制让你能恢复被覆盖或删除的文件。类比 Git，每次修改都留一个历史版本。

``` bash
# 开启版本控制（需要 Bucket 为空或刚创建）
mc version enable myminio/my-bucket

# 上传 v1
echo "Version 1" > test.txt
mc cp test.txt myminio/my-bucket/

# 覆盖上传 v2
echo "Version 2" > test.txt
mc cp test.txt myminio/my-bucket/

# 查看所有版本
mc ls --versions myminio/my-bucket/test.txt
# 输出两个版本，带 version ID

# 下载指定版本
mc cp --version-id "<VersionID>" myminio/my-bucket/test.txt ./old.txt
cat old.txt
# Version 1
```

### 4.2 删除标记 vs 永久删除

``` bash
# 普通删除：创建 delete marker，可恢复
mc rm myminio/my-bucket/test.txt

# 列出包括已删除的版本
mc ls --versions myminio/my-bucket/test.txt
# 看到 DELETE 标记

# 永久删除指定版本
mc rm --version-id "<VersionID>" myminio/my-bucket/test.txt
```

### 4.3 生命周期规则 — 自动清理旧版本

版本控制开久了，历史版本会越来越多。用**生命周期规则**自动清理：

``` bash
# 定义生命周期：7 天后自动删除旧版本
cat > /tmp/lifecycle.json <<'EOF'
{
  "Rules": [
    {
      "ID": "cleanup-old-versions",
      "Status": "Enabled",
      "Filter": { "Prefix": "" },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 7
      },
      "Expiration": {
        "Days": 90,
        "ExpiredObjectDeleteMarker": true
      }
    }
  ]
}
EOF

# 导入到 Bucket
mc ilm import myminio/my-bucket < /tmp/lifecycle.json

# 查看已生效的规则
mc ilm ls myminio/my-bucket
```

| 参数 | 含义 |
|---|---|
| `NoncurrentVersionExpiration` | 非当前版本（旧版本）保留天数 |
| `Expiration` | 当前版本过期天数 |
| `ExpiredObjectDeleteMarker` | 自动清理无用的 delete marker |

---

## 五、分布式部署篇

单机适合开发和测试，生产环境需要**分布式 + 纠删码**保证高可用。

### 5.1 分布式部署原理

MinIO 分布式集群最少 **4 个节点**，每个节点**至少 1 块磁盘**。数据通过纠删码（Erasure Code）分片存储，容忍最多 **一半节点故障**（4 节点中 2 台挂了数据仍可读）。

**启动命令：**

``` bash
# 所有节点执行相同命令，只需改 IP 和挂载路径
minio server \
  http://minio-{1...4}.local:9000/data
```

### 5.2 用 4 个目录模拟分布式集群

单机也能模拟，用 4 个目录代表 4 个节点的磁盘：

``` bash
# 创建 4 个数据目录
mkdir -p ~/minio-cluster/{disk1,disk2,disk3,disk4}

# 先停掉旧进程
pkill minio

# 启动分布式模式
nohup minio server \
  http://127.0.0.1:9000/~/minio-cluster/disk1 \
  http://127.0.0.1:9001/~/minio-cluster/disk2 \
  http://127.0.0.2:9002/~/minio-cluster/disk3 \
  http://127.0.0.2:9003/~/minio-cluster/disk4 \
  --console-address ":19001" \
  > ~/minio-cluster.log 2>&1 &
```

> **注意**：真实分布式环境各节点 IP 不同、时间同步（NTP）、磁盘格式为 XFS。这里用本地多目录只是学习分布式概念。

### 5.3 验证集群状态

``` bash
mc admin info myminio
# 显示 4 个节点，每节点 1 块盘，Total Capacity = 各盘之和

# 查看各节点状态
mc admin info myminio --json | jq '.info.servers[] | {endpoint: .endpoint, state: .state, drives: [.drives[].state]}'
```

---

## 六、编程篇：Python SDK 操作 MinIO

对象存储最终是要在代码里用的。以 Python 为例：

### 6.1 安装 SDK

``` bash
pip install minio
```

### 6.2 基础操作：上传、下载、列表

``` python
from minio import Minio
from minio.error import S3Error

# 连接
client = Minio(
    "127.0.0.1:9000",
    access_key="admin",
    secret_key="MyP@ssw0rd",
    secure=False  # HTTP；HTTPS 设为 True
)

bucket_name = "my-bucket"

# 确保 Bucket 存在
if not client.bucket_exists(bucket_name):
    client.make_bucket(bucket_name)
    print(f"Bucket '{bucket_name}' 已创建")

# 上传文件
client.fput_object(
    bucket_name, "hello.txt", "/tmp/hello.txt"
)
print("上传成功")

# 下载文件
client.fget_object(
    bucket_name, "hello.txt", "/tmp/downloaded.txt"
)
print("下载成功")

# 列出 Bucket 中的对象
objects = client.list_objects(bucket_name)
for obj in objects:
    print(f"  {obj.object_name}  {obj.size} bytes  {obj.last_modified}")
```

### 6.3 预签名 URL — 临时分享文件

不想暴露访问密钥，但需要让别人临时下载文件？用**预签名 URL**：

``` python
# 生成一个 7 天有效的下载链接
url = client.presigned_get_object(
    bucket_name, "hello.txt",
    expires=timedelta(days=7)
)
print(f"下载链接（7天内有效）：\n{url}")
```

这个 URL 包含签名参数，任何人拿到都能直接下载，过期自动失效。分享大文件、日志导出等场景非常方便。

### 6.4 预签名上传 — 让客户端直传

传统文件上传流程：客户端 → 应用服务器 → MinIO。应用服务器成了瓶颈。

用预签名上传 URL，客户端直接上传到 MinIO，绕开应用服务器：

``` python
# 应用端生成上传链接
url = client.presigned_put_object(
    bucket_name, "uploads/avatar.png",
    expires=timedelta(minutes=10)
)
print(f"上传链接（10分钟内有效）：\n{url}")

# 客户端拿到 URL 后：
# curl -X PUT -T avatar.png "<预签名URL>"
```

```
传统流程：  客户端 ──→ 服务器 ──→ MinIO  （服务器过流）
预签名流程： 客户端 ──→ MinIO             （直传，带宽无限）
```

---

## 七、总结

从单机到集群、从命令行到代码调用，你已经覆盖了 MinIO 的核心能力：

```
单机部署 ──→ mc 客户端操作 ──→ TLS + 权限策略
     ↓
版本控制 + 生命周期  ──→  分布式集群
     ↓
Python SDK ──→ 预签名 URL（上传 / 下载）
```

| 场景 | 推荐配置 |
|---|---|
| 本地开发 | 单机 + HTTP，`minio server /data` 一行搞定 |
| 团队测试 | 单机 + HTTPS + 只读/读写用户分离 |
| 生产环境 | 4 节点分布式 + Nginx 反向代理 + Prometheus 监控 |

MinIO 的内核极其精简但功能完备，多花点时间把安全策略和生命周期配置做好，它就能成为一个稳定可靠的生产级对象存储。

---

## 常用命令速查

| 场景 | 命令 |
|---|---|
| 启动服务 | `minio server /data --console-address ":9001"` |
| 添加别名 | `mc alias set <name> <url> <ak> <sk>` |
| 创建 Bucket | `mc mb <alias>/<bucket>` |
| 上传 | `mc cp <file> <alias>/<bucket>/` |
| 下载 | `mc cp <alias>/<bucket>/<file> .` |
| 开启版本控制 | `mc version enable <alias>/<bucket>` |
| 查看历史版本 | `mc ls --versions <alias>/<bucket>/<file>` |
| 创建用户 | `mc admin user add <alias> <user> <password>` |
| 绑定策略 | `mc admin policy attach <alias> <policy> --user <user>` |
| 导入生命周期 | `mc ilm import <alias>/<bucket> < lifecycle.json` |
| Python 连接 | `Minio("host:9000", access_key, secret_key, secure=True)` |
| 生成下载链接 | `client.presigned_get_object(bucket, obj, expires=...)` |
| 生成上传链接 | `client.presigned_put_object(bucket, obj, expires=...)` |
