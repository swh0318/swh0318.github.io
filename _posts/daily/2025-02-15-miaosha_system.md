---
layout: post
title: "[每日一文]秒杀系统设计要点"
date: 2025-02-15
tags: [系统设计,每日一文]
comments: false
author: 小辣条
toc : false
---

<!-- more -->
## 设计要点
```
1、页面端做重复点击拦截、网关层拦截限流
2、采用将商品信息、商品库存缓存Redis的方式来提高系统的响应和拦截无效的请求
3、通过异步下单（MQ）的方式来给流量消峰处理，维持系统的稳定
4、采用WebSocket的方式将下单成功的消息推送给客户端
5、超卖问题采用乐观锁的机制做兜底处理
6、针对用户超时未支付或者多次下单同一个商品导致商品少卖的问题，采用真实库存回补的方式来处理预库存（Redis的中的库存）
```
## 参考
https://mp.weixin.qq.com/s/zCbRiA6c9phXs5BJ-hqTfQ