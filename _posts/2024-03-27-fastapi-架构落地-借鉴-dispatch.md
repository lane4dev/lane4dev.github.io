---
layout: post
title: "再谈 FastAPI 架构落地：从 Dispatch 学来的几件事"
date: 2024-03-27 00:00:00 +0800
categories: Code
tags: ["Python", "FastAPI"]
comments: true
---

做 FastAPI 项目，我发现自己有时候会迷恋类似 **Clean Architecture / Hexagonal**，好像选了这么一个“万金油”的架构就能解决一切问题，但真到落地时，目录还是一团乱：`main.py` 越写越胖、业务逻辑塞进 router、运维动作全靠口口相传。

后来仔细拆了 Netflix 的 Dispatch，才发现人家厉害的不在于挂了多少“架构名词”，而是一些非常落地的选择：按领域分组代码，让目录本身就是语义边界；用 API-First 把前后端、插件和集成都串起来；用统一的 CLI 管开发和迁移，减少“口口相传”的运维黑魔法；再用 插件机制 把 Slack、Jira 这一类能力长在“边上”而不是写死在核心域里。

这篇《再谈 FastAPI 架构落地：从 Dispatch 学来的几件事》想做的，就是把这些做法抽成一套可以直接复用的骨架：目录怎么按领域拆，核心设施放哪儿，生命周期和数据库迁移怎么收口，插件和 CLI 怎么接进来，方便自己在之后的项目里，也能快速搭出一个“小而清晰的 Dispatch 式” FastAPI 后端。


## Dispatch 的关键特点

