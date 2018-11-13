---
title: LockSupport原理解读
date: 2018-11-10 17:16:56
tags:
---
LockSupport作为并发的基础, 在CountDownLatch、ReentrantLock、Semaphore、ReentrantReadWriteLock中都是作为阻塞/唤醒线程的基本工具, 因此, 很有必要了解LockSupport的用法及原理, 以下展示了LockSupport的基本用法:
```
```