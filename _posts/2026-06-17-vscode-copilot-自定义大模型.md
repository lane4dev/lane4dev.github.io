---
layout: post
title: "VSCode Colpilot 自定义大模型"
date: 2026-06-17 10:20:00 +0800
categories: AI
tags: ["Agent", "VSCode", "Methodology"]
comments: true
---

VS Code 过去的 Copilot 使用体验有一个很明显的短板：模型基本由 GitHub Copilot 决定，开发者能选的范围有限。
这个边界从 VS Code 1.122 开始被明显放宽。

从该版本起，VS Code Stable 提供了 **Custom Endpoint** 能力。
它允许开发者把兼容 OpenAI Chat Completions、OpenAI Responses 或 Anthropic Messages 协议的模型端点接入 Chat，
并继续使用 VS Code 内置的工具调用、Agent、MCP 和文件编辑流程。

这件事的价值不在于 **可以换一个聊天模型**，而在于 Copilot 的工作流开始可以和模型供应商解耦。

你可以继续在 VS Code 中使用 Agent 模式、让模型读取代码、调用终端、执行 MCP 工具；
模型背后却可以是 DeepSeek、企业内部模型网关、本地 vLLM，或者经由 LiteLLM 统一转发的多个模型服务。

本文以 DeepSeek V4 为例，完整说明如何把一个 OpenAI 兼容模型接入 VS Code Copilot Chat。

{% include image.html
  src="assets/images/2026-06/17-001-vscode-custom-model-select.png"
  alt="VS Code Copilot Chat 的模型选择器"
  caption="VS Code Copilot Chat 的模型选择器" %}


## VS Code 1.122 之后到底新增了什么

VS Code 的自定义模型能力通常被称为 **BYOK（Bring Your Own Key）**，即 **使用自己的模型 API Key**。

它解决的是 Chat 和 Agent 工作流中的模型来源问题。

配置完成后，VS Code 会把请求直接发送到你指定的模型服务，而不是由 GitHub Copilot 代为转发。
模型可以来自公有云，也可以来自企业内部服务，甚至可以运行在本地机器上。

Custom Endpoint 支持三类接口协议：

| 协议类型               | 典型场景                                              |
| ------------------ | ------------------------------------------------- |
| `chat-completions` | OpenAI 兼容接口、DeepSeek、vLLM、LiteLLM、OneAPI、企业 AI 网关 |
| `responses`        | 新版 OpenAI Responses API 或兼容网关                     |
| `messages`         | Anthropic Messages API 或兼容服务                      |

DeepSeek 官方 API 同时兼容 OpenAI 和 Anthropic 风格接口。
对于 VS Code 的 Custom Endpoint 配置，最直接的选择是 OpenAI Chat Completions。

需要先澄清一个常见误解：这并不等于完全替换 GitHub Copilot。

自定义模型主要用于：

- Copilot Chat 对话；
- Agent 模式中的规划、代码修改和工具调用；
- MCP 工具调用；
- 文件编辑与终端任务；
- 部分后台 utility task，例如标题生成、提交信息生成和意图识别。

但它通常不能直接替代：

- 普通行内代码补全；
- Next Edit Suggestions；
- 依赖 GitHub Copilot 服务的语义搜索；
- 依赖 embeddings 的能力。

换句话说，BYOK 解决的是 **聊天与 Agent 用哪个模型** 的问题，不是把 VS Code 的所有 AI 功能全部迁移到第三方模型。


## 用 DeepSeek 作为示例

DeepSeek 是一个比较典型的接入对象，因为它同时满足几个条件：

1. 提供 OpenAI 兼容接口；
2. 支持工具调用；
3. 支持思考模式；
4. 支持较长上下文；
5. 支持 JSON 输出；
6. 能用于 Agent 类编码任务。

目前可以重点关注两个模型：

| 模型                  | 更适合的任务                               |
| ------------------- | ------------------------------------ |
| `deepseek-v4-flash` | 日常问答、代码解释、常规修复、提交信息、轻量级 Agent 任务     |
| `deepseek-v4-pro`   | 多文件重构、复杂业务逻辑、架构设计、长任务链路、复杂 Agent 工作流 |

Flash 的定位是高频、低延迟、低成本。Pro 更适合复杂推理和需要多轮工具调用的任务。

一个比较实用的组合是：

- 日常聊天、代码解释、提交信息：使用 Flash；
- 复杂重构、方案设计、跨模块修改：使用 Pro；
- Agent 模式默认优先使用 Pro。

不要一开始就把所有工作都扔给 Pro。很多任务根本不值得消耗高推理成本。模型选择本身也是工程设计的一部分。

