---
layout: post
title: "MySQL常见面试题"
date: 2025-02-14
tags: [MySQL, 面试]
comments: false
author: 小辣条
toc : true
---
收集了一些MySQL方面常见的面试题，希望对大家有帮助
<!-- more -->

# 1、select * .... for update语句是行锁还是表锁？
没用索引/主键的话就是表锁，否则就是是行锁。

# 2、一条sql的执行过程是怎样的？
