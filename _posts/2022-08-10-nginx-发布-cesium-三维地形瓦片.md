---
layout: post
title: "在 Nginx 里发布 Cesium 三维地形瓦片"
date: 2022-08-10 18:14:00 +0800
categories: Code
tags: ["Nginx", "Cesium"]
comments: true
---

这是一篇简短的备忘录，记录如何用 Nginx 把本地的 **Cesium 地形瓦片（.terrain）** 当静态文件发出去，让 Cesium 通过 URL 直接访问。

## 场景 & 目录

假设目录结构：

```bash
/usr/share/nginx/html/
  └── tilesets/
      └── terrain/
          └── {z}/{x}/{y}.terrain
```

浏览器访问示例：

```text
http://localhost/tilesets/terrain/0/0/0.terrain
```

---

## Nginx 核心配置

只记关键点：**root 对齐** 以及为 .terrain 设置**正确响应头**。

```nginx
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    # 静态站点根目录
    root   /usr/share/nginx/html;
    index  index.html index.htm;

    # 默认静态文件
    location / {
        try_files $uri $uri/ =404;
    }

    # Cesium 地形瓦片
    location ~* ^/tilesets/terrain/.*\.terrain$ {
        # 二进制流
        default_type application/octet-stream;
        # 如果瓦片本身已经 gzip 压缩，告诉浏览器不要再解错
        add_header Content-Encoding gzip;
        # 如无特殊需要，可以不用 attachment，避免触发下载框
        # add_header Content-Disposition attachment;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

---

## Cesium 端配置示例

在前端里这样用：

```js
const terrainProvider = new Cesium.CesiumTerrainProvider({
  url: "http://localhost/tilesets/terrain",
});

viewer.terrainProvider = terrainProvider;
```

---

## 4. 备忘

- **路径要对齐**：`root + URL` 能拼到真实文件。

  - 例：`root=/usr/share/nginx/html` + `/tilesets/...` → `/usr/share/nginx/html/tilesets/...`

- **gzip 只加一次**：瓦片文件如果是 `.terrain` 已经 gzip 过了，就用 `add_header Content-Encoding gzip;`，不要再让 Nginx 主动压缩同一路径。

- **先用浏览器/ curl 测试**：

  - 能直接请求到 `.terrain`（状态 200）
  - 响应头里有 `Content-Encoding: gzip`，`Content-Type: application/octet-stream` 即可。