{% include image.html
  src="assets/images/2026-06/17-002-deepseek-v4-pro-and-flash.png"
  alt="DeepSeek V4 Flash 与 V4 Pro 对比"
  caption="DeepSeek V4 Flash 与 V4 Pro 对比" %}


## 准备工作

开始前，确认下面几项。

### VS Code 版本

建议使用 VS Code 1.122 或更高版本。

可以通过命令面板执行：

```text
Help: About
```

或在终端中运行：

```bash
code --version
```

如果版本较低，先升级 VS Code。Custom Endpoint 在较早版本中可能仍属于预览能力，配置入口和字段也可能不同。

### DeepSeek API Key

需要先在 DeepSeek Platform 创建 API Key。

API Key 一般以 `sk-` 开头。它等同于模型服务的访问凭证，应按密码管理。

不要把 Key 写入：

- 项目仓库；
- `.vscode/settings.json`；
- 代码文件；
- 文档截图；
- 团队共享 Markdown；
- Git 提交记录。

本文中的 Key 都使用占位符表示。

### 明确 DeepSeek 的接口信息

本教程使用 OpenAI Chat Completions 格式。

基础地址：

```text
https://api.deepseek.com
```

完整请求地址：

```text
https://api.deepseek.com/chat/completions
```

模型 ID：

```text
deepseek-v4-flash
deepseek-v4-pro
```

注意这里的区别：

- `https://api.deepseek.com` 是 Base URL；
- `https://api.deepseek.com/chat/completions` 是 VS Code 配置中应填写的完整模型请求地址。

很多 404 问题，最后都只是因为把 Base URL 当成了完整 Endpoint。


## 通过 VS Code 界面创建 Custom Endpoint

先不要急着手改 JSON。建议先走一遍 VS Code 的图形化流程，让编辑器生成模型配置骨架。

### 打开 Language Models 管理界面

按下：

```text
Ctrl + Shift + P
```

输入并执行：

```text
Chat: Manage Language Models
```

也可以从 Copilot Chat 输入框附近的模型选择器中，点击管理模型的齿轮图标进入。


### 添加模型

在 Language Models 页面中选择：

```text
Add Models
```

然后选择：

```text
Custom Endpoint
```


### 填写基础信息

VS Code 会要求填写模型组信息。

建议填法如下：

| 字段           | 推荐值                                     |
| ------------ | --------------------------------------- |
| Group Name   | `DeepSeek`                              |
| Display Name | `DeepSeek V4 Pro` 或 `DeepSeek V4 Flash` |
| API Key      | 你的 DeepSeek API Key                     |
| API Type     | `Chat Completions`                      |

其中，Group Name 决定模型选择器里的分组名称。Display Name 决定具体模型展示给你的名字。

完成后，VS Code 会打开一个名为 `chatLanguageModels.json` 的配置文件。

这个文件是自定义模型的核心。


## 配置 chatLanguageModels

下面是一份可以同时配置 Flash 和 Pro 的示例。

```json
[
  {
    "name": "DeepSeek",
    "vendor": "customendpoint",
    "apiKey": "YOUR_DEEPSEEK_API_KEY",
    "apiType": "chat-completions",
    "models": [
      {
        "id": "deepseek-v4-flash",
        "name": "DeepSeek V4 Flash",
        "url": "https://api.deepseek.com/chat/completions",

        "toolCalling": true,
        "thinking": true,
        "streaming": true,

        "maxInputTokens": 616000,
        "maxOutputTokens": 384000,

        "supportsReasoningEffort": [
          "high",
          "max"
        ],
        "reasoningEffortFormat": "chat-completions"
      },
      {
        "id": "deepseek-v4-pro",
        "name": "DeepSeek V4 Pro",
        "url": "https://api.deepseek.com/chat/completions",

        "toolCalling": true,
        "thinking": true,
        "streaming": true,

        "maxInputTokens": 616000,
        "maxOutputTokens": 384000,

        "supportsReasoningEffort": [
          "high",
          "max"
        ],
        "reasoningEffortFormat": "chat-completions"
      }
    ]
  }
]
```

这段配置可以直接说明 VS Code 是如何理解一个第三方模型的。

### vendor

```json
"vendor": "customendpoint"
```

表示这是一个通过 Custom Endpoint 接入的模型供应商。

不要继续使用旧的 OpenAI Compatible Provider 配置方式，也不要再依赖：

```text
github.copilot.chat.customOAIModels
```

该设置已经属于旧方案。

### apiType

```json
"apiType": "chat-completions"
```

表示 VS Code 会使用 OpenAI Chat Completions 风格向模型发送请求。

