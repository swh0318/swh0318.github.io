---
layout: post
title: "HTTPS加密流程"
date:   2025-02-09
tags: [https]
comments: false
author: 小辣条
toc : false
---

HTTPS加密流程
<!-- more -->
<div style="display: flex; justify-content: center;"> 
<img src="https://raw.githubusercontent.com/swh0318/swh0318.github.io/refs/heads/main/_posts/2025/assets/2025-02-09-https_jiamiguocheng/image.png" width="50%" height="50%" />
</div>

下面我们将详细介绍每个步骤的细节。

### 步骤 1：客户端向服务器发送 HTTPS 请求
当客户端需要从服务器获取数据时，它会向服务器发送一个 HTTPS 请求。这个请求包括请求的 URL、HTTP 请求头和请求体。例如：
```
GET / HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:60.0) Gecko/20100101 Firefox/60.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Upgrade-Insecure-Requests: 1

```
### 步骤 2：服务器将公钥证书发送给客户端
当服务器接收到 HTTPS 请求后，它会将公钥证书发送给客户端。公钥证书中包含了服务器的公钥、服务器的域名、证书颁发机构、证书有效期等信息。
客户端接收到证书后，会从中提取出服务器的公钥。

### 步骤 3：客户端验证服务器的证书
客户端接收到服务器的证书后，会对其进行验证，以确保该证书是由可信任的证书颁发机构颁发的，并且证书中的域名和服务器的实际域名一致。

如果证书验证失败，客户端会中断连接。如果验证通过，客户端会生成一个用于会话的对称密钥。

### 步骤 4：客户端生成一个用于会话的对称密钥
客户端生成一个用于会话的对称密钥。对称密钥是一种加密方式，它使用相同的密钥进行加密和解密。这个密钥只存在于客户端和服务器之间，因此被称为“对称”。

### 步骤 5：客户端使用服务器的公钥对对称密钥进行加密，并将加密后的密钥发送给服务器
客户端使用服务器的公钥对对称密钥进行加密，并将加密后的密钥发送给服务器。在这个过程中，客户端和服务器都知道对称密钥，但是只有客户端知道对称密钥的值。

### 步骤 6：服务器使用私钥对客户端发送的加密密钥进行解密，得到对称密钥
服务器使用私钥对客户端发送的加密密钥进行解密，得到对称密钥。由于私钥只在服务器端保存，因此只有服务器才能解密客户端发送的加密密钥，并得到对称密钥的值。

### 步骤 7：服务器和客户端使用对称密钥进行加密和解密数据传输
服务器和客户端使用对称密钥进行加密和解密数据传输。这个对称密钥只存在于客户端和服务器之间，因此对数据的加密和解密只有客户端和服务器可以进行。

## 参考
[HTTPS 的加密过程及其工作原理](https://xie.infoq.cn/article/007a9bd16f44303fbd8b40689)
