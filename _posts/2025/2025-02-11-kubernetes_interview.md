---
layout: post
title: "Kubernetes常见面试题(一)"
date:  2025-02-11
tags: [kubernetes, 面试]
comments: false
author: 小辣条
toc : true
---
收集了一些kubernetes常见的面试题，希望对大家有帮助
<!-- more -->

# 1、kubernetes有哪些组件以及它们的作用
Kubernetes 是一个复杂的容器编排系统，由多个核心组件协同工作。这些组件可以分为 控制平面组件（Control Plane Components） 和 工作节点组件（Node Components） 两部分。

控制平面组件
- kube-apiserver
- etcd
- kube-scheduler
- kube-controller-manager

工作节点组件
- kubelet
- kube-proxy
- 容器运行时（Container Runtime）

插件
- DNS 插件（如 CoreDNS）
- 网络插件（如 Calico、Flannel）
- Ingress 控制器（如 Nginx Ingress）
- 存储插件（如 CSI 插件）

组件交互关系

- 用户通过 kubectl 或 API 向 kube-apiserver 发送请求。
- kube-apiserver 将状态存储到 etcd。
- kube-controller-manager 和 kube-scheduler 监听 kube-apiserver，调整集群状态。
- kubelet 监听 kube-apiserver，在节点上启动和管理 Pod。
- kube-proxy 处理服务的网络流量。


# 2、kubernetes中Pod生命周期过程
Pod 的生命周期涉及多个阶段，从创建到终止，每个阶段包含不同的状态和事件。以下是 **详细的生命周期流程**，涵盖启动（包括初始化容器、主容器启动、钩子等）和停止（包括终止流程、钩子等）：

---

### **1. Pod 启动流程**
#### **阶段 1：Pod 初始化**
1. **Pending**  
   - Pod 被创建后首先进入 `Pending` 状态，等待被调度到合适的 Node。
   - 触发事件：调度器（Scheduler）选择目标 Node，分配资源。
   - 可能阻塞原因：资源不足、镜像拉取策略限制、Node 选择器不匹配等。

2. **Init Containers（初始化容器）**  
   - 如果定义了 `initContainers`，它们会按顺序执行：
     1. 每个 Init 容器必须成功退出（Exit Code 0），才会启动下一个。
     2. 如果某个 Init 容器失败，Pod 会根据 `restartPolicy` 重启（默认策略是 `Always`，但 Init 容器失败会导致 Pod 重启整个流程）。
   - **用途**：执行主容器启动前的准备工作（如数据库迁移、依赖检查）。

---

#### **阶段 2：主容器启动**
1. **镜像拉取（Image Pull）**  
   - 根据 `imagePullPolicy` 拉取镜像（`Always`、`IfNotPresent`、`Never`）。
   - 如果拉取失败，Pod 保持 `Pending` 状态，事件中会显示 `ErrImagePull` 或 `ImagePullBackOff`。

2. **容器创建（Container Create）**  
   - 容器运行时（如 Docker、containerd）创建容器环境，挂载卷（Volumes）、配置网络等。

3. **启动主容器**  
   - **执行命令**：
     - 如果定义了 `command`（即 `ENTRYPOINT`）和 `args`（即 `CMD`），容器会运行这些指令。
   - **PostStart Hook**（启动后钩子）：
     - 在容器启动后立即执行（与主进程并行）。
     - 可以是 `exec`（执行命令）或 `httpGet`（发送 HTTP 请求）。
     - **注意**：如果 PostStart 执行失败，容器会被杀死并根据 `restartPolicy` 重启。

4. **探针检测（Probes）**  
   - **Liveness Probe**：检测容器是否存活。失败会触发重启。
   - **Readiness Probe**：检测容器是否就绪。失败会从 Service 的 Endpoints 中移除 Pod。
   - **Startup Probe**（可选）：延迟其他探针，直到应用完成启动。

5. **Running 状态**  
   - 主容器成功启动并通过探针检测后，Pod 进入 `Running` 状态。

---

### **2. Pod 停止流程**
#### **阶段 1：触发终止**
1. **用户删除 Pod**（如 `kubectl delete pod`）或 **系统驱逐**（如资源不足）。
2. Pod 进入 `Terminating` 状态，并从 Service 的 Endpoints 列表中移除。

