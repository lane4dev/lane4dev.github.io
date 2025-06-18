---
layout: post
title: "Clean Architecture 与 Hexagonal 的差异与实践指南"
date: 2024-02-21 00:00:00 +0800
categories: Code
tags: ["Python", "FastAPI"]
comments: true
---

Clean Architecture 和 Hexagonal Architecture 常常被放在一起讨论，结果很多人看完文档还是面对同样的问题：
概念都懂了，可是实战里到底有什么差异？我这个项目应该按哪一套来拆？

如果你已经用 FastAPI 搭过一圈服务，现在开始认真思考 **目录如何分层**、**Use Case 怎么落位**、**API 层要不要再加 Controller**，
这篇文章会试着给出一套可操作的「对比与实践指引」。

## 先看共同点：它们都想保护的，是“业务内核”

无论是 Clean 还是 Hexagonal，本质上都在做一件事：
**把“真正值钱的业务规则”从框架、数据库和各种 SDK 里解救出来。**

换成工程师语言，就是这几条：

- **依赖只能指向内核**
  UI、数据库、HTTP 客户端、消息队列，都不能反向侵入业务内核。
  你的 Domain / Use Case 不应该 import FastAPI、SQLAlchemy、aiohttp。

- **业务逻辑要容易单测**
  不用起数据库、不用起 HTTP server，就能把核心逻辑跑一遍，这才叫保护**可维护性**。

- **技术细节是可拔插的**
  想从 SQLite 换成 Postgres？想从内部 HTTP 调用换成消息队列？理论上都应该只改 Adapter，而不用改业务。

在这一点上，Clean 和 Hexagonal 是完全一致的，只是画图的方式、层次的颗粒度不太一样。

---

## 差异：圈和边的审美不一样

### 图形隐喻：同心圆 vs 六边形

- **Clean Architecture**：
  经典那张图是几层同心圆：

  - Entities（实体/核心业务规则）
  - Use Cases（用例/应用服务）
  - Interface Adapters（接口适配）
  - Frameworks & Drivers（框架、UI、DB、外设）

- **Hexagonal Architecture**：
  不太画圈，而是画一个六边形，中间是 Domain（领域），周围每一条边都是一个 Port（端口），外面挂各种 Adapter（适配器），比如：

  - REST Adapter
  - CLI Adapter
  - DB Adapter
  - Message Adapter

直观一点的理解：

Hexagonal 关注“内核 ↔ 端口 ↔ 适配器”的对称通信；
Clean 在此基础上，又强调了“内核内部也要分层”，特别是用例层。

### 内核的颗粒度不同

- 在 **Hexagonal** 里，只要你把 Domain 和 I/O 隔开就行，内部可以简单理解成：

  - Domain Model + Domain Service
  - Port Interface（比如 Repository、外部服务接口）

- 在 **Clean** 里，内核被分得更细：

  - **Entities**：最稳定的业务规则（比如资金账户、订单状态机）
  - **Use Cases**：一条一条具体应用场景（创建订单、关闭订单、计算账单…）

所以粗暴一点理解：
**Clean = Hexagonal + 更细的内核分层（多了一圈 Use Case）**

---

## 在 FastAPI 项目里，大概会长成什么形状？

如果你用 Clean 的思路来设计 FastAPI 后台，大致会有这么几块：

- **Entities / Domain**
  纯 Python 类、dataclass、pydantic BaseModel（但不依赖框架），只管业务规则。

- **Use Cases**
  一条一条用例：`CreateUser`, `ResetPassword`, `CalculateBilling`…
  只依赖 Entities，不 import FastAPI、SQLAlchemy 等。

- **Interface Adapters**

  - Controller（或 API Handler）：负责把 HTTP 请求变成 Use Case 输入，把 Use Case 输出变成 HTTP 响应。
  - Repository 实现：用 SQLAlchemy 操作数据库，但对外暴露的是 Repository 接口。
  - Presenter / Mapper：把 Domain 对象转成对外暴露的 DTO。

- **Frameworks & Drivers**

  - FastAPI Router
  - SQLAlchemy 引擎、Session
  - aiohttp ClientSession
  - Celery、消息队列客户端等

Hexagonal 的图会简单一点：Domain 在中间，路由、Controller、Repository 实现、外部 HTTP 调用，统统看作各种 Adapter，只是方向不一样（驱动 vs 被驱动）。

