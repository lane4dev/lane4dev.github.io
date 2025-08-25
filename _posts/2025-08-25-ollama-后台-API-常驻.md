---
layout: post
title: "如何让 Ollama 后台 API 常驻"
date: 2025-06-23 00:00:00 +0800
categories: AI
tags: ["Ollama"]
comments: true
---

在默认配置下，Ollama 会在一段时间没有请求后，把模型从内存里卸载掉——这对节省资源是好事，
但对 **需要低延迟响应** 的服务来说就很烦：

第一次请求总是要“热身”，尤其是大模型，冷启动几秒甚至十几秒都不奇怪。

这篇文档的目标是：在 **不改动 Ollama 服务本身** 的前提下，通过合理调用 API，让模型常驻内存，尽量避免冷启动。

---

## 背景

Ollama 的 HTTP 接口（例如 `/api/generate`、`/api/chat`）会在请求结束后，根据 `keep_alive` 参数决定：

**模型在内存里保留多久**

是否在空闲一段时间后自动卸载

常见的几种 `keep_alive` 用法（简化版理解）：

- `keep_alive` 省略或为默认值：
  服务自己决定闲置多久卸载（取决于版本和配置）。

- `keep_alive: 0`：
  请求结束就卸载模型。

- `keep_alive: 数字（秒）`：
  在这段“空闲时间”内保留模型，超时后才卸载。

- `keep_alive: -1`：
  **尽量不卸载**，保持常驻（除非服务重启或资源不足等原因）。

因此，我们可以利用 `keep_alive: -1`，配合一次“预热调用”，让模型常驻内存，后续请求就不再经历冷启动。

---

## 最简单的方案

```bash
curl -sS http://192.168.50.131:11434/api/generate \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen3:8b",
    "prompt": "warmup",
    "stream": false,
    "keep_alive": -1
  }'
```

这一行背后做了几件事：

1. **加载模型 `qwen3:8b`**

   如果模型未加载，会触发一次完整加载过程（耗时）。

2. **执行一次很轻的推理**

   `prompt: "warmup"` 不要求任何业务输出，只是“点一下火”。

3. **设置模型常驻**

   `keep_alive: -1` 告诉 Ollama：这次跑完之后，不要自动卸载模型。

结果：

- 这条命令执行完后，`qwen3:8b` 会留在内存中。
- 后续调用 `http://192.168.50.131:11434/api/generate` 或 `/api/chat` 时，基本都是“热模型”，响应明显更快。

如果只是个人使用、开发环境，这一行命令已经能解决 80% 的冷启动问题。

---

## 更稳定的方案

仅仅手动跑一条 `curl`，有几个问题：

1. 主机重启、Ollama 重启之后，模型又会回到“未加载”状态。
2. 多个模型需要常驻时，你需要记得逐个手动执行。
3. 没有监控和自愈：模型被手动卸载/挤出内存后，没人会“重新帮你暖机”。

下面是一个更像**解决方案**的设计思路。

### 方案 A：系统启动时自动预热（systemd / 开机脚本）

**适用场景：**

自己的服务器、NAS 或云主机上部署了 Ollama，希望“机器一开机，模型就自动常驻”

**思路：**

1. 写一个简单的 shell 脚本，负责给所有需要常驻的模型发一遍“预热请求”。
2. 配置为 systemd 服务（或开机启动脚本），在系统启动后自动执行一次。

**示例脚本 `warmup_ollama.sh`：**

```bash
#!/usr/bin/env bash
set -e

OLLAMA_HOST="192.168.50.131"
OLLAMA_PORT="11434"

MODELS=("qwen3:8b")  # 需要常驻的模型列表

for model in "${MODELS[@]}"; do
  echo "Warming up model: $model"
  curl -sS http://$OLLAMA_HOST:$OLLAMA_PORT/api/generate \
    -H 'Content-Type: application/json' \
    -d "{
      \"model\": \"$model\",
      \"prompt\": \"warmup\",
      \"stream\": false,
      \"keep_alive\": -1
    }" > /dev/null
done

echo "Warmup done."
```

给脚本加执行权限：

```bash
chmod +x /opt/ollama/warmup_ollama.sh
```

如果用 systemd，可以写一个简单的服务：

```ini
[Unit]
Description=Warmup Ollama Models
After=network-online.target

[Service]
Type=oneshot
ExecStart=/opt/ollama/warmup_ollama.sh

[Install]
WantedBy=multi-user.target
```

启用后，每次系统启动，都会自动执行一次预热。

搭配 `keep_alive: -1`，模型就会一直常驻，直到服务或机器重启。

---

### 方案 B：定时“心跳 + 自愈”脚本

**适用场景：**

服务器上还跑着其它模型/服务，内存紧时可能把模型挤掉。
希望，如果模型被卸载，系统能自动重新预热。