---

#### **阶段 2：优雅终止（Graceful Termination）**
1. **PreStop Hook**（终止前钩子）：
   - 在发送终止信号（SIGTERM）前执行。
   - 用于执行清理任务（如通知其他服务、保存状态）。
   - 如果钩子执行时间过长，可能会被强制终止。

2. **SIGTERM 信号**：
   - 容器主进程收到 SIGTERM，启动优雅关闭流程。
   - Kubernetes 默认等待 `terminationGracePeriodSeconds`（默认 30 秒）。

3. **强制终止（SIGKILL）**：
   - 如果容器在宽限期后仍未停止，发送 SIGKILL 强制终止。

4. **清理资源**：
   - 卸载卷、删除容器、释放网络资源。

---

### **3. Pod 状态总结**
| 状态          | 描述                                                                 |
|---------------|----------------------------------------------------------------------|
| `Pending`     | 等待调度或初始化容器运行中。                                         |
| `Running`     | 至少一个主容器在运行。                                               |
| `Succeeded`   | 所有容器成功退出（Exit Code 0），且不再重启。                        |
| `Failed`      | 至少一个容器异常退出（Exit Code ≠ 0），且不再重启。                  |
| `Terminating` | 正在停止过程中（执行 PreStop 钩子或等待进程退出）。                  |

---

### **4. 示例配置片段**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  initContainers:
    - name: init-service
      image: busybox
      command: ['sh', '-c', 'until nslookup myservice; do echo waiting; sleep 2; done']
  containers:
    - name: main-container
      image: nginx
      lifecycle:
        postStart:
          exec:
            command: ["/bin/sh", "-c", "echo 'Container started' > /tmp/start.log"]
        preStop:
          httpGet:
            path: /graceful-shutdown
            port: 80
      terminationGracePeriodSeconds: 60
  restartPolicy: OnFailure
