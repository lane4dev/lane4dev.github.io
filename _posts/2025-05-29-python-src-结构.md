---
layout: post
title: "为什么需要在 Python 项目里坚持用 src 目录结构"
date: 2025-05-11 00:00:00 +0800
categories: Code
tags: ["Python", "FastAPI"]
comments: true
---

做 Python 项目久了，你大概率经历过这些场景：

本地跑测试一切正常，打包装到干净环境里就各种 `ImportError`；
重构包名之后，仓库根目录的某个同名文件悄悄“抢”了导入；
Docker 容器里 `uvicorn` 启不起来，本地却完全没问题。

很多时候，罪魁祸首不是 Python 本身，而是项目结构，尤其是把包直接丢在仓库根目录的 flat 布局。
社区这几年越来越推荐另一种写法：**`src` 布局**。
PyPA 的打包指南、py-pkgs 等教程都明确倾向 `src` 布局，认为对新项目更稳。

这篇笔记就当是我给自己写的一份“`src` 布局复习指南”：什么是 `src`，它解决了什么问题，什么场景值得用。


## 对齐概念：flat vs src 布局

**flat 布局** 大概长这样：

```text
myproject/
├── mypackage/
│   ├── __init__.py
│   └── ...
├── tests/
├── pyproject.toml
└── README.md
```

包目录 `mypackage/` 就躺在项目根目录。Python 启动时会把当前工作目录加进 `sys.path`，所以在项目根跑：

```bash
pytest
python -m mypackage.something
```

即便你从来没 `pip install .`，`import mypackage` 也会成功，因为解释器直接从当前目录找到这个包。

**`src` 布局** 只多了一层：

```text
myproject/
├── src/
│   └── mypackage/
│       ├── __init__.py
│       └── ...
├── tests/
├── pyproject.toml
└── README.md
```

这一次，项目根目录里只放元数据和杂物，真正要 `import` 的包藏在 `src/` 下面。
要想 `import mypackage` 成功，通常需要你执行：

```bash
pip install -e .
# 或在 CI 中
pip install .
```

PyPA 的文档里把这两个分别叫 **flat layout** 和 **src layout**，
并明确说明：**`src` 布局会强迫你“先安装再跑”，从而避免无意中引用了仓库里的在开发版本。**


## src 布局到底解决了什么坑？

### 避免“当前目录阴影”问题

在 flat 布局下，只要你在项目根运行，当前目录就会在 `sys.path` 最前面。结果就是：

即便打包配置写错了、`pyproject.toml` 漏了包，**本地照样能 import**。

某个模块恰好跟你安装的第三方库同名，很可能优先导入到你本地那份。

CI 用另一套路径结构时，这些问题会晚很多才暴雷。

而 `src` 布局恰好反过来：项目根目录上面没有“可以直接 import 的包”，除非你手动把 `src` 加到 `PYTHONPATH`，否则：

**不安装，就不能 import。**

听上去麻烦半步，但代价换来的是：
**测试环境 ≈ 实际安装环境，很多打包和导入的错误能在本地、在 CI 上更早暴露。**

James Bennett、Hynek 等人最近几年都在各种文章和演讲里强调这一点，
基本已经形成共识：**现在更推荐默认使用 `src` 布局。**


### 让“安装之后跑测试”变成自然行为

`src` 布局的一个“副作用”是：
你被迫在开发流程里加上 `pip install -e .` 这一步。

以前很多项目是这样：

```bash
# 在项目根
pytest   # 直接跑
```

在 `src` 布局下，更推荐：

```bash
pip install -e .

pytest
```

听起来只是多一行命令，但它带来几个好处：

1. 测试用的包和用户安装的包结构一致（都来自打包结果）。
2. 可以顺带验证入口脚本、`[project.scripts]` 等配置是否正确。
3. 改 metadata 的时候有一个很自然的“重新安装”提醒。

PEP 660 把 editable 安装的行为标准化之后，`pip install -e .` 不再是 setuptools 的“私房菜”，
而是各家构建后端都要支持的能力：你改 `src/` 里的代码，**不需要重装**；
只有改依赖、入口脚本、包结构这些“元数据”时才需要重新 `pip install -e .`。


### 物理隔离更清爽

一个稍微成型的项目，顶层通常会堆很多东西：

- `tests/`、`docs/`、`scripts/`、`infra/`、`docker/`……
- Jupyter 笔记本、一次性脚本、临时工具……

