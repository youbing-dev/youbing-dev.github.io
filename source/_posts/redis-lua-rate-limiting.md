---
title: 用 Redis + Lua 实现分布式限流的四种方案对比
date: 2026-06-10 10:00:00
tags: [Redis, Lua, 分布式系统]
categories: [后端开发, 分布式系统]
description: 对比了固定窗口、滑动窗口、漏桶和令牌桶四种限流算法在 Redis + Lua 下的实现，附带各方案的 QPS 压测结果。
---

## 为什么需要分布式限流

单机限流在分布式部署下会失效——假设你有 10 个节点，每个节点限 100 QPS，实际总限流就是 1000 QPS，而且无法做到全局精确控制。分布式限流通过 Redis 做集中计数，解决这个问题。

## 四种方案

### 固定窗口

最简单的实现，按固定时间窗口（如 1 分钟）计数：

```lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local current = redis.call('INCR', key)
if current == 1 then
    redis.call('EXPIRE', key, window)
end
return current <= limit and 1 or 0
```

**优点**：实现简单，性能高。
**缺点**：窗口边界处可能出现 2 倍突发流量。

### 滑动窗口

通过 Redis 的 ZSET 实现更精确的滑动窗口：

```lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
local count = redis.call('ZCARD', key)
if count < limit then
    redis.call('ZADD', key, now, now .. math.random())
    redis.call('EXPIRE', key, window / 1000)
    return 1
end
return 0
```

### 漏桶 & 令牌桶

漏桶平滑输出，令牌桶允许一定突发。两者都用 Redis 存储令牌/水位状态，通过 Lua 脚本原子操作。

## 压测对比

| 方案 | 精度 | 突发控制 | 实现复杂度 | 适用场景 |
|------|------|----------|-----------|---------|
| 固定窗口 | 低 | 差 | 简单 | 非关键接口 |
| 滑动窗口 | 高 | 好 | 中等 | API 网关 |
| 漏桶 | 最高 | 最好 | 较复杂 | 下游保护 |
| 令牌桶 | 高 | 允许突发 | 较复杂 | 用户体验优先 |

## 我的选择

生产环境中我用了**滑动窗口**作为 API 网关的默认限流策略，对下游数据库服务则额外加了**令牌桶**保护。两者组合既保证了精确性，又防止了突发流量击穿下游。