```

---

### **5. 关键注意事项**
1. **PostStart 与 PreStop**：
   - PostStart 不保证在 `ENTRYPOINT` 之前执行，两者是并行的。
   - PreStop 是容器终止的唯一可靠方式，SIGTERM 可能被应用忽略。

2. **终止宽限期**：
   - 调整 `terminationGracePeriodSeconds` 以适应应用的关闭时间。

3. **Init 容器与 Sidecar**：
   - Init 容器适合前置任务，Sidecar 容器（如日志收集）与主容器并行运行。

通过理解这些细节，可以更好地设计健壮的容器化应用，处理启动依赖、优雅关闭等关键场景。


# 3、kubernetes中有哪些业务预热手段？
```
1、利用istio或者deployment能力
istio流量管理destinationRule warmupDurationSecs设置、deployment中的minReadySeconds为预热持续时间
2、自定义脚本逻辑(本质利用deployment中的Prestart能力)
将自定义参数等打到镜像形成脚本，通过prestart执行预热
```

# 4、kubernetes中controller和operator的区别？
```
Controller 是 Kubernetes 的核心组件之一，用于确保集群的当前状态与期望状态一致。它通过 控制循环（Control Loop） 不断监控资源的状态，并根据差异执行调谐（Reconcile）操作。
Operator 是一自定义控制器，用于管理 Kubernetes 外部的复杂应用或服务。它基于 Kubernetes 的 Custom Resource Definitions (CRDs) 和 Controller 模式，将领域知识编码到 Kubernetes 中。
```

# 5、kubernetes中webhook的作用是什么？

在 Kubernetes 中，**Webhook** 是一种扩展机制，允许集群在特定事件发生时向外部服务发送 HTTP 请求，以便执行自定义逻辑。Webhook 通常用于 **准入控制（Admission Control）** 和 **动态配置**，能够增强 Kubernetes 的安全性和灵活性。

---

### **1. Webhook 的作用**
Webhook 主要用于以下场景：
1. **准入控制（Admission Control）**：
   - 在资源创建、更新或删除时，拦截请求并执行自定义验证或修改。
   - 例如：验证资源是否符合安全策略、注入默认值、修改资源配置等。
2. **动态配置**：
   - 在资源创建时动态注入配置（如 Sidecar 容器、环境变量）。
3. **审计和日志**：
   - 记录资源变更事件，用于审计或监控。

---

### **2. Webhook 的类型**
Kubernetes 支持两种主要的 Webhook 类型：
1. **Validating Admission Webhook**：
   - 用于验证资源是否符合特定规则。
   - 如果验证失败，资源创建或更新请求会被拒绝。
2. **Mutating Admission Webhook**：
   - 用于修改资源的内容。
   - 例如：注入 Sidecar 容器、添加注解或标签。

---

### **3. Webhook 的工作流程**
1. **用户发起请求**：
   - 用户通过 `kubectl` 或 API 创建、更新或删除资源。
2. **API Server 拦截请求**：
   - API Server 将请求转发给配置的 Webhook。
3. **Webhook 处理请求**：
   - Webhook 服务接收请求，执行自定义逻辑（验证或修改）。
4. **返回响应**：
   - Webhook 返回允许或拒绝的响应。
5. **API Server 处理响应**：
   - 根据 Webhook 的响应，决定是否继续处理请求。

---

### **4. Webhook 的使用示例**
以下是一个完整的示例，展示如何配置和使用 **Validating Admission Webhook** 和 **Mutating Admission Webhook**。

#### **步骤 1：部署 Webhook 服务**
Webhook 服务是一个 HTTP 服务器，用于处理 API Server 的请求。以下是一个简单的 Webhook 服务示例（使用 Python Flask）：

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/validate', methods=['POST'])
def validate():
    request_data = request.json
    # 自定义验证逻辑
    if request_data["request"]["object"]["metadata"]["labels"].get("allow") != "true":
        return jsonify({
            "response": {
                "allowed": False,
                "status": {"message": "Label 'allow=true' is required"}
            }
        })
    return jsonify({"response": {"allowed": True}})

@app.route('/mutate', methods=['POST'])
def mutate():
    request_data = request.json
    # 自定义修改逻辑
    patch = [
        {
            "op": "add",
            "path": "/metadata/labels/added-by-webhook",
            "value": "true"
        }
    ]
    return jsonify({
        "response": {
            "allowed": True,
            "patch": patch,
            "patchType": "JSONPatch"
        }
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=443, ssl_context=('webhook.crt', 'webhook.key'))
```

- **Validating Webhook**：`/validate` 路径用于验证资源。
- **Mutating Webhook**：`/mutate` 路径用于修改资源。

#### **步骤 2：创建 TLS 证书**
Webhook 服务必须通过 HTTPS 提供服务。生成自签名证书：

```bash
openssl req -x509 -newkey rsa:2048 -keyout webhook.key -out webhook.crt -days 365 -nodes -subj "/CN=webhook.default.svc"
```

#### **步骤 3：部署 Webhook 服务到 Kubernetes**
将 Webhook 服务部署为 Kubernetes 服务：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webhook
  namespace: default
spec:
  ports:
    - port: 443
      targetPort: 443
  selector:
    app: webhook
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webhook
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webhook
  template:
    metadata:
      labels:
        app: webhook
    spec:
      containers:
        - name: webhook
          image: my-webhook-image
          ports:
            - containerPort: 443
          volumeMounts:
            - name: cert
              mountPath: /etc/certs
              readOnly: true
      volumes:
        - name: cert
          secret:
            secretName: webhook-cert
---
apiVersion: v1
kind: Secret
metadata:
  name: webhook-cert
  namespace: default
type: Opaque
data:
  webhook.crt: $(base64 -w 0 webhook.crt)
  webhook.key: $(base64 -w 0 webhook.key)
```

#### **步骤 4：配置 Validating Webhook**
创建 Validating Webhook 配置：

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: validating-webhook
webhooks:
  - name: validate.example.com
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    clientConfig:
      service:
        name: webhook
        namespace: default
        path: "/validate"
        port: 443
      caBundle: $(base64 -w 0 webhook.crt)
    admissionReviewVersions: ["v1"]
    sideEffects: None
    timeoutSeconds: 5
```

