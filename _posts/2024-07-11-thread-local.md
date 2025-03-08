---
layout: post
title: "ThreadLocal 常见的使用场景"
date: 2024-07-11 10:00:00 +0800
tags: [ Jekyll, Markdown, 博客 ]
author: "孙珂"
---

#  ThreadLocal 常见的使用场景

ThreadLocal 是一种维持线程封闭性的方法，使线程中的某个值和保存值的对象关联起来。get() 总是返回当前执行县城在调用 set() 时设置的最新值。

ThreadLocal 避免了在调用每个方法时都要传递上下文信息。

ThreadLocal 通过为每个线程提供独立的变量副本，线程安全，每个线程维护独立的变量，避免了多线程共享数据时的竞争问题，从而保证了线程安全性。


## 官方文档定义

```

 Each thread holds an implicit reference to its copy of a thread-local variable as long as the thread is alive 
 and the {@code ThreadLocal} instance is accessible; 
 
 after a thread goes away, 
 all of its copies of thread-local instances are subject to garbage collection 
 (unless other references to these copies exist).

```

## 使用场景

使用 ThreadLocal 存储 traceId，确保同一请求的所有日志都带上相同的 traceId。需要注意的是，一定要进行 remove() 操作，防止数据污染和内存泄漏;

```java
    private static final ThreadLocal<Integer> currentUser = ThreadLocal.withInitial(() -> 1);
```

## 注意事项

当一个线程结束时，ThreadLocal 会随之被垃圾回收，通常不会造成内存泄漏。

但如果使用到线程池的时候，由于线程池的线程是复用的，如果不及时清理 ThreadLocal 的值，会导致数据污染，还可能会造成内存泄漏。




## 参考

- [ThreadLocal](https://docs.oracle.com/javase/8/docs/api/java/lang/ThreadLocal.html)
- [Java并发编程实战]()