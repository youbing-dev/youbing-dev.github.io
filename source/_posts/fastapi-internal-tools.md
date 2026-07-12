---
title: Python 脚本自动化：我用 FastAPI 搭建了一个内部工具平台
date: 2026-04-22 10:00:00
tags: [Python, FastAPI, 自动化]
categories: [后端开发, Python]
description: 作为后端 Java 开发者，我用 Python + FastAPI 快速搭建了一个内部运维工具平台，包含日志分析、告警管理和数据导出功能。
---

## 起因

团队里散落着各种 Python 运维脚本：日志分析的、告警处理的、数据导出的。每个人本地一份，版本不同，经常出错。我想把这些脚本统一收拢到一个 Web 平台上，让大家通过浏览器就能操作。

## 为什么选 FastAPI

作为 Java 开发者，我选 Python 框架的标准是：类型提示友好、自动生成文档、学习曲线平缓。FastAPI 完美满足这三点——它基于 Pydantic 做数据校验，自动生成 Swagger UI，而且代码风格接近 Spring Boot，Java 开发者上手很快。

## 平台架构

```
内部工具平台
├── /api/logs      → 日志分析（支持按时间/关键词/级别过滤）
├── /api/alerts    → 告警管理（创建/确认/静默规则）
├── /api/export    → 数据导出（MySQL → CSV/Excel）
└── /api/tools     → 通用工具（JSON 格式化、时间戳转换等）
```

## 核心代码示例

```python
from fastapi import FastAPI, Query
from pydantic import BaseModel

app = FastAPI(title="内部工具平台")

class LogQuery(BaseModel):
    keyword: str
    level: str = "ERROR"
    start_time: str
    end_time: str

@app.post("/api/logs/search")
async def search_logs(query: LogQuery):
    # 调用底层日志分析模块
    results = log_analyzer.search(
        keyword=query.keyword,
        level=query.level,
        time_range=(query.start_time, query.end_time)
    )
    return {"count": len(results), "logs": results}
```

## 部署

用 Docker 打包，docker-compose 一键启动，配合 Nginx 反向代理，半天就上线了。团队成员反馈非常好用，以前需要 SSH 到服务器上跑脚本，现在浏览器里点点就行。

## 心得

作为 Java 开发者写 Python 项目，最大的感受是"快"——从想法到上线的时间短了许多。FastAPI 的类型提示和自动文档让我不需要额外写接口文档，Pydantic 的数据校验帮我减少了很多防御性代码。对于内部工具这种"够用就好"的场景，Python + FastAPI 是很好的选择。