#### **步骤 5：配置 Mutating Webhook**
创建 Mutating Webhook 配置：

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: mutating-webhook
webhooks:
  - name: mutate.example.com
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    clientConfig:
      service:
        name: webhook
        namespace: default
        path: "/mutate"
        port: 443
      caBundle: $(base64 -w 0 webhook.crt)
    admissionReviewVersions: ["v1"]
    sideEffects: None
    timeoutSeconds: 5
```

#### **步骤 6：测试 Webhook**
1. 创建一个 Pod：
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: test-pod
     labels:
       allow: "false"
   spec:
     containers:
       - name: nginx
         image: nginx
   ```
   - 由于 `allow` 标签不为 `true`，Validating Webhook 会拒绝创建。

2. 创建一个带有 `allow=true` 标签的 Pod：
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: test-pod
     labels:
       allow: "true"
   spec:
     containers:
       - name: nginx
         image: nginx
   ```
   - Mutating Webhook 会添加标签 `added-by-webhook: true`。

---

### **5. 注意事项**
1. **性能影响**：
   - Webhook 会增加 API Server 的延迟，确保 Webhook 服务高效。
2. **高可用性**：
   - Webhook 服务必须高可用，否则会影响集群操作。
3. **安全性**：
   - 使用 TLS 加密通信，确保 Webhook 服务的安全性。

通过 Webhook，Kubernetes 可以实现高度定制化的准入控制和资源管理，满足复杂的业务需求。

# 6、kubernetes中deployment的minReadySeconds和prestart的作用？
minReadySeconds

- minReadySeconds 定义了 Pod 在就绪后必须保持运行的最短时间（以秒为单位），然后才能被视为可用。它主要用于控制滚动更新（Rolling Update）的速度，确保新 Pod 在接收流量之前已经稳定运行。
- 防止新 Pod 过早接收流量，导致应用启动未完成时出现错误。在滚动更新时，减缓更新速度，避免一次性替换过多 Pod。
- 当新 Pod 启动并通过readinessProbe检测后，Kubernetes会等待 minReadySeconds时间（30 秒）。在这 30秒内，即使Pod已经就绪，也不会将其添加到Service的Endpoints中。30秒后，Pod被视为可用，滚动更新继续。

postStart（生命周期钩子）

- postStart 是容器生命周期钩子（Lifecycle Hook）的一部分，在容器启动后立即执行。它用于在容器主进程启动前执行一些初始化任务（如配置检查、依赖准备）。
- 在容器启动前执行脚本或命令。确保容器启动时依赖的服务或资源已就绪
- 当容器启动后，Kubernetes会立即执行postStart钩子。postStart钩子与容器的主进程并行执行。如果 postStart钩子执行失败，容器会被杀死并重启。


# 7、deployment支持的三种健康检测类型？
在 Kubernetes 中，健康检测是确保应用稳定运行的重要机制。Kubernetes 提供了三种类型的健康检测：

1. **存活探针（Liveness Probe）**
2. **就绪探针（Readiness Probe）**
3. **启动探针（Startup Probe）**

每种探针都有详细的控制参数，用于定义检测的行为和条件。以下是每种探针的详细说明及其控制参数。

---

## **1. 存活探针（Liveness Probe）**
### **作用**
- 用于检测容器是否处于运行状态。
- 如果探针失败，Kubernetes 会认为容器不健康，并重启容器。

### **适用场景**
- 检测应用是否陷入死锁或不可用状态。

### **控制参数**
| 参数                  | 说明                                                                 |
|-----------------------|--------------------------------------------------------------------|
| `initialDelaySeconds` | 容器启动后，等待多少秒开始执行探针（默认 0）。                       |
| `periodSeconds`       | 探针的执行间隔时间（默认 10 秒）。                                  |
| `timeoutSeconds`      | 探针的超时时间，超过此时间未响应则视为失败（默认 1 秒）。            |
| `successThreshold`    | 探针连续成功多少次才视为成功（默认 1）。                             |
| `failureThreshold`    | 探针连续失败多少次才视为失败（默认 3）。                             |

### **示例**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 1
  successThreshold: 1
  failureThreshold: 3
```

---

## **2. 就绪探针（Readiness Probe）**
### **作用**
- 用于检测容器是否已准备好接收流量。
- 如果探针失败，Kubernetes 会将该容器从 Service 的 Endpoints 中移除，直到探针成功。

