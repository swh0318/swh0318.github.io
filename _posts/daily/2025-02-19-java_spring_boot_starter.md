---
layout: post
title: "[每日一文]如何编写一个SpringBoot Starter"
date: 2025-02-19
tags: [java]
comments: false
author: 小辣条
toc : false
---
编写一个 Spring Boot Starter 是一种将自定义功能封装为可重用组件的好方法；下面是创建自定义 Starter 的完整步骤。
<!-- more -->

# 如何编写一个 Spring Boot Starter

## 1. 项目结构规划

标准的 Starter 项目通常包含两个模块：
```
my-spring-boot-starter/
├── my-spring-boot-starter-autoconfigure (自动配置核心)
└── my-spring-boot-starter (Starter 入口，依赖autoconfigure)
```

## 2. 创建自动配置模块

### 基本依赖 (pom.xml)
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-autoconfigure</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

### 创建配置属性类
```java
@ConfigurationProperties(prefix = "my.starter")
public class MyStarterProperties {
    private String name = "default";
    private int timeout = 1000;
    // getters/setters
}
```

### 创建自动配置类
```java
@Configuration
@EnableConfigurationProperties(MyStarterProperties.class)
@ConditionalOnClass(MyService.class) // 当类路径存在MyService时生效
@ConditionalOnProperty(prefix = "my.starter", value = "enabled", havingValue = "true", matchIfMissing = true)
public class MyStarterAutoConfiguration {
    
    @Autowired
    private MyStarterProperties properties;
    
    @Bean
    @ConditionalOnMissingBean // 当容器中没有该Bean时创建
    public MyService myService() {
        return new MyService(properties.getName(), properties.getTimeout());
    }
}
```

### 注册自动配置
在 `resources/META-INF` 下创建 `spring.factories` 文件：
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.mystarter.autoconfigure.MyStarterAutoConfiguration
```

## 3. 创建 Starter 模块

### 基本依赖 (pom.xml)
```xml
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>my-spring-boot-starter-autoconfigure</artifactId>
        <version>${project.version}</version>
    </dependency>
</dependencies>
```

## 4. 高级功能

### 条件化配置
```java
@ConditionalOnWebApplication // 仅Web应用生效
@ConditionalOnMissingClass("com.example.OtherService") // 当不存在指定类时生效
@ConditionalOnExpression("${my.starter.enabled:true}") // SpEL表达式条件
```

### 自定义指标
```java
@Bean
public MyStarterMetrics myStarterMetrics() {
    return new MyStarterMetrics();
}
```

### 健康检查
```java
@Bean
public HealthIndicator myStarterHealth() {
    return () -> Health.up().withDetail("version", "1.0").build();
}
```

## 5. 测试 Starter

### 测试自动配置
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class MyStarterAutoConfigurationTest {

    @Autowired(required = false)
    private MyService myService;
    
    @Test
    public void testMyServiceAutoConfigured() {
        assertNotNull(myService);
    }
}
```

## 6. 发布和使用

1. 安装到本地仓库或发布到Maven中央仓库
2. 其他项目引入依赖：
```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>my-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

3. 在 application.properties 中配置：
```properties
my.starter.name=custom-name
my.starter.timeout=2000
```

## 最佳实践

1. **命名规范**：遵循 `spring-boot-starter-{name}` 模式
2. **模块分离**：将自动配置和Starter分开
3. **合理使用条件注解**：确保按需加载
4. **提供默认配置**：减少用户必须的配置
5. **完善文档**：说明配置项和使用方法

通过以上步骤，您可以创建一个功能完善、易于集成的 Spring Boot Starter。

---
### 保持好奇，勇敢打破边界，做永远不被定义的人，祝你快乐❤️