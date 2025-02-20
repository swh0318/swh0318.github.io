---
layout: post
title: "servicemesh常见面试题"
date: 2025-02-20
tags: [servicemesh, istio, 面试]
comments: false
author: 小辣条
toc : true
---
收集了一些servicemesh方面常见的面试题，希望对大家有帮助
<!-- more -->

# 1、istio都有哪些组件和作用？
Istio 是一个开源的服务网格（Service Mesh），用于管理、保护和监控微服务架构中的服务通信。它由多个核心组件组成，每个组件都有特定的功能。以下是 Istio 的主要组件及其作用：

---

## **1. 数据平面（Data Plane）**
数据平面由 **Envoy 代理** 组成，负责处理服务之间的通信。

### **Envoy**
- **作用**：
  - 拦截所有进出服务的流量。
  - 实现流量管理（如负载均衡、路由、重试、熔断等）。
  - 提供安全性（如 mTLS 加密、身份验证）。
  - 收集遥测数据（如指标、日志、追踪）。
- **部署方式**：
  - 以 Sidecar 容器的形式注入到每个 Pod 中，与应用容器共存。

---

## **2. 控制平面（Control Plane）**
控制平面负责管理和配置数据平面，包括以下组件：

### **Istiod**
- **作用**：
  - 核心控制平面组件，整合了 Pilot、Citadel 和 Galley 的功能。
  - 提供服务发现、配置管理和证书颁发。
  - 将配置推送到 Envoy 代理。
- **部署方式**：
  - 以 Kubernetes Deployment 的形式部署在集群中。

#### **Pilot（已整合到 Istiod 中）**
- **作用**：
  - 负责服务发现和流量管理。
  - 将路由规则、负载均衡策略等配置推送到 Envoy。
- **部署方式**：
  - 作为 Istiod 的一部分部署。

#### **Citadel（已整合到 Istiod 中）**
- **作用**：
  - 负责证书管理和身份验证。
  - 为服务之间提供 mTLS 加密。
- **部署方式**：
  - 作为 Istiod 的一部分部署。

#### **Galley（已整合到 Istiod 中）**
- **作用**：
  - 负责配置验证和分发。
  - 确保用户提供的配置是有效的。
- **部署方式**：
  - 作为 Istiod 的一部分部署。

---

## **3. 可观测性组件**
Istio 提供了丰富的可观测性工具，用于监控和分析服务网格中的流量。

### **Prometheus**
- **作用**：
  - 收集和存储 Envoy 代理的指标数据。
  - 提供查询和告警功能。
- **部署方式**：
  - 以 Kubernetes Deployment 的形式部署。

### **Grafana**
- **作用**：
  - 可视化 Prometheus 中的指标数据。
  - 提供预定义的仪表盘，展示服务网格的运行状态。
- **部署方式**：
  - 以 Kubernetes Deployment 的形式部署。

### **Jaeger**
- **作用**：
  - 分布式追踪系统，用于分析请求在服务之间的流动。
  - 帮助诊断性能问题。
- **部署方式**：
  - 以 Kubernetes Deployment 的形式部署。

### **Kiali**
- **作用**：
  - 服务网格的可视化管理工具。
  - 提供拓扑图、指标、健康状态等信息。
- **部署方式**：
  - 以 Kubernetes Deployment 的形式部署。

---

## **4. 网关组件**
Istio 提供了两种网关，用于管理进出服务网格的流量。

### **Ingress Gateway**
- **作用**：
  - 管理进入服务网格的外部流量。
  - 提供负载均衡、TLS 终止等功能。
- **部署方式**：
  - 以 Kubernetes Deployment 的形式部署。

### **Egress Gateway**
- **作用**：
  - 管理从服务网格到外部服务的流量。
  - 提供流量控制和安全性。
- **部署方式**：
  - 以 Kubernetes Deployment 的形式部署。

---

## **5. 其他组件**

### **Sidecar Injector**
- **作用**：
  - 自动将 Envoy Sidecar 代理注入到应用的 Pod 中。
- **部署方式**：
  - 作为 Istiod 的一部分部署。

### **Mixer（已弃用）**
- **作用**：
  - 在早期版本中负责策略控制和遥测数据收集。
  - 在 Istio 1.5 及更高版本中，Mixer 的功能已整合到 Istiod 和 Envoy 中。
- **部署方式**：
  - 不再单独部署。

# 2、istio如何部署？

镜像：pilot  proxyv2(ingressgateway)





# 3、istio常见的资源及其使用
- VirtualService
```
```

- DestinatioRule
```
```

- ServicEntry
```
```
- Gateway
```
```
- AuthorizationPolicy
```
```
- EnvoyFilter
```
```



