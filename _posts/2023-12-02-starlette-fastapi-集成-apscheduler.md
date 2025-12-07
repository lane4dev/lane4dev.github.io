---
layout: post
title: "在 Starlette / FastAPI 集成 APScheduler"
date: 2023-12-02 00:00:00 +0800
categories: Code
tags: ["Python", "APScheduler"]
comments: true
---

在 Web 服务里跑一些**定时任务**（比如每分钟拉一次数据、定时清理缓存），又不想单独开一个脚本，
就可以把 APScheduler 嵌进 Starlette / FastAPI 里，随应用一起启动和停止。


## 核心思路

1. 用 **AsyncIOScheduler** 管理任务（和 Starlette/FastAPI 的 asyncio 世界兼容）。
2. 在应用的 **startup 事件** 中创建并启动 scheduler。
3. 用 **CronTrigger / IntervalTrigger** 之类添加任务。
4. 进程退出时，记得关掉 scheduler（可选但推荐）。


## Starlette 示例（每分钟执行一次）

```python
from apscheduler.triggers.cron import CronTrigger
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from starlette.applications import Starlette

async def cron_job():
    print("task is running")

async def startup_event():
    scheduler = AsyncIOScheduler()
    # 每分钟执行一次
    scheduler.add_job(cron_job, CronTrigger.from_crontab("* * * * *"))
    scheduler.start()

app = Starlette(on_startup=[startup_event])
```

### 关键点

- `AsyncIOScheduler()`：基于当前事件循环工作，适合 Starlette/FastAPI。
- `CronTrigger.from_crontab("* * * * *")`：用 crontab 表达式定义调度规则。
- `on_startup=[startup_event]`：应用启动时，自动创建并启动 scheduler。


## FastAPI 版本（lifespan 的方式）

记住关键词：`FastAPI lifespan AsyncIOScheduler`

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger

async def cron_job():
    print("task is running")

@asynccontextmanager
async def lifespan(app: FastAPI):
    scheduler = AsyncIOScheduler()
    scheduler.add_job(cron_job, CronTrigger.from_crontab("* * * * *"))
    scheduler.start()
    try:
        yield
    finally:
        scheduler.shutdown()

app = FastAPI(lifespan=lifespan)
```


## 常用触发器速记

- **CronTrigger**：适合“每天 / 每周 / 每月 XX 点”的任务

  - 例：`CronTrigger.from_crontab("0 3 * * *")` → 每天 3:00

- **IntervalTrigger**：适合“每隔 N 秒 / 分钟 / 小时”

  - 例：`scheduler.add_job(job, "interval", minutes=5)`


## 备注

- **任务函数尽量写成 async**，直接用 `async def`，避免在事件循环里到处套线程。

- 日志/print 看不到时，检查：

  - 应用是不是以守护进程方式跑；
  - 输出是否被重定向；
  - APScheduler 的日志级别是否太高。

- 如果有多个模块需要共享 scheduler，可以：

  - 在 `lifespan` / `startup` 里把 scheduler 挂到 `app.state.scheduler` 上，其他地方 `request.app.state.scheduler` 拿来用。