DeepSeek 也提供 Anthropic 格式接口，但本文不使用该路径。对大多数 OpenAI 兼容网关而言，
`chat-completions` 是兼容性最好、排错成本最低的选项。

### id

```json
"id": "deepseek-v4-pro"
```

这是实际发送给 DeepSeek API 的模型标识。

它不是显示名称，不能随意修改。

### name

```json
"name": "DeepSeek V4 Pro"
```

这是 VS Code 模型选择器中的显示名称。

### url

```json
"url": "https://api.deepseek.com/chat/completions"
```

必须填写完整请求地址。

下面这种写法不正确：

```json
"url": "https://api.deepseek.com"
```

因为它只是 Base URL，没有指向具体请求接口。

### toolCalling

```json
"toolCalling": true
```

这是 Agent 模式中最重要的字段之一。

VS Code 只有在模型声明支持工具调用时，才会将其提供给 Agent 工作流。
没有这个字段，模型可能仍能用于普通 Chat，但不会出现在 Agent 可选模型中。

DeepSeek V4 支持 Tool Calls，因此应设置为 `true`。

### thinking

```json
"thinking": true
```

告诉 VS Code，这个模型支持思考能力。

DeepSeek V4 同时支持思考模式和非思考模式。默认情况下，思考模式是开启的。

### supportsReasoningEffort

```json
"supportsReasoningEffort": [
  "high",
  "max"
]
```

DeepSeek V4 支持 `high` 与 `max` 两档 reasoning effort。

配置后，VS Code 的模型选择器中会出现对应的推理强度选项。

通常可以这样使用：

| 推理强度   | 适合场景                      |
| ------ | ------------------------- |
| `high` | 普通代码生成、Bug 分析、一般 Agent 任务 |
| `max`  | 大型重构、复杂调试、跨模块设计、难以定位的逻辑问题 |

别把 `max` 当成默认选项。它不是 **更聪明** 的免费按钮，而是更多推理时间、更多 token 和更高成本。

### 上下文长度配置

DeepSeek V4 的上下文窗口为 1M，最大输出为 384K。

VS Code 会将：

```text
maxInputTokens + maxOutputTokens
```

视为模型总上下文窗口。

因此配置：

```json
"maxInputTokens": 616000,
"maxOutputTokens": 384000
```

两者相加正好为 1,000,000。

这并不意味着每次请求都会真的发送 61.6 万 token。
它只是告诉 VS Code：在上下文裁剪、文件内容选择和任务规划时，这个模型最多能容纳多少输入和输出。


## 保存配置并在 Copilot Chat 中选择模型

保存 `chatLanguageModels.json` 后，执行：

```text
Developer: Reload Window
```

或者直接重启 VS Code。

接着打开 Copilot Chat：

```text
Ctrl + Shift + I
```

在 Chat 输入框附近找到模型选择器。

正常情况下，你应该能看到：

```text
DeepSeek
├── DeepSeek V4 Flash
└── DeepSeek V4 Pro
```

如果没有出现，先不要怀疑 API Key。

按这个顺序排查：

1. 检查 JSON 是否存在语法错误；
2. 确认 `vendor` 是 `customendpoint`；
3. 确认模型 `url` 是完整地址；
4. 确认 `toolCalling` 已设为 `true`；
5. 执行 `Developer: Reload Window`；
6. 完全退出并重启 VS Code。


## 如何在 Chat、Agent 与 MCP 中使用 DeepSeek

配置成功后，DeepSeek 模型可以用于普通对话，也可以用于 Agent 模式。

### 普通 Chat

适合：

- 解释代码；
- 阅读报错；
- 生成函数；
- 输出技术文档；
- 拆解需求；
- Review 一段代码；
- 将业务描述转换为开发任务。

这类任务一般优先用 Flash。

例如：

```text
请分析当前模块的依赖关系，并指出最可能导致循环依赖的位置。
```

或者：

```text
根据当前仓库的目录结构，给出一个 Feature-First 的重构方案，不要直接修改文件。
```

### Agent 模式

Agent 模式更适合需要 **读取上下文—制定计划—修改文件—调用工具—验证结果** 的任务。

例如：

```text
分析这个仓库的认证模块，定位 refresh token 失效后仍可能重复刷新的原因。
先输出排查计划，确认后再修改代码。
```

或者：

```text
根据 specs/001-user-center/spec.md 实现用户资料编辑功能。
先读取相关规范、数据模型和现有接口，再按阶段执行任务。
每个阶段结束后说明改动文件与验证结果。
```

对于复杂任务，建议选择 DeepSeek V4 Pro，并将 reasoning effort 调整为 `max`。

### MCP 工具调用

Custom Endpoint 并不会阻断 MCP。

只要：

