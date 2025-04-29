---
layout: post
title: "[庖丁解牛]spring-boot开发中常见的注解和配置文件的作用"
date: 2025-02-26
tags: [java]
comments: false
author: 小辣条
toc : false
---
本文将详细介绍spring-boot开发中常见的注解和配置文件的作用。
<!-- more -->

# 常见配置文件的作用

## 一、bootstrap.yml

`bootstrap.yml` 是 Spring Cloud 项目中的一个特殊配置文件，它在应用程序启动的**最早阶段**被加载，优先于常规的 `application.yml` 或 `application.properties`。

### 核心作用

1. **引导阶段配置**：
   - 在 Spring ApplicationContext 创建之前加载
   - 用于获取后续配置所需的关键信息

2. **与常规配置文件的区别**：
   | 特性                | bootstrap.yml               | application.yml            |
   |---------------------|----------------------------|---------------------------|
   | **加载时机**         | 最先加载                   | 后于 bootstrap 加载        |
   | **主要用途**         | 获取外部配置源信息          | 应用业务配置               |
   | **典型配置项**       | 配置中心地址、加密信息      | 数据源、服务端口等         |

### 典型使用场景

#### 1. 连接配置中心
```yaml
# bootstrap.yml
spring:
  cloud:
    config:
      uri: http://config-server:8888
      name: my-application
      profile: dev
      label: master
```

#### 2. 加密配置
```yaml
spring:
  cloud:
    vault:
      host: 192.168.1.100
      port: 8200
      scheme: https
      authentication: TOKEN
      token: s.1234567890abcdef
```

### 3. 服务注册发现
```yaml
spring:
  application:
    name: order-service  # 必须在此处定义服务名
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
```

### 加载顺序

1. `bootstrap.yml` (或 `bootstrap.properties`)
2. 应用级配置 (`application-{profile}.yml`)
3. 外部配置 (如环境变量、命令行参数)

### 如何启用

在 Spring Cloud 项目中自动生效，如需禁用：
```yaml
# application.yml
spring:
  cloud:
    bootstrap:
      enabled: false
```

### 最佳实践

1. **最小化原则**：
   - 只存放必要的外部配置源信息
   - 避免放入业务相关配置

2. **安全考虑**：
   - 敏感信息应使用加密配置
   - 不要提交含密码的 bootstrap.yml 到代码库

3. **环境隔离**：
   - 可创建 `bootstrap-dev.yml`/`bootstrap-prod.yml`
   - 通过 `spring.profiles.active` 指定环境

### 常见问题解决

**问题**：配置中心地址无法加载  
**解决**：确保 bootstrap.yml 中的配置中心地址正确，且网络可达

**问题**：变量替换不生效  
**解决**：在 bootstrap.yml 中使用 `${}` 占位符时，确保这些值在早期可用

**问题**：与 Spring Boot 2.4+ 版本兼容性  
**注意**：Spring Boot 2.4 后需要显式引入依赖：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
```

`bootstrap.yml` 是 Spring Cloud 架构中实现外部化配置的关键机制，正确使用可以保证配置的有序加载和系统的稳定启动。


# 常见注解的作用

## 一、



---
### 保持好奇，勇敢打破边界，做永远不被定义的人，祝你快乐❤️