---
layout: post
title: "关于如何在 Flask 中配置 Celery"
date: 2019-01-03 21:14:00 +0800
categories: Code
tags: [Python, Flask]
comments: true
---

当服务器要处理 **很耗时** 或者 **需要定期执行** 的任务时，比如：

- 解析一份体积不小的上传文件，
- 给一大批用户发邮件，
- 生成报表、导出数据……

我们通常不会在同一个 HTTP 请求里 **傻等** 所有步骤都跑完再把结果丢给用户。
更常见的做法是：**请求先快速返回给用户**，而真正“费时间的活”被丢进一个队列，由后台的另一个进程慢慢处理。

这时候，就需要 Celery 就登场了。

Celery 是一个功能很全的 **异步任务队列框架**：

- 支持实时执行任务，也支持定时调度；
- 可以很简单地跑在单机上，也能扩展到多进程、多机器；
- 支持同步调用结果，也支持完全异步地扔一个任务就不管。

代价是：它需要依赖像 **RabbitMQ、Redis** 这样的消息中间件，所以初次搭建会稍微麻烦一点。下面就以 **Flask** 为例，做一个小示例。

---

## 安装

使用 `pip` 安装 Celery 非常直接：

```shell
pip install celery
```

注意确认你的环境里已经有可用的 Redis（或 RabbitMQ），之后会用到。

---

## 配置：在 Flask 中集成 Celery

在 Flask 的官方文档中，有一段经典的 Celery 集成示例，可以作为模板来用（[Flask 官方文档](http://flask.pocoo.org/docs/1.0/patterns/celery/#configure)）：

```python
from celery import Celery

def make_celery(app):
    celery = Celery(
        app.import_name,
        backend=app.config['CELERY_RESULT_BACKEND'],
        broker=app.config['CELERY_BROKER_URL'],
    )
    celery.conf.update(app.config)

    class ContextTask(celery.Task):
        def __call__(self, *args, **kwargs):
            # 在 Flask 的应用上下文中执行任务
            with app.app_context():
                return self.run(*args, **kwargs)

    celery.Task = ContextTask
    return celery
```

这段代码做了几件事：

1. 从 Flask 的配置里拿到消息中间件和结果存储的地址，创建了一个 `Celery` 实例。
2. 用 `app.config` 更新了 Celery 的配置，避免重复配置同样的内容。
3. 自定义了一个 `ContextTask`，确保每次执行任务时，都会自动进入 Flask 的应用上下文，这样在任务里也能正常访问数据库、配置等资源。

最终返回的就是一个和当前 Flask 应用“绑在一起”的 `celery` 对象。

---

## 示例：一个简单的“长任务”接口

下面我们写一个非常简化的 Flask 应用，模拟一个需要跑很久的任务，并提供**查询任务状态**的接口。

```python
import time
from flask import Flask, jsonify
from your_module import make_celery  # 假设上面的函数写在这里

app = Flask(__name__)
app.config['CELERY_BROKER_URL'] = 'redis://localhost:6379'
app.config['CELERY_RESULT_BACKEND'] = 'redis://localhost:6379'

celery = make_celery(app)

@app.route('/longtask', methods=['POST'])
def longtask():
    # 异步触发任务，立即返回 task_id
    task = long_task.apply_async()
    return jsonify({'task_id': task.id})

@app.route('/status/<task_id>')
def taskstatus(task_id):
    task = long_task.AsyncResult(task_id)

    if task.state == 'PENDING':
        # 任务刚创建，还没开始真正跑
        response = {
            'state': task.state,
            'current': 0,
            'total': 100,
        }
    elif task.state != 'FAILURE':
        # 任务在进行中或已完成，info 里存了进度
        response = {
            'state': task.state,
            'current': task.info.get('current', 0),
            'total': task.info.get('total', 100),
        }
    else:
        # 任务失败，这里只简单返回一个“失败完成”的状态
        response = {
            'state': task.state,
            'current': 100,
            'total': 100,
        }

    return jsonify(response)

@celery.task(name='app.long_task', bind=True)
def long_task(self):
    for i in range(100):
        # 持续更新任务进度
        self.update_state(state='PROGRESS', meta={'current': i, 'total': 100})
        time.sleep(1)

    return {'current': 100, 'total': 100}
```

这个例子展示了一个典型用法：

- `/longtask`：创建一个“长任务”，立刻返回 `task_id`，不阻塞请求。
- `/status/<task_id>`：根据 `task_id` 查询当前任务的状态和进度。
- `long_task`：在后台慢慢跑的任务，通过 `update_state` 持续更新进度。

实际项目里，把 `time.sleep(1)` 换成真正耗时的操作，比如处理上传文件、调用第三方接口、生成报告等。
对用户来说，体验就是：**提交请求瞬间就有反馈**，而“实际繁琐的工作”静悄悄在后台完成。
