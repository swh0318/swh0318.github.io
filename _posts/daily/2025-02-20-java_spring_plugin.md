---
layout: post
title: "[每日一文]Java父模块和子模块的插件plugin是什么关系"
date: 2025-02-20
tags: [java]
comments: false
author: 小辣条
toc : false
---
在 Maven 多模块项目中，父模块（Parent POM）与子模块（Child Module）的插件（Plugins）关系遵循以下规则：
<!-- more -->

在 Maven 多模块项目中，父模块（Parent POM）与子模块（Child Module）的插件（Plugins）关系遵循以下规则：

---

### **1. 插件继承机制**
#### **父模块的作用**
- **统一管理插件版本**：通过 `<pluginManagement>` 定义插件版本和默认配置（**声明但不立即生效**）
- **强制继承插件**：直接在 `<plugins>` 中定义的插件会**强制所有子模块继承**

#### **子模块的行为**
- 可**继承**父模块 `<pluginManagement>` 中定义的插件（需显式声明）
- 可**覆盖**父模块的插件配置
- 可**添加**父模块未定义的新插件

---

### **2. 典型配置对比**
#### **父模块 POM (parent/pom.xml)**
```xml
<project>
  <!-- 定义插件管理（子模块需显式引用） -->
  <pluginManagement>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.11.0</version>
        <configuration>
          <source>17</source>
          <target>17</target>
        </configuration>
      </plugin>
    </plugins>
  </pluginManagement>

  <!-- 强制所有子模块继承的插件 -->
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>3.1.2</version>
    </plugin>
  </plugins>
</project>
```

#### **子模块 POM (child/pom.xml)**
```xml
<project>
  <!-- 继承父模块 pluginManagement 中的编译器插件 -->
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <!-- 可覆盖父配置 -->
      <configuration>
        <source>21</source>
        <target>21</target>
      </configuration>
    </plugin>

    <!-- 添加父模块未定义的新插件 -->
    <plugin>
      <groupId>org.jacoco</groupId>
      <artifactId>jacoco-maven-plugin</artifactId>
      <version>0.8.11</version>
    </plugin>
  </plugins>
</project>
```

---

### **3. 优先级规则**
| 配置来源                | 优先级 | 说明                                 |
|------------------------|-------|--------------------------------------|
| 子模块 `<plugins>`       | 最高   | 完全覆盖父模块同名插件的配置           |
| 父模块 `<plugins>`       | 中     | 强制继承，子模块无法移除但可覆盖配置    |
| 父模块 `<pluginManagement>` | 最低   | 需子模块显式引用才会生效               |

---

### **4. 最佳实践**
#### **父模块应**：
1. 在 `<pluginManagement>` 中统一管理**所有公共插件的版本**
2. 在 `<plugins>` 中仅定义**必须全局强制使用的插件**（如安全扫描插件）
3. 避免在父模块中定义**子模块可能不需要的插件**

#### **子模块应**：
1. 通过 `<plugins>` 显式声明需要使用的插件
2. 优先继承父模块的 `<pluginManagement>` 配置
3. 仅在必要时覆盖父配置（如修改 `configuration` 或 `executions`）

---

### **5. 常见问题解决**
#### **问题：子模块不想继承父插件**
```xml
<!-- 在子模块中排除插件 -->
<plugin>
  <groupId>父插件GroupId</groupId>
  <artifactId>父插件ArtifactId</artifactId>
  <executions>
    <execution>
      <id>父插件ExecutionId</id>
      <phase>none</phase> <!-- 禁用该执行 -->
    </execution>
  </executions>
</plugin>
```

#### **问题：插件版本冲突**
在父模块的 `<pluginManagement>` 中明确定义版本：
```xml
<pluginManagement>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-jar-plugin</artifactId>
      <version>3.3.0</version>
    </plugin>
  </plugins>
</pluginManagement>
```

---

通过合理使用父子模块的插件管理，可实现 **「集中管控」** 与 **「灵活定制」** 的平衡。

---
### 保持好奇，勇敢打破边界，做永远不被定义的人，祝你快乐❤️