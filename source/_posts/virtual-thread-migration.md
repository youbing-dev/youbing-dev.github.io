---
title: Spring Boot 3 虚拟线程实战：从 Thread Pool 到 Virtual Thread 的迁移记录
date: 2026-06-28 10:00:00
tags: [Java, Spring Boot, Virtual Thread]
categories: [后端开发, Java]
description: 记录了一次将核心订单服务从传统线程池迁移到 Java 21 虚拟线程的完整过程，包含压测数据对比和踩过的坑。
---

## 背景

我们的核心订单服务一直使用传统的 Tomcat 线程池模型，在高并发场景下经常出现线程饥饿和响应延迟飙升的问题。Java 21 引入的虚拟线程（Virtual Threads）承诺能以极低的资源开销支撑百万级并发，我决定在一个非关键链路上先做试点。

## 迁移步骤

### 1. 升级 Spring Boot 到 3.2+

Spring Boot 3.2 原生支持虚拟线程，只需要在配置文件中开启：

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

### 2. 替换自定义线程池

原来的 `ThreadPoolTaskExecutor` 全部替换为虚拟线程执行器：

```java
@Bean
public Executor taskExecutor() {
    return Executors.newVirtualThreadPerTaskExecutor();
}
```

### 3. 处理 synchronized 和 ThreadLocal

虚拟线程不适合 `synchronized` 块和 `ThreadLocal` 重度使用的场景，需要逐一排查替换：

- `synchronized` → `ReentrantLock`
- `ThreadLocal` → `ScopedValue`（Java 21 Preview）

## 压测结果

| 指标 | 迁移前 | 迁移后 | 变化 |
|------|--------|--------|------|
| P99 延迟 | 320ms | 85ms | -73% |
| 最大并发 | 2,000 | 50,000 | 25x |
| 内存占用 | 4.2GB | 1.8GB | -57% |

## 踩过的坑

最大的坑是 JDBC 驱动兼容性。HikariCP 在早期版本中对虚拟线程支持不完善，升级到 5.1.0 后才稳定。另外，Redis 客户端 Lettuce 在虚拟线程下表现良好，无需额外调整。

## 总结

虚拟线程不是银弹，但在 I/O 密集型服务上确实带来了显著的性能提升。建议先在非关键服务上试点，验证稳定性后再逐步推广。