### **适用场景**
- 检测应用是否已完成初始化，是否可以处理请求。

### **控制参数**
| 参数                  | 说明                                                                 |
|-----------------------|--------------------------------------------------------------------|
| `initialDelaySeconds` | 容器启动后，等待多少秒开始执行探针（默认 0）。                       |
| `periodSeconds`       | 探针的执行间隔时间（默认 10 秒）。                                  |
| `timeoutSeconds`      | 探针的超时时间，超过此时间未响应则视为失败（默认 1 秒）。            |
| `successThreshold`    | 探针连续成功多少次才视为成功（默认 1）。                             |
| `failureThreshold`    | 探针连续失败多少次才视为失败（默认 3）。                             |

### **示例**
```yaml
readinessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 1
  successThreshold: 1
  failureThreshold: 3
```

---

## **3. 启动探针（Startup Probe）**
### **作用**
- 用于检测容器是否已成功启动。
- 如果启动探针失败，Kubernetes 不会重启容器，而是继续等待，直到探针成功。
- 启动探针成功后，存活探针和就绪探针才会生效。

### **适用场景**
- 检测启动时间较长的应用（如 Java 应用）是否已完成启动。

### **控制参数**
| 参数                  | 说明                                                                 |
|-----------------------|--------------------------------------------------------------------|
| `initialDelaySeconds` | 容器启动后，等待多少秒开始执行探针（默认 0）。                       |
| `periodSeconds`       | 探针的执行间隔时间（默认 10 秒）。                                  |
| `timeoutSeconds`      | 探针的超时时间，超过此时间未响应则视为失败（默认 1 秒）。            |
| `successThreshold`    | 探针连续成功多少次才视为成功（默认 1）。                             |
| `failureThreshold`    | 探针连续失败多少次才视为失败（默认 3）。                             |

### **示例**
```yaml
startupProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 1
  successThreshold: 1
  failureThreshold: 30  # 允许较长的启动时间
```

---

## **4. 探针的检测方式**
每种探针都支持以下三种检测方式：

### **1. HTTP 检测（httpGet）**
- 向容器发送 HTTP GET 请求，根据响应状态码判断是否成功（2xx 或 3xx 表示成功）。
- **参数**：
  - `path`：请求路径。
  - `port`：请求端口。
  - `httpHeaders`：自定义 HTTP 头。

#### **示例**
```yaml
httpGet:
  path: /healthz
  port: 8080
  httpHeaders:
    - name: Custom-Header
      value: Awesome
```

### **2. 命令检测（exec）**
- 在容器内执行命令，根据命令的退出状态码判断是否成功（0 表示成功）。
- **参数**：
  - `command`：执行的命令。

#### **示例**
```yaml
exec:
  command:
    - cat
    - /tmp/healthy
```

### **3. TCP 检测（tcpSocket）**
- 尝试与容器的指定端口建立 TCP 连接，连接成功则视为成功。
- **参数**：
  - `port`：检测的端口。

#### **示例**
```yaml
tcpSocket:
  port: 8080
```

---

## **5. 综合示例**
以下是一个包含三种探针的完整示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: my-app
      image: my-app:1.0
      ports:
        - containerPort: 8080
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
      readinessProbe:
        exec:
          command:
            - cat
            - /tmp/healthy
        initialDelaySeconds: 5
        periodSeconds: 10
      startupProbe:
        tcpSocket:
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
        failureThreshold: 30
