---
title: GPU虚拟化 - MIG
date: 2026-07-14 10:00:00
tags:
  - Kubernetes
  - gpu虚拟化
categories:
  - 云原生
---
**前提条件**：仅A30、A100、H100支持MIG 硬件切分。

# 一、硬件切分

## 1. 检查GPU驱动是否已安装成功；
```bash
nvidia-smi
```
![fce7778a9544626d989c75c942151602.png](/images/A100通过MIG虚拟化并进行K8S调用/fce7778a9544626d989c75c942151602.png)

## 2. 开启 MIG 模式 (需要重启 GPU 或节点)
```bash
nvidia-smi -mig 1
```
![2bb1911a19062102e86b5a8293ba4695.png](/images/A100通过MIG虚拟化并进行K8S调用/2bb1911a19062102e86b5a8293ba4695.png)
如图，如果GPU被某些进程占用，则执行会告警，必须找到对应进程结束停止占用。
![0eccf164a7554cb7b214fffa0790ff72.png](/images/A100通过MIG虚拟化并进行K8S调用/0eccf164a7554cb7b214fffa0790ff72.png)
出现此内容，代表开启成功，因为有两张A100, 输出两次。

关闭命令：
```bash
nvidia-smi -mig 0
```
## 3. 查看支持的 Profile ID
```bash
nvidia-smi mig -lgip
```
![52bda1b672b528c958db3ea442206d57.png](/images/A100通过MIG虚拟化并进行K8S调用/52bda1b672b528c958db3ea442206d57.png)

## 4. 创建 MIG 实例 
#### 假设 Profile ID 为 19
```bash
nvidia-smi mig -cgi 19  # 例如创建一个 1g.5gb 实例
nvidia-smi mig -cgi 19,19,19,19,19,19,19 -C # 例如创建七个 1g.5gb 实例
```
![242a9d94644a22387318e7f20facc57c.png](/images/A100通过MIG虚拟化并进行K8S调用/242a9d94644a22387318e7f20facc57c.png)

## 5. 确认实例已创建
```bash
nvidia-smi mig -lgi
```
![7449a46f5e83c1dcb0c48db0407ef9ad.png](/images/A100通过MIG虚拟化并进行K8S调用/7449a46f5e83c1dcb0c48db0407ef9ad.png)

# 二、安装gpu-device-plugin
## 1. GPU节点打标签
```bash
kubectl label node gpu-node01,gpu-node02 nvidia-device-enable=enable --overwrite
```
![2da5089cd76bd4d72e319dfa26cc5d75.png](/images/A100通过MIG虚拟化并进行K8S调用/2da5089cd76bd4d72e319dfa26cc5d75.png)
## 2. 安装GPU插件
**vim device-plugin.yaml**
```bash
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: nvidia-device-plugin-ds
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: nvidia-device-plugin-ds
    spec:
      # 关键：只调度带有 nvidia-device-enable=enable 标签的节点
      nodeSelector:
        nvidia-device-enable: "enable"
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      priorityClassName: "system-node-critical"
      containers:
      - image: nvcr.io/nvidia/k8s-device-plugin:v0.19.2
        name: nvidia-device-plugin-ctr
        command:
        - "/usr/bin/nvidia-device-plugin"
        - "--mig-strategy=single"
        env:
        - name: NVIDIA_MIG_MONITOR_DEVICES
          value: all
        - name: NVIDIA_MIG_CONFIG_DEVICES
          value: all
        - name: NVIDIA_MIG_STRATEGY
          value: single
        - name: NVIDIA_LOG_LEVEL
          value: info
        securityContext:
          privileged: true
          allowPrivilegeEscalation: true
        volumeMounts:
        - name: kubelet-device-plugins-dir
          mountPath: /var/lib/kubelet/device-plugins
        - name: host-dev
          mountPath: /dev
        - name: nvidia-mig
          mountPath: /var/lib/nvidia-mig
        - name: lib-modules
          mountPath: /lib/modules
          readOnly: true
        - name: usr-src
          mountPath: /usr/src
          readOnly: true
      volumes:
      - name: kubelet-device-plugins-dir
        hostPath:
          path: /var/lib/kubelet/device-plugins
          type: Directory
      - name: host-dev
        hostPath:
          path: /dev
      - name: nvidia-mig
        hostPath:
          path: /var/lib/nvidia-mig
          type: DirectoryOrCreate
      - name: lib-modules
        hostPath:
          path: /lib/modules
          type: Directory
      - name: usr-src
        hostPath:
          path: /usr/src
          type: Directory
```
		  
## 2. 执行安装
```bash
kubectl apply -f device-plugin.yaml
```

查看
```bash
kubectl get pod -A
```
![62ef2d1dae171690410922ddde8f563e.png](/images/A100通过MIG虚拟化并进行K8S调用/62ef2d1dae171690410922ddde8f563e.png)
## 3. 查看节点GPU信息
```bash
kubectl describe node 172.10.9.8
```
![4395ab604ed70b42ac0fe9280e73b0c6.png](/images/A100通过MIG虚拟化并进行K8S调用/4395ab604ed70b42ac0fe9280e73b0c6.png)
可以看到GPU节点信息正常。

# 三、（可选）不依赖平台测试容器引用GPU资源
## 1. 创建测试环境
**vim gpu-test.yaml**
```apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-test
  labels:
    app: gpu-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gpu-test
  template:
    metadata:
      labels:
        app: gpu-test
    spec:
      containers:
        - name: ubuntu-container
          image: swr.cn-north-4.myhuaweicloud.com/tc-deliver-lab-amd64/ai-envs-x86_64:1.3.4.21
          ports:
            - containerPort: 61000
          resources:
            requests:
              nvidia.com/gpu: "1"
            limits:
              nvidia.com/gpu: "1"
---
apiVersion: v1
kind: Service
metadata:
  name: gpu-service
spec:
  selector:
    app: gpu-test
  type: NodePort
  ports:
    - port: 61000
      targetPort: 61000
      nodePort: 31000
```

创建：
```bash
kubectl apply -f gpu-test.yaml
```
## 2. 访问测试
访问节点：http://<ip>:31000/?token=2ec7381f2d00c7cd7a0cf2b2163f86cd920d74906f3b7d21
token获取：
```bash
kubectl logs -f $(kubectl get pod | grep gpu-test | awk '{print $1}')
```
![1473bb3fbfd915ea5d884eb348088574.png](/images/A100通过MIG虚拟化并进行K8S调用/1473bb3fbfd915ea5d884eb348088574.png)

打开对应内核测试代码
```bash
import tensorflow as tf
tf.test.is_gpu_available()
```
![a74c9cd9d9a58da06c7c2403ce8e9e8b.png](/images/A100通过MIG虚拟化并进行K8S调用/a74c9cd9d9a58da06c7c2403ce8e9e8b.png)

输出**true**及切分后大小即可。

--- 结束