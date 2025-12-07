---
layout: post
title: "geoserver 跨域设置"
date: 2018-07-01 09:56:00 +0800
categories: Code
tags: [Network]
comments: true
---

在前端项目中，如果 WebGIS 应用和 GeoServer **不在同一个域名 / 端口** 下运行，例如：

- 前端在 `http://localhost:3000`
- GeoServer 在 `http://localhost:8080/geoserver`

浏览器会因为同源策略（Same-Origin Policy）阻止前端直接请求 GeoServer 的服务（WMS、WFS、WMTS 等）。
这时，通常会在控制台里看到类似 **CORS**（Cross-Origin Resource Sharing，跨域资源共享）相关的报错。

简单理解：

> **跨域限制** 是浏览器为了安全做的保护机制，用来限制页面随便访问第三方服务器的数据。

如果我们确认 GeoServer 是可信服务，就可以通过 **启用 CORS** 的方式，让浏览器“放心地”放行这些跨域请求。


### 在 GeoServer 中启用 CORS（基于内置 Jetty）

GeoServer 的独立发行版默认使用 Jetty 作为内置应用服务器。
要让其他域名下的 JavaScript 应用可以访问 GeoServer 服务，需要在 Jetty 中开启 CORS 支持。

官方文档会提到：

> Enable Cross-Origin Resource Sharing (CORS) to allow JavaScript applications outside of your own domain to use GeoServer.
> For more information on what this does and other options see Jetty Documentation.

也就是说，我们要在 GeoServer 的 `web.xml` 中打开 Jetty 提供的 `CrossOriginFilter`。

> 配置文件位置（默认独立版）：
> `webapps/geoserver/WEB-INF/web.xml`

在该文件中找到如下配置，并将其“取消注释”（去掉注释符号，使其生效）：

```xml
<web-app>
    <filter>
        <filter-name>cross-origin</filter-name>
        <filter-class>org.eclipse.jetty.servlets.CrossOriginFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>cross-origin</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
</web-app>
```

保存后，重启 GeoServer 服务，前端就可以从不同域名向 GeoServer 发送请求，而不会再被浏览器的跨域限制拦截。


### 注意

上面的配置是一个**比较“通用且粗放”的开启方式**：

- 通过 `url-pattern>/*` 对所有路径开放 CORS
- 其他细节（允许的域、方法、头信息等）使用 Jetty 默认行为

在本地开发或内部测试环境，这样做通常问题不大；
但在生产环境，建议进一步在 `web.xml` 或 Jetty 配置中做以下配置：

- 限制允许访问的域名（例如只允许公司前端域名）
- 明确允许的 HTTP 方法（GET、POST 等）
- 根据业务需要控制允许携带的请求头、是否允许携带 Cookie 等

具体可参考 Jetty 官方文档中的 `CrossOriginFilter` 配置说明，根据实际需求配置有限的权限，避免把服务暴露得过于宽松。
