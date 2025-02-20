---
layout: post
title: "[庖丁解牛]一文详解Kubernetes Deployment"
date: 2025-02-19
tags: [kubernetes, 庖丁解牛]
comments: false
author: 小辣条
toc : true
---
Kubernetes 的 Deployment 是一种用于定义和管理 Pod 副本的资源对象。它支持声明式更新、滚动升级、回滚等功能。以下是 Deployment 的所有字段及其详细说明和示例。

<!-- more -->

---

### **Deployment 字段详细说明**

#### **1. 基础字段**
- **apiVersion**: API 版本，通常为 `apps/v1`。
- **kind**: 资源类型，固定为 `Deployment`。
- **metadata**: 元数据，包括名称、命名空间、标签等。
  - **name**: Deployment 的名称。
  - **namespace**: 命名空间（可选）。
  - **labels**: 标签（可选）。
  - **annotations**: 注解（可选）。

#### **2. spec**: Deployment 的期望状态。
- **replicas**: Pod 副本数，默认为 1。
- **selector**: 标签选择器，用于匹配 Pod。
  - **matchLabels**: 精确匹配的标签键值对。
  - **matchExpressions**: 基于表达式的匹配规则（可选）。
- **template**: Pod 模板，定义 Pod 的配置。
  - **metadata**: Pod 的元数据（如标签、注解）。
  - **spec**: Pod 的期望状态（如容器、卷等）。

#### **3. spec.template.spec**: Pod 的详细配置。
- **containers**: 容器列表。
  - **name**: 容器名称。
  - **image**: 容器镜像。
  - **ports**: 容器暴露的端口列表。
    - **containerPort**: 容器监听的端口。
  - **env**: 环境变量列表。
    - **name**: 环境变量名称。
    - **value**: 环境变量值。
  - **resources**: 资源限制和请求。
    - **limits**: 资源上限（如 CPU、内存）。
    - **requests**: 资源请求（如 CPU、内存）。
  - **volumeMounts**: 挂载的卷列表。
    - **name**: 卷名称。
    - **mountPath**: 挂载路径。
  - **livenessProbe**: 存活探针，检测容器是否存活。
    - **httpGet**: 通过 HTTP GET 请求检测。
      - **path**: 请求路径。
      - **port**: 请求端口。
    - **tcpSocket**: 通过 TCP 端口检测。
      - **port**: 检测的端口。
    - **exec**: 通过执行命令检测。
      - **command**: 执行的命令。
    - **initialDelaySeconds**: 容器启动后等待的时间（秒）。
    - **periodSeconds**: 检测间隔时间（秒）。
    - **timeoutSeconds**: 检测超时时间（秒）。
    - **successThreshold**: 检测成功的最小连续成功次数。
    - **failureThreshold**: 检测失败的最小连续失败次数。
  - **readinessProbe**: 就绪探针，检测容器是否准备好接收流量。
    - 字段与 `livenessProbe` 相同。
  - **startupProbe**: 启动探针，检测容器是否启动完成。
    - 字段与 `livenessProbe` 相同。
  - **securityContext**: 容器的安全上下文。
    - **runAsUser**: 运行容器的用户 ID。
    - **runAsGroup**: 运行容器的用户组 ID。
    - **privileged**: 是否以特权模式运行。
    - **capabilities**: 添加或删除 Linux 能力。
    - **readOnlyRootFilesystem**: 是否以只读模式挂载根文件系统。
- **volumes**: 卷列表。
  - **name**: 卷名称。
  - **configMap**: 使用 ConfigMap 作为卷。
  - **secret**: 使用 Secret 作为卷。
  - **emptyDir**: 临时目录卷。
  - **hostPath**: 挂载主机路径。
- **restartPolicy**: 容器重启策略，默认为 `Always`。
- **nodeSelector**: 节点选择器，指定 Pod 调度到特定节点。
- **affinity**: 亲和性配置。
  - **nodeAffinity**: 节点亲和性。
  - **podAffinity**: Pod 亲和性。
  - **podAntiAffinity**: Pod 反亲和性。
- **tolerations**: 容忍配置，允许 Pod 调度到带有污点的节点。
- **securityContext**: Pod 的安全上下文。
  - **runAsUser**: 运行 Pod 的用户 ID。
  - **runAsGroup**: 运行 Pod 的用户组 ID。
  - **fsGroup**: 文件系统组 ID。
  - **seccompProfile**: Seccomp 配置文件。

#### **4. spec.strategy**: 更新策略。
- **type**: 更新类型，默认为 `RollingUpdate`。
  - **RollingUpdate**: 滚动更新。
    - **maxUnavailable**: 更新期间允许的最大不可用 Pod 数（百分比或绝对值）。
    - **maxSurge**: 更新期间允许的最大额外 Pod 数（百分比或绝对值）。
  - **Recreate**: 先删除旧 Pod，再创建新 Pod。

#### **5. spec.minReadySeconds**: Pod 就绪后等待的时间（秒），默认为 0。
#### **6. spec.revisionHistoryLimit**: 保留的历史版本数，默认为 10。
#### **7. spec.paused**: 暂停 Deployment 的更新，默认为 false。

---

### **完整示例**

以下是一个包含健康检测、资源限制、调度策略等的完整 `Deployment` 示例：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  minReadySeconds: 5
  revisionHistoryLimit: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.19.10
        ports:
        - containerPort: 80
        env:
        - name: ENV_NAME
          value: "production"
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 2
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 2
          successThreshold: 1
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /startup
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 2
          successThreshold: 1
          failureThreshold: 30
        securityContext:
          runAsUser: 1000
          runAsGroup: 3000
          readOnlyRootFilesystem: true
      volumes:
      - name: config-volume
        configMap:
          name: nginx-config
      nodeSelector:
        disktype: ssd
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx
            topologyKey: kubernetes.io/hostname
      tolerations:
      - key: "key1"
        operator: "Equal"
        value: "value1"
        effect: "NoSchedule"
      securityContext:
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
```

---

### **关键字段详解**

1. **健康检测**
   - **livenessProbe**: 检测容器是否存活，失败会重启容器。
   - **readinessProbe**: 检测容器是否准备好接收流量，失败会从 Service 中移除 Pod。
   - **startupProbe**: 检测容器是否启动完成，适用于启动较慢的应用。

2. **资源限制**
   - **resources**: 设置 CPU 和内存的请求和上限。

3. **调度策略**
   - **nodeSelector**: 将 Pod 调度到带有 `disktype=ssd` 标签的节点。
   - **affinity**: 使用 Pod 反亲和性，避免同一应用的 Pod 调度到同一节点。
   - **tolerations**: 允许 Pod 调度到带有 `key1=value1` 污点的节点。

4. **安全上下文**
   - **securityContext**: 设置容器和 Pod 的运行用户、用户组和文件系统权限。

---

通过以上字段和示例，可以更全面地定义和管理 Kubernetes 中的 `Deployment` 资源。希服对你有所帮助！