---

## Interface Adapters 里，Controller 到底该不该单独一层？

这是项目里最实际的问题：
在 FastAPI 里，`api router` 和 Use Case 之间，要不要搞一个“Controller 层”？

答案是先看职责分工：

- **Router（框架层）**：
  把 URL、HTTP 方法、状态码、依赖注入这些东西交给 FastAPI 管，尽量只描述“这条路由长什么样”。

- **Controller（Interface Adapters 层）**：

  - 接收已经解析好的请求模型（Pydantic 模型、路径参数等）；
  - 调用对应 Use Case；
  - 处理异常、日志，决定返回什么响应模型。

- **Use Case（内核）**：

  - 完成业务流程：校验业务规则、操纵 Entity、调用 Port；
  - 不关心 HTTP、Pydantic 的细节。

基于这个划分，有三种常见实践：

### 方案 A：Router 直接当 Controller 用（小项目常见）

```python
@router.post("/users")
async def create_user(dto: UserCreate, svc: CreateUserUseCase = Depends(...)):
    user = await svc.execute(dto)
    return UserOut.from_domain(user)
```

- 优点：简洁，文件少，上手快。
- 缺点：路由函数很容易越来越胖，一旦逻辑复杂（包含异常映射、审计日志、额外查询），维护成本会迅速上来。

### 方案 B：显式定义 Controller 类（中大型项目推荐）

```python
# adapters/controllers/user_controller.py
class UserController:
    def __init__(self, create_user: CreateUserUseCase):
        self.create_user = create_user

    async def create(self, dto: UserCreate) -> UserOut:
        user = await self.create_user.execute(dto)
        return UserOut.from_domain(user)

# routers/user_router.py
@router.post("/users")
async def create_user(
    dto: UserCreate,
    controller: UserController = Depends(get_user_controller),
):
    return await controller.create(dto)
```

- 优点：

  - Router 变干净了，只负责 URL → 函数的映射；
  - Controller 专门负责“Interface Adapters 的活儿”：日志、异常转换、DTO 转换；
  - Controller 可以单独写单元测试，不需要起 Web 层。

- 缺点：

  - 文件、类会多一些，需要团队在风格上达成共识，否则容易“一半路由自己干，一半拉 Controller”。

### 方案 C：Use Case 直接接 Router（不推荐长期使用）

有的项目会直接让 Use Case 接收请求 DTO、返回响应 DTO，等于把“Interface Adapters”这层压扁：

```python
class CreateUserUseCase:
    async def execute(self, dto: UserCreate) -> UserOut:
        ...
```

看起来很简单，但有一个副作用：Use Case 和外部协议/DTO 就绑死了。逻辑一旦需要在 CLI、消息消费者里复用，会明显感觉不顺手。

---

## 怎么结合 Clean 和 Hexagonal 的思路落地？

如果已经按 Hexagonal 的方式在 FastAPI 里写了 Ports & Adapters，那其实只需要微调几步，就能对齐 Clean 的视角：

1. **保留 Domain + Port + Adapter 的结构不变**：
   这是 Hexagonal 的基石，没必要推翻。

2. **在 Domain 内部加一层 Use Case 服务**：
   把“业务流程”的那部分逻辑从 Domain Service / Controller 里剥离出来，单独放进 `use_cases` 或 `application` 子包。

3. **把 Controller 看成一种 Driving Adapter**：
   它站在 Interface Adapters / 外圈的位置，但依赖方向仍然是：

   - `Router (框架)` → `Controller (Adapter)` → `Use Case (内核)`。

4. **按复杂度决定 Controller 要不要显式存在**：

   - 简单 CRUD：Router 兼任 Controller。
   - 大量业务流程、统一错误处理、审计日志：单独 Controller 层，别犹豫。

---

## 小结

- **Clean 和 Hexagonal 是一家人**：
  一个强调“用例层”、一个强调“端口-适配器”，目标都是保护业务内核。

- **Interface Adapters 不等于可有可无**：
  它是你把 **技术世界** 和 **业务世界** 隔离开的缓冲带，而 Controller 就是里面非常关键的一块。

- **在 FastAPI 里要不要单独 Controller 层？**

  看规模和复杂度：

  - 小项目：Router 直接调 Use Case，注意别把逻辑写太肥。
  - 中大型项目：建议显式引入 Controller，把路由从业务细节中解耦出来。