- **按领域分组（Domain-Oriented）**：目录即语义边界，同一目录内聚 `models/schemas/service/router`。这同时也是 [fastapi-best-practices](https://github.com/zhanymkanov/fastapi-best-practices) 里提到应该借鉴 Dispatch 的地方。
- **API-First**：优先定义 OpenAPI，再写实现，便于前后端并行、也利于插件与集成。Dispatch 文档就把 “API First” 标得很醒目。
- **CLI 驱动**：开发、迁移、插件操作走统一命令，降低“口口相传”的运维风险。Dispatch 的安装与开发流程同样依赖统一脚本/镜像。
- **插件化扩展**：通过钩子/入口点（entry points）发现并加载功能，不污染核心域模型；这是 Dispatch 成熟的一条路径。

## 目录骨架

下面这份目录结构是参考了 Dispatch 和 [fastapi-best-practices](https://github.com/zhanymkanov/fastapi-best-practices)
之后整理出来的，目标很简单：让 **领域代码** 和 **基础设施** 一眼就能分得开。

```
fastapi-app/
├─ alembic/                    # 迁移脚本
├─ alembic.ini
├─ pyproject.toml
├─ README.md
├─ .env.example
├─ src/
│  ├─ app/
│  │  ├─ core/                 # 基础设施（配置/db/日志/中间件）
│  │  │  ├─ config.py
│  │  │  ├─ db.py
│  │  │  ├─ logging.py
│  │  │  └─ lifespan.py
│  │  ├─ common/               # 通用异常/工具/响应模型
│  │  ├─ auth/                 # 领域：认证（示例）
│  │  │  ├─ models.py
│  │  │  ├─ schemas.py
│  │  │  ├─ service.py
│  │  │  └─ router.py
│  │  ├─ incident/             # 领域：事件（示例，向 Dispatch 致意）
│  │  │  ├─ models.py
│  │  │  ├─ schemas.py
│  │  │  ├─ service.py
│  │  │  └─ router.py
│  │  ├─ plugins/              # 插件宿主与协议
│  │  │  ├─ base.py
│  │  │  └─ loader.py
│  │  └─ main.py               # FastAPI app/路由聚合/CLI 入口
│  └─ cli.py                   # Typer/Click：dev/db/plug 子命令
└─ tests/
   ├─ conftest.py
   ├─ test_auth.py
   └─ test_incident.py
```

可以看到：

- `core/common`：放的是“任何一个项目都可能复用”的基础设施。
- 每个领域（`auth`、`incident`…）自成一格：该领域的 `models/schemas/service/router` 全部内聚在一起。
- `plugins` 单独成层，只跟“宿主协议”打交道。
- 项目入口非常克制：`main.py` 聚合路由，`cli.py` 聚合命令。


## 配置与生命周期

配置层的目标是：环境变量有统一入口，应用生命周期有统一收口。这里用到了 `Pydantic Settings` 和 FastAPI 的 `lifespan` 机制。

```python
# src/app/core/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    APP_NAME: str = "fastapi-app"
    DATABASE_URL: str
    LOG_LEVEL: str = "INFO"

    model_config = {"env_file": ".env"}

settings = Settings()
```

```python
# src/app/core/db.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine(settings.DATABASE_URL, pool_pre_ping=True, future=True)
SessionLocal = sessionmaker(bind=engine, autocommit=False, autoflush=False, future=True)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

```python
# src/app/core/lifespan.py
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # 这里可以放连接检查、插件加载等
    yield
    # 这里做回收与清理
```

```python
# src/app/main.py
from fastapi import FastAPI
from app.core.lifespan import lifespan
from app.auth.router import router as auth_router
from app.incident.router import router as incident_router

app = FastAPI(lifespan=lifespan, title="fastapi-app")
app.include_router(auth_router, prefix="/api/auth", tags=["auth"])
app.include_router(incident_router, prefix="/api/incidents", tags=["incidents"])
```

这么做有两个直接好处：

- 语义更集中：启动 / 关闭逻辑集中在 `lifespan`，不会散落在各个模块。
- 测试更顺滑：用 TestClient 时可以完整触发生命周期，不会出现“本地跑得好好的，CI 一执行就崩”的情况。


## 数据层与迁移

在数据层上，这里采用的是 **SQLAlchemy 2.0 + Alembic 自动迁移** 的组合。
核心经验就一句话，所有要迁移的模型，Alembic 必须**看得见**。

```python
# src/app/incident/models.py
from sqlalchemy import String, Integer, func
from sqlalchemy.orm import Mapped, mapped_column
from app.core.db_base import Base  # 你的 Base
from datetime import datetime

class Incident(Base):
    __tablename__ = "incidents"
    id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
    title: Mapped[str] = mapped_column(String(200), nullable=False)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

```bash
# 初始化与迁移示例
alembic init alembic
# 编辑 alembic/env.py：导入 Base.metadata
alembic revision -m "init incidents" --autogenerate
alembic upgrade head
```

**踩坑提醒**

- 忘了导入模型 → Alembic 自然“看不见”。Dispatch 文档强调“把模型 import 到统一模块里供 Alembic introspection”。
- 生产库改大表结构的过程中，避免“结构 + 重度数据迁移”同一条 revision，先改结构、再小步数据迁移。


## 领域模块规范

每个领域目录下，保持一个固定的小分层：`schemas` 负责输入/输出，`service` 负责业务操作，`router` 负责协议与装配。

```python
# src/app/incident/schemas.py
from pydantic import BaseModel

class IncidentCreate(BaseModel):
    title: str

class IncidentRead(BaseModel):
    id: int
    title: str
```

```python
# src/app/incident/service.py
from sqlalchemy.orm import Session
from .models import Incident
from .schemas import IncidentCreate

def create_incident(db: Session, data: IncidentCreate) -> Incident:
    obj = Incident(title=data.title)
    db.add(obj)
    db.commit()
    db.refresh(obj)
    return obj
```

```python
# src/app/incident/router.py
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from app.core.db import get_db
from .schemas import IncidentCreate, IncidentRead
from .service import create_incident

router = APIRouter()

@router.post("/", response_model=IncidentRead)
def create(data: IncidentCreate, db: Session = Depends(get_db)):
    return create_incident(db, data)
```

这一套 `models/schemas/service/router` 的内聚分层，
是我看到包括 [fastapi-best-practices](https://github.com/zhanymkanov/fastapi-best-practices) 从 Dispatch 身上学来的核心模式：
一个目录讲清楚一个领域，不把逻辑拆散到全项目各个角落。


## 插件化

Dispatch 支持一大堆外部集成（Slack、Jira、G Suite、Zoom…），背后的思路其实很朴素，
宿主只负责发事件，插件自己决定怎么处理。在这个示例里我们用 Python 的 `entry points` 做一个极简版。

```python
# src/app/plugins/base.py
from typing import Protocol

class IncidentHooks(Protocol):
    def on_incident_created(self, title: str) -> None: ...
```

```python
# src/app/plugins/loader.py
from importlib.metadata import entry_points
from typing import List
from .base import IncidentHooks

def load_incident_plugins() -> List[IncidentHooks]:
    eps = entry_points(group="app.incident_plugins")
    return [ep.load()() for ep in eps]
```

```toml
# pyproject.toml
[project.entry-points."app.incident_plugins"]
example = "example_plugin:ExamplePlugin"
```

```python
# example_plugin.py
class ExamplePlugin:
    def on_incident_created(self, title: str) -> None:
        print(f"[plugin] created: {title}")
```

上面这一段只是定义了“插件长什么样”以及“如何被发现并加载”。真正关键的是：

核心业务代码 **在哪里**、**如何** 调用这些插件？

下面是在 `service.create_incident` 里调用插件钩子的一个完整示例：

```python
# src/app/incident/service.py
from typing import List
from sqlalchemy.orm import Session

from .models import Incident
from .schemas import IncidentCreate
from app.plugins.base import IncidentHooks
from app.plugins.loader import load_incident_plugins

_plugins_cache: List[IncidentHooks] | None = None

def get_incident_plugins() -> List[IncidentHooks]:
    """懒加载 + 简单缓存，避免每次请求都重新扫描 entry points。"""
    global _plugins_cache
    if _plugins_cache is None:
        _plugins_cache = load_incident_plugins()
    return _plugins_cache


def create_incident(db: Session, data: IncidentCreate) -> Incident:
    # 正常创建事件
    obj = Incident(title=data.title)
    db.add(obj)
    db.commit()
    db.refresh(obj)

    # 在“事件已创建”之后，统一触发插件钩子
    for plugin in get_incident_plugins():
        try:
            plugin.on_incident_created(obj.title)
        except Exception as exc:
            # 这里简单打印，实际项目里建议打日志，不要让插件异常影响主流程
            print(f"[plugin error] {plugin!r} failed: {exc}")

    return obj
```

这样一来，
核心服务只负责“发一个 `incident` 已创建的事件”，完全不关心插件内部干什么，
每个插件都通过实现 `IncidentHooks` 接口，来决定 收到这个事件后要做什么：

- 有的去发一条 Slack 消息；
- 有的写一条审计日志；
- 有的推到监控系统。

在 `service.create_incident` 的最后调用这些插件钩子，就等于把“事件创建后该做什么”这件事，
从核心服务里剥离出去，交给插件体系来演化。新需求来了，加插件就好，不必进核心改代码。


## CLI

Dispatch 在安装、开发、运维上都有完整脚本和镜像，开发者只需要记住少数几个命令。
这一节做的是同一件事：把常用动作收敛到一个 CLI 里。

```python
# src/cli.py
import typer, uvicorn, os
from alembic.config import Config
from alembic import command

app = typer.Typer()

@app.command()
def dev(host: str = "0.0.0.0", port: int = 8000):
    uvicorn.run("app.main:app", host=host, port=port, reload=True)

@app.command()
def db_upgrade():
    cfg = Config("alembic.ini")
    command.upgrade(cfg, "head")

if __name__ == "__main__":
    app()
```

常见的本地开发、数据库迁移、插件管理、数据初始化，
都可以往这个 CLI 里逐步沉淀。


## 测试与质量

- **生命周期测试**：用 TestClient 触发 `lifespan`，避免“本地好好的，CI 掉链子”的尴尬。
- **规范**：`black + flake8 + isort + pre-commit`，这与 Dispatch 文档里对代码一致性的要求相呼应。
- **数据库用例**：独立的测试数据库 + 事务回滚 fixture，覆盖成功/失败/并发/边界。


## API-First 的好处

Dispatch 在首页就强调 **API First**。做法很简单，
先写 OpenAPI（pydantic 模型/响应示例/错误码），再生成实现与文档。
这让前端、外部系统、插件都能**并行推进**。

也就是：

- 先定义 pydantic 模型、响应结构、错误码、示例；
- 让前端、外部系统、插件都根据这份规范并行推进；
- 后端实现则围绕这份规范去补齐细节，而不是边写边改接口。

这样一来，API 不再是代码写完顺便导出一下的副产品，而是整个系统协作的中心。
