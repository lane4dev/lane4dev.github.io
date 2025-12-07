---
layout: post
title: "Gunicorn 和 Uvicorn 的适用场景选择"
date: 2023-03-11 00:00:00 +0800
categories: Code
tags: ["Python", "Server"]
comments: true
---

如果你手头有个 FastAPI/Starlette 应用，要把它丢到线上跑，常见做法有两条：

- **Gunicorn + UvicornWorker**
- **纯 Uvicorn（多 worker）**

先把开发环境里那句最朴素的命令拎出来问一句：
为什么不直接 `uvicorn main:app` 就完事了？

原因也不复杂：**这个默认形式只有一个 worker**，等于是单进程跑在一个核上，多核 CPU 根本吃不满。
Uvicorn 当然可以加 `--workers` 开多进程，但在“多进程跑生产流量”这个前提下，就会回到一个更具体的问题：

> 用 Gunicorn 管理 Uvicorn worker，和用 Uvicorn 自己的多 worker，有什么区别？

下面就从多进程的角度，把这两种方案摊开聊一聊。


## 进程管理：托管 vs 自理

- **Gunicorn + UvicornWorker**

  - **托管式进程管理**：Gunicorn 主进程干活儿前还先 fork 出 N 个子进程，它能监控、自动重启宕掉的 worker。
  - **好处**：稳定可靠，信号处理和平滑重载都有现成方案。
  - **小 Tips**：用 `-w 4` 或 `--workers 4` 指定 worker 数量，比如：

  ```bash
  gunicorn --bind 0.0.0.0:8000 \
    -k uvicorn.workers.UvicornWorker \
    -w 4 \
    src.main:app
  ```

  - **场景**：高并发、流量不太好预测、想要管得死死的生产环境。

- **纯 Uvicorn（`--workers`）**

  - **多进程**：`uvicorn main:app --host 0.0.0.0 --port 8080 --workers 4`，简单粗暴。
  - **好处**：部署命令少、启动快，开发测试和小规模服务够用。
  - **注意**：它对 worker 的管理比较轻量，不会自动重启崩掉的进程，监控、平滑重载要你自己搞。


## 性能对比：几乎拉不开太大差距

- **单纯测速**：Uvicorn 内置 uvloop+httptools，单核性能跑起来稍微快点。
- **高并发**：多进程后，两者性能都差不多，瓶颈往往在数据库或者外部 API。


## 怎么选？

1. **想要稳定、可控** → Gunicorn + UvicornWorker

   > 一般在生产环境里推荐这么干，省得一两个 worker 宕了没人管。

2. **想要简单、快速上线** → 纯 Uvicorn 多 worker

   > 小团队、小项目，或者暂时没流量压力，用它最省事。
