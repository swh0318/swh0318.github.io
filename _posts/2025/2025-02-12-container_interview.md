---
layout: post
title: "容器技术常见面试题"
date:   2025-02-12
tags: [docker, container, 面试]
comments: false
author: 小辣条
toc : true
---
收集了一些容器方面常见的面试题，希望对大家有帮助
<!-- more -->

## 1、docker指令中cmd和entrypoint的区别
在 Docker 中，CMD 和 ENTRYPOINT 都用于定义容器启动时执行的命令，但它们的用途和行为有所不同

CMD提供默认命令或参数，可以被 docker run 参数覆盖，如果 Dockerfile 中有多个CMD，只有最后一个生效；

ENTRYPOINT定义容器启动时的主命令，不会被 docker run 的参数覆盖，而是将 docker run 的参数追加到 ENTRYPOINT 之后
## 2、



