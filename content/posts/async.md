---
title: 什么是异步？
date: 2022-03-07T11:33:54+08:00
draft: true
tags:
  - async
---

阻塞和非阻塞的概念来自于操作系统的网络 IO，我们以 Linux 为例

Linux 有以下几种网络 IO 模型
- blocking IO
- nonblocking IO
- IO multiplexing
- asynchronous IO

可见，阻塞还是非阻塞看的是调用 API 后会不会把当前线程阻塞住







同步 API you call framework(kernel).
异步 API 就是 framework(kernel) call you.


由此可见，异步 API 当然不会阻塞线程，所以异步 API 都是非阻塞的

同步 API 可能是阻塞的(blocking IO)，也可能是非阻塞的(nonblocking IO)