```

---

## **6. 总结**
| 探针类型       | 作用                           | 检测方式         | 关键参数                          |
|----------------|--------------------------------|------------------|-----------------------------------|
| **Liveness**   | 检测容器是否存活               | HTTP、Exec、TCP  | `initialDelaySeconds`、`failureThreshold` |
| **Readiness**  | 检测容器是否就绪               | HTTP、Exec、TCP  | `initialDelaySeconds`、`failureThreshold` |
| **Startup**    | 检测容器是否启动完成           | HTTP、Exec、TCP  | `initialDelaySeconds`、`failureThreshold` |

通过合理配置这三种探针，可以确保 Kubernetes 中的应用在启动、运行和终止时都能保持健康状态。

# 8、一个公网域名请求进入带有istio功能的k8s集群中的某个pod，需要哪些步骤和经过哪些组件?
一个公网域名请求进入带有 Istio 的 Kubernetes 集群中的 Pod，需经过以下步骤和组件：

---

### **步骤详解**

1. **DNS 解析**  
   - 用户访问公网域名（如 `example.com`），DNS 服务器将其解析为 **Istio Ingress Gateway** 的外部 IP 地址（通常是云服务商提供的负载均衡器 IP）。

2. **外部负载均衡器**  
   - 请求到达云服务商的负载均衡器（如 AWS ALB、GCP Load Balancer），由其将流量转发到 Kubernetes 集群的 **Istio Ingress Gateway**。

3. **Istio Ingress Gateway**  
   - **组件**：运行 Envoy 代理的 Pod（通常通过 `istio-ingressgateway` 服务暴露，类型为 `LoadBalancer`）。
   - **作用**：作为集群入口，接收外部流量。Envoy 根据配置的 `Gateway` 资源确定监听的端口、协议（如 HTTP/HTTPS）和 TLS 证书。

4. **路由匹配（Gateway 和 VirtualService）**  
   - **Gateway 资源**：定义入口流量的监听规则（如端口 80/443）。
   - **VirtualService 资源**：根据请求的域名（Host）、路径（Path）等规则，将流量路由到目标 Kubernetes 服务（如 `my-service`）。

5. **服务发现与负载均衡**  
   - **目标 Kubernetes Service**：通过标签选择器（Selector）关联后端 Pod。
   - **Envoy 动态路由**：Istio 控制平面（Istiod）将服务端点（Endpoints）信息动态下发到 Envoy，绕过传统的 kube-proxy，直接负载均衡到 Pod。

6. **目标 Pod 的 Envoy Sidecar**  
   - **组件**：每个 Pod 注入的 Envoy 容器（通过 Istio Sidecar 注入）。
   - **作用**：拦截入站流量，执行策略（如 mTLS 加密、认证授权、流量监控、熔断等），然后将请求转发到应用容器。

7. **应用容器处理请求**  
   - 最终由 Pod 内的业务容器处理请求并返回响应。

---

### **关键组件**

1. **DNS 服务器**  
   - 将公网域名解析为 Istio Ingress Gateway 的外部 IP。

2. **云服务商负载均衡器**  
   - 将外部流量转发到集群的 Istio Ingress Gateway。

3. **Istio Ingress Gateway**  
   - Envoy 代理 Pod，作为流量入口，配置通过 `Gateway` 资源定义。

4. **VirtualService 和 DestinationRule**  
   - `VirtualService`：定义路由规则（如按路径或 Header 分流）。
   - `DestinationRule`：定义子集（Subset）、负载均衡策略（如轮询、最少连接）。

5. **Kubernetes Service 和 Endpoints**  
   - Service 提供逻辑分组，Endpoints 维护 Pod IP 列表供 Envoy 动态路由。

6. **Envoy Sidecar**  
   - 在目标 Pod 中处理流量策略（如安全、观测、流量控制）。

7. **Istiod（控制平面）**  
   - 管理配置下发（如将路由规则同步到 Envoy 代理）。

---

### **示例配置**

1. **Gateway 资源**  
   ```yaml
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: example-gateway
   spec:
     selector:
       istio: ingressgateway
     servers:
     - port:
         number: 80
         name: http
         protocol: HTTP
       hosts:
       - "example.com"
   ```

2. **VirtualService 资源**  
   ```yaml
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: example-vs
   spec:
     hosts:
     - "example.com"
     gateways:
     - example-gateway
     http:
     - route:
       - destination:
           host: my-service
           port:
             number: 8080
   ```

---

### **流量路径总结**
```
公网用户 → DNS → 云负载均衡器 → Istio Ingress Gateway → VirtualService → 目标 Service 的 Pod（Envoy Sidecar → 应用容器）
```

通过以上流程，Istio 实现了对外部流量的精细控制（如灰度发布、安全策略），同时保持了 Kubernetes 服务的原生灵活性。


# 9、

# 10、
