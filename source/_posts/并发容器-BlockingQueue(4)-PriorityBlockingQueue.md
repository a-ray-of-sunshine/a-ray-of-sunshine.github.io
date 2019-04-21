---
title: PriorityBlockingQueue
date: 2016-8-8 17:33:05
tags: [BlockingQueue, PriorityBlockingQueue]
---

无界的 blocking queue，因为它是 unbouned 的，所以从逻辑上来说，put 操作是不会失败的，除非内存资源被耗尽（causing OutOfMemoryError）。

不支持 null 元素。

iterator 不保证顺序。

内部使用二叉堆（binary heap）来实现维护入队元素的顺序。