**基本思路：**

1. 定期发一个轻量请求，既当“心跳检测”，又可以顺带 keep alive。
2. 如果发现模型响应异常或耗时异常（疑似被卸载重载），可记录日志或告警。
3. 有需要的话，可以在失败后自动重试或重新发“暖机请求”。

**例：用 cron 每 10 分钟跑一次简单预热**

```bash
*/10 * * * * /opt/ollama/warmup_ollama.sh >> /var/log/ollama_warmup.log 2>&1
```

考虑到你已经用了 `keep_alive: -1`，实际心跳的重点在于：

- 确认模型确实还在（避免“看着常驻其实已被卸载”的错觉）。
- 发现异常时能快速知道，而不是等线上接口报“首个请求爆慢”。

---

### 方案 C：在业务网关 / API 服务中内建“预热逻辑”

如果你有自己的后端服务（FastAPI、Node.js、Go 等）对接 Ollama，完全可以在 **业务服务启动时** 做一遍预热：

好处：

1. 部署流程天然绑定：**只要你的业务服务在，模型就会被预热**。
2. 接口层可以统一管理多个模型的 warmup。

做法：

1. 在服务的 startup hook / on_startup 回调里，调用一次上面的预热接口。
2. 如果预热失败，可以选择：

   - 直接启动失败（避免带着冷模型上线）；
   - 或容忍失败但打日志/报警。

代码（以 Python/FastAPI 为例）：

```python
import httpx
from fastapi import FastAPI

app = FastAPI()

OLLAMA_URL = "http://192.168.50.131:11434/api/generate"
MODEL_NAME = "qwen3:8b"

async def warmup_model():
    payload = {
        "model": MODEL_NAME,
        "prompt": "warmup",
        "stream": False,
        "keep_alive": -1,
    }
    async with httpx.AsyncClient(timeout=60) as client:
        resp = await client.post(OLLAMA_URL, json=payload)
        resp.raise_for_status()

@app.on_event("startup")
async def startup_event():
    try:
        await warmup_model()
        print(f"Model {MODEL_NAME} warmup done.")
    except Exception as e:
        # 根据你的容错策略决定要不要抛出
        print(f"Warmup failed: {e}")
```

---

## 方案设计时需要注意的几点

### 内存与 GPU 资源

`keep_alive: -1` 的本质，是 **“尽量不卸载”**。
如果你常驻多个大模型，很容易把显存/内存吃满，导致：

- 其它任务无法启动
- 系统频繁触发 OOM / 任务被杀

建议：

在同一台机器上，把“常驻模型数量”和“可接受的资源占用”绑定设计。

可以只让一个“主力模型”常驻，其它模型按需加载。

---

### 多模型场景

如果你有多个模型（例如 `qwen3:8b`、`llama3:8b`、`qwen3:1.8b`）：

可以在 `MODELS` 数组里列出来逐个 warmup。

对重要程度不同的模型，策略也可以不同：

- 核心线上模型：`keep_alive: -1`
- 次要模型：`keep_alive` 设置为几百秒/几千秒即可，兼顾资源与性能。

---

### 监控和报警（可选）

想要更“工程化”一点，可以：

在心跳脚本里统计每次响应时间，如果突然变得很慢（比如 >10 秒），就认为发生了冷启动。

通过：

- 写日志 + Loki / ELK
- 调用飞书/企业微信/Webhook

把这种“重新加载模型”的事件打出去，方便排查问题。

---

## 总结

1. **目标**：

   避免 Ollama 模型频繁冷启动，让后台 API 响应更稳定、更快速。

2. **核心手段**：

   使用 `/api/generate` 或 `/api/chat` 接口，带上 `keep_alive: -1` 做一次“暖机调用”：

   ```bash
   curl -sS http://192.168.50.131:11434/api/generate \
     -H 'Content-Type: application/json' \
     -d '{
       "model": "qwen3:8b",
       "prompt": "warmup",
       "stream": false,
       "keep_alive": -1
     }'
   ```

   这会把 `qwen3:8b` 拉进内存并尽量保持常驻。

3. **工程化落地方式**（可按需要组合使用）：

   **系统启动预热**：写 `warmup_ollama.sh` + systemd 服务，机器一启动就预热模型。

   **定时心跳/自愈**：用 cron 定时跑预热脚本，检查模型是否还“热着”。

   **业务服务内置预热**：在 FastAPI / Node / Go 服务启动时自动调用一次 warmup，使部署过程更自动化。

4. **注意事项**：

   1. 常驻模型会长期占用内存/GPU，谨慎选择常驻模型数量。
   2. 多模型环境下，核心模型常驻，非核心模型用有限 `keep_alive` 即可。
   3. 如有条件，可配合监控和报警，跟踪“模型被迫重载”的情况。
