---
title: 'PostgreSQL 慢查询优化：从 12s 到 50ms 的排查过程'
date: 2026-05-18 10:00:00
tags: [PostgreSQL, 性能优化, SQL]
categories: [后端开发, 数据库]
description: 一次生产环境的慢查询排查记录，涉及 EXPLAIN ANALYZE 分析、索引策略调整和查询重写，最终性能提升 240 倍。
---

## 问题现象

线上收到告警：订单统计报表接口响应时间超过 12 秒。这个接口每天被运营团队使用，12 秒的等待严重影响了工作效率。

## 排查过程

### 第一步：EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT o.id, o.order_no, o.amount, u.name
FROM orders o
LEFT JOIN users u ON o.user_id = u.id
WHERE o.created_at >= '2026-01-01'
ORDER BY o.created_at DESC
LIMIT 50;
```

结果发现 `Seq Scan on orders`，全表扫描了 800 万行。

### 第二步：检查索引

`created_at` 列上居然没有索引！之前的 DBA 离职后，这个索引从未被创建。

```sql
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
```

加索引后从 12s 降到 800ms，但还是不够快。

### 第三步：查询重写

原来的 LEFT JOIN 其实不需要——报表只需要有用户信息的订单。改成 INNER JOIN 并添加覆盖索引：

```sql
CREATE INDEX idx_orders_created_at_covering
ON orders(created_at DESC)
INCLUDE (order_no, amount, user_id);
```

### 最终结果

优化后查询稳定在 45-55ms，提升约 240 倍。

## 经验总结

1. 新表上线时一定要评估查询模式，提前创建索引
2. `EXPLAIN ANALYZE` 是排查慢查询的第一工具
3. 覆盖索引（INCLUDE）可以避免回表，效果显著
4. LEFT JOIN 和 INNER JOIN 的执行计划差异巨大，按需选择