1. VS Code 已配置 MCP Server；
2. DeepSeek 模型支持 Tool Calls；
3. 模型声明中设置了 `"toolCalling": true`；

Agent 就可以继续调用 MCP 工具。

模型只是决策层，MCP 是工具层。两者不需要来自同一个供应商。


## 可选：为后台辅助任务指定 DeepSeek Flash

VS Code 不只在你输入问题时使用模型。

它还会处理很多后台任务，例如：

- 自动生成聊天标题；
- 生成 Git Commit Message；
- 推荐分支名称；
- 判断用户意图；
- 生成轻量摘要；
- 处理部分 Git Review 辅助任务。

这些任务不值得消耗 Pro。

可以在 VS Code Settings 中搜索：

```text
chat.utilityModel
```

和：

```text
chat.utilitySmallModel
```

建议配置为：

| 设置项                      | 推荐模型              |
| ------------------------ | ----------------- |
| `chat.utilityModel`      | DeepSeek V4 Flash |
| `chat.utilitySmallModel` | DeepSeek V4 Flash |

这样可以让复杂 Agent 任务使用 Pro，而轻量任务使用 Flash。

这是一个容易被忽略的成本控制点。很多人只盯着主聊天模型，却让所有后台任务继续走默认模型，
最后模型费用和实际体验都没有真正统一。


## DeepSeek Flash 与 Pro 的工作分工建议

下面是一套更接近日常开发的模型使用策略。

| 场景          | 推荐模型  | 推理强度 |
| ----------- | ----- | ---- |
| 简单代码解释      | Flash | High |
| 生成单个函数      | Flash | High |
| 修改一个小型组件    | Flash | High |
| 写提交信息       | Flash | High |
| 分析复杂报错      | Pro   | High |
| 多文件重构       | Pro   | Max  |
| 设计数据模型      | Pro   | Max  |
| 业务流程拆解      | Pro   | Max  |
| 大型 Agent 任务 | Pro   | Max  |
| 长上下文代码库分析   | Pro   | Max  |

可以把 Flash 看成开发中的 **常驻模型**，把 Pro 留给真正复杂的问题。

模型能力很强时，最大的浪费往往不是模型不够好，而是所有任务都使用了最贵的模型。


## 常见问题排查

### 模型没有出现在模型选择器中

优先检查：

```json
"toolCalling": true
```

如果模型不支持工具调用，或者 VS Code 认为它不支持工具调用，它可能不会出现在 Agent 相关的模型列表中。

然后检查 JSON 是否可解析。

一个逗号遗漏，就足以让整个 Provider 失效。


### 请求返回 401 Unauthorized

通常是 API Key 问题。

检查：

- Key 是否完整复制；
- 是否带了额外空格；
- Key 是否已失效；
- DeepSeek 账户是否有可用余额；
- 是否误用了其他平台的 Key；
- 是否将 Key 写进了错误的 Provider 配置中。

不要在聊天窗口、截图或日志中暴露 API Key。


### 请求返回 404 Not Found

通常是 URL 写错。

正确：

```text
https://api.deepseek.com/chat/completions
```

错误示例：

```text
https://api.deepseek.com
```

或者：

```text
https://api.deepseek.com/v1/chat/completions
```

DeepSeek 当前的官方 Base URL 是 `https://api.deepseek.com`，
Chat Completions 请求路径是 `/chat/completions`。


### 普通 Chat 能用，但 Agent 模式无法选择模型

检查两项：

```json
"toolCalling": true
```

以及模型服务本身是否真实支持 OpenAI Tool Calls 格式。

有些第三方网关表面上“兼容 OpenAI”，但只兼容纯聊天请求，
不支持 `tools`、`tool_calls`、`tool_call_id` 等字段。
这样的服务可以聊天，但很难可靠运行 Agent。


### 模型回复慢，或者价格明显偏高

先检查自己是否把所有任务都开成了高推理强度。

可以先从下面的组合开始：

```text
Flash + High：默认日常使用
Pro + High：复杂分析
Pro + Max：大型 Agent 任务
```

此外，长上下文不等于应该无节制地塞文件。

把整个仓库一次性扔进 Chat，通常不会让答案更好，只会让上下文更乱。
Agent 最有效的方式仍然是：先定位文件，再读取相关模块，再逐步执行。


### 自定义模型能聊天，但没有行内补全

这是正常现象。

BYOK Custom Endpoint 主要覆盖 Chat、Agent、工具调用和部分 utility task。
普通行内补全、语义搜索、embeddings 等能力仍可能依赖 GitHub Copilot 服务和 GitHub 登录状态。

不要把“模型接进了 Copilot Chat”理解成“所有 Copilot 功能都换成了 DeepSeek”。