flat 布局里，包目录就夹在这些东西中间，久而久之会变成“大杂烩”。

`src` 布局简单粗暴：**所有可发布的源码，一律塞进 `src/`**。
像 setuptools 文档里写的一样：项目根有个 `src` 目录，
其中放所有要分发的模块和包，顶层的其他目录都不会误入 wheel。

好处是：

1. 一眼就能看出“真正打包出去的代码”有哪些。
2. 把“工程资产”（脚本、infra、工具）和“库源码”物理隔离开。
3. 将来多包、monorepo 时，直接在 `src/` 下面再加子包即可。


### 命名空间更自然

当一个仓库开始承载多个服务或库时，`src` 布局会比 flat 布局清爽得多。例如：

```text
src/
├── app_api/         # FastAPI 后台入口
├── domain/          # 领域模型
└── common_utils/    # 公共工具库
```

在 `pyproject.toml` 里配置 `where = ["src"]`，
或者在 Poetry 里把这些包都列出来，就可以在代码中直接：

```python
from app_api.main import app
from domain.models import Project
from common_utils.logging import get_logger
```

如果你更想暴露一个统一命名空间，也可以：

```text
src/
└── app_backend/
    ├── api/
    ├── domain/
    └── common/
```

外部只看到 `app_backend.*`，内部怎么拆包，你自己说了算。


## src 布局在后台应用在 Docker 里的落地

很多人会觉得：`src` 是给“要发 PyPI 的库”用的，后台应用就没必要。
但现在不少人（包括 Hynek）更激进一点：**哪怕是纯应用，也建议当成“正常包”来写，然后用 `src` 布局。**

举个 FastAPI + Docker 的例子：

```text
app-backend/
├── pyproject.toml
├── src/
│   └── app_backend/
│       ├── main.py      # app = FastAPI()
│       └── ...
└── tests/
```

Dockerfile 里：

```dockerfile
WORKDIR /app
COPY pyproject.toml ./
COPY src ./src
RUN pip install --no-cache-dir .

CMD ["uvicorn", "app_backend.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

这样做有几个直接收益：

1. 容器里跑的是“已安装的包”，而不是靠 `/app/src` 暴露到 `PYTHONPATH` 的裸目录。
2. 本地、CI、Docker 三套环境里的导入逻辑是统一的。
3. 将来要把 `app_backend` 抽出一部分发布成内部库，也是一条路。


## 配置 & 导入的一些实践细节

结合 `src` 布局，几个需要记住的点：

1. **不要把 `src` 当包名用。**
   也就是说尽量不要写 `from src.xxx import xxx`，而是让 `src` 做一个纯粹的“容器目录”，
   真正的包名是 `mypackage` 或 `app_backend`。这样将来如果你想把项目挪到别的结构，不会整仓库的 import 全军覆没。

2. **打包工具要知道你的源码在 `src` 里。**

   - setuptools: `[tool.setuptools.packages.find] where = ["src"]`。
   - Poetry: 在 `[tool.poetry.packages]` 里列 `include = "mypackage", from = "src"` 等。

3. **开发时用 editable 安装。**

   - 本地：`pip install -e .` 然后再跑 `pytest` / `uvicorn`。
   - 修改 `src/...` 下的 Python 文件，无需重装就能生效。
   - 只有当你改动依赖、入口脚本、版本号等 metadata 时，才需要重新 `pip install -e .`。


## 哪些场景可以先不用 src？

话说回来，也没必要把 `src` 当成政治正确。有几个场景 flat 布局反而更轻松：

- 一次性的脚本、小工具仓库，只会在自己机器或几台服务器上跑；
- 没有打包需求，不打算让别人 `pip install`；
- 更像是“应用目录”，没有清晰的包边界。

这种时候，更重要的是做好虚拟环境和依赖管理，项目结构就别太折腾了。


## 结语：src 布局是给未来的自己留条后路

对于需要：

- 发布成库、复用到多个服务；
- 在 CI/CD、Docker、云环境里反复部署；
- 有多包、monorepo、插件式架构打算的项目，
  `src` 布局带来的那点“多一层目录”的麻烦，和它帮你挡掉的坑比起来，性价比非常高。

把包藏在 `src/` 下面，本质上是在强迫你用更接近“真实用户”的方式来开发和测试。
这种自带“护栏”的项目结构，大概率会在你几个月后、或者项目迭代几轮之后，让你感谢当初那个多写了 `/src` 的自己。
