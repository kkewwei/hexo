---
title: Flink原理-slot分配
date: 2019-03-12 20:04:11
tags:
---
本文将从ExecutionGraph开始向后将起JobManager是如何进行slot的分配及部署的, ExecutionGraph定义了TaskManager的并发分布结构, 作为任务执行的以后一层逻辑结构, 也是最核心数据结构
本文先盗一张官方的ExecutionGraph的结构:
