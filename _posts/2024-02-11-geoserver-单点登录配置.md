---
layout: post
title: "Geoserver 单点登录配置"
date: 2024-02-11 00:00:00 +0800
categories: Code
tags: ["GeoServer", "Keycloak"]
comments: true
---

GeoServer 自带的账号体系在小规模使用场景下其实已经够用，但一旦进入生产环境，账号分散、权限分散就会带来数据安全和审计上的风险。

如果你的系统里已经有了 Keycloak 这类统一身份认证平台，把 GeoServer 納入同一套认证与授权体系，会更利于集中管理访问权限、降低泄露风险、做好安全审计。

这篇教程带你把 **Keycloak 的 OIDC 登录** 接到 **GeoServer** 上：

- 后台管理可以用 Keycloak 账号登录
- 第三方应用也可以通过 Bearer Token 访问 GeoServer 的 REST 接口
- 可以限制匿名用户访问图层


## 前置条件

开始前，建议你先确认：

- 已部署并可访问的 **Keycloak** 实例
- 已创建好对应的 **Realm**
- 有权限访问的 **Keycloak 管理员账号**
- 已部署并可访问的 **GeoServer**
- GeoServer 管理员账号（默认是 `admin`）


## 在 Keycloak 中创建 GeoServer 对应的 Client

首先，我们需要在 Keycloak 里为 GeoServer 建一个专用的 OIDC Client，用来发起登录和验证 Token。

1. 登录 Keycloak 管理控制台
2. 选择你要使用的 **Realm**
3. 创建 Client（名称可以类似：`geoserver-oidc`）
4. 关键参数示例（仅作参考，具体以实际环境为准）：

   - **Client Type**：`OpenID Connect`
   - **Client ID**：`geoserver-oidc`（下面会在 GeoServer 里用到）
   - **Client Authentication**：开启（需要 Client Secret）
   - **Valid redirect URIs**：填入你的 GeoServer 地址，例如：
     `http://your-geoserver-host/geoserver/*`

完成后记下：

- `Client ID`
- `Client Secret`
- Keycloak 服务地址和 Realm 名字（后面要拼出 issuer / discovery 地址）


## 登录 GeoServer 管理端

1. 在浏览器中打开 GeoServer 管理地址，例如：
   `http://your-geoserver-host/geoserver/web`
2. 使用管理员账号登录（例如 `admin`）


## 在 GeoServer 中添加新的 Authentication Filter（使用 OpenID Connect）

接下来我们要让 GeoServer “认识” Keycloak 的 OIDC。

### 打开 Authentication 配置页面

1. 在左侧菜单点击 **`Security` → `Authentication`**

![]({{ base.siteurl }}/assets/images/2024-02/11-001-gs-auth.png)

2. 在页面中找到 **`Authentication Filters`** 区域
3. 点击「**Add new**」或类似按钮，新建一个 Filter

![]({{ base.siteurl }}/assets/images/2024-02/11-002-gs-auth-filter.png)

### 选择 OpenId Connect 类型

在弹出的对话框或新页面中：

- 类型选择：**`OpenId Connect`**

![]({{ base.siteurl }}/assets/images/2024-02/11-003-gs-auth-filter-2.png)

### 使用 OpenID Discovery 自动加载配置

在 OIDC 配置页面中，你会看到一个叫：

- **OpenID Discovery document** 的字段

在这里填入 Keycloak 的 Discovery URL，一般形式如下：

```text
http://{keycloak_server}/realms/{realm}/.well-known/openid-configuration
```

示例（请根据实际情况替换）：

```text
http://auth.example.com/realms/myrealm/.well-known/openid-configuration
```

填好后，点击 **`Discover`** 按钮，GeoServer 会自动从这个 URL 读取 issuer、授权端点、Token 端点等信息，并填入下面的相关字段。

![]({{ base.siteurl }}/assets/images/2024-02/11-004-gs-auth-oidc.png)

小提示，如果点击 `Discover` 报错，先检查：

- URL 是否可以在浏览器直接访问
- Keycloak 是否对 GeoServer 所在的网络开放
- 是否有 HTTP/HTTPS、端口写错的问题

### 填写 Client ID 与 Client Secret

Discovery 只会帮你填好服务端地址，客户端身份还是要自己补上：

1. 在对应字段中填入之前在 Keycloak 创建的：

   - **Client ID**：如 `geoserver-oidc`
   - **Client Secret**：从 Keycloak Client 的 Credentials 中复制

2. 确认保存

### 允许使用 Bearer Token 访问 GeoServer（可选但常用）

如果你希望别的服务通过 REST API + Bearer Token 访问 GeoServer，就需要勾上：

- **`Allow Attached Bearer Tokens`**

勾上后，外部应用可以通过在 HTTP 请求头里带上：

```http
Authorization: Bearer <token>
```

来访问 GeoServer 资源（前提是 Token 有合适的权限）。

![]({{ base.siteurl }}/assets/images/2024-02/11-005-gs-auth-oidc-bt.png)

### 配置 Role service（从哪里拿用户角色）

针对 Keycloak，你通常有两种常用选择：

- **Access Token**：从 Access Token 中直接解析角色（通常是 `realm_access` 或 `resource_access`）
- **UserInfo Endpoint**：通过调用 UserInfo 接口获取用户信息和角色

在 GeoServer 的 Role service 设置中：

- 选择 **`Access Token`** 或 **`UserInfo Endpoint`**，根据你的 Keycloak 配置和角色映射方式来决定。

![]({{ base.siteurl }}/assets/images/2024-02/11-007-gs-auth-oidc-rs-1.png)

### 保存 Filter 配置

确认前面所有字段（特别是 URL、Client ID、Client Secret）无误后，点击 **保存**。


## 将新建的 OIDC Filter 加入 Filter Chains

只有创建 Filter 还不够，我们还需要告诉 GeoServer：哪些入口要走 OIDC 认证。

### 找到 Filter Chains 配置

仍然在 **`Security → Authentication`** 页面，下方有一块是 **`Filter Chains`**。这里定义了不同类型请求（web 管理界面、REST 接口、瓦片缓存等）需要经过哪些认证 Filter。

![]({{ base.siteurl }}/assets/images/2024-02/11-008-gs-auth-fc.png)


### 为需要保护的 Chain 添加 OIDC Filter

你可以按需要为不同的 Chain 添加刚才创建的 OIDC Filter。常见的做法是：

如果你希望使用 OAuth2/OIDC 保护所有 GeoServer 服务和管理界面，可以把 OIDC Filter 添加到以下所有 Filter Chains：

- `web`（GeoServer Web 管理界面）
- `rest`（REST 接口）
- `gwc`（GeoWebCache）
- `default`（其他默认入口）

对每一个你想保护的 **Filter Chain**：

1. 选中对应的 Chain（例如 `web`、`rest`）
2. 在配置界面中，将左侧的 **`Available`** 列表中你的 OIDC Filter（可能显示类似 `oidcKeycloak`）移动到右侧 **`Selected`** 列表
3. 注意顺序：

   - 确保 `anonymous` 始终处于 **底部**，否则可能导致匿名绕过你期望的认证流程

![]({{ base.siteurl }}/assets/images/2024-02/11-009-gs-auth-fc-2.png)

配置完成后，点击 **保存**。


## 验证：通过 Keycloak 登录 GeoServer

配置顺利的话，这时你再访问 GeoServer 的登录界面，会看到一个新按钮。

1. 打开 GeoServer 管理界面登录页
2. 在右侧登录区域，你会看到多了一个 **`OIDC`** 按钮（文字可能略有不同）

![]({{ base.siteurl }}/assets/images/2024-02/11-010-gs-login-1.png)

3. 点击 `OIDC` 按钮，页面会跳转到 Keycloak 登录页
4. 输入 Keycloak 账号密码并登录

![]({{ base.siteurl }}/assets/images/2024-02/11-011-gs-login-2.png)

5. 登录成功后，浏览器会自动跳回 GeoServer，此时你应该已经是登录状态，右上角会显示当前登录用户的用户名（来自 Keycloak）。

![]({{ base.siteurl }}/assets/images/2024-02/11-012-gs-login-3.png)


## 限制匿名用户访问 GeoServer 图层

如果你不希望匿名用户随便查看图层数据，可以在 GeoServer 里收紧数据安全规则。

### 打开 Data Security 配置

1. 使用管理员账号（或者已有足够权限的账号）登录 GeoServer
2. 在左侧菜单点击：**`Security` → `Data`**

### 修改默认读取规则 `*.*.r`

在 **`Data Security`** 页面中，你会看到各种规则，其中有一条类似：

![]({{ base.siteurl }}/assets/images/2024-02/11-014-gs-security.png)

- `*.*.r` —— 表示“所有工作区、所有图层的读取权限”

1. 找到并点击这一条规则（`*.*.r`）

2. 在编辑页面中：

   - **取消勾选** `Grant access to any role`（允许任何角色访问）
   - 确保 `ROLE_ANONYMOUS` 没有出现在右侧 **`Selected Roles`** 列表中（即匿名用户不再拥有读取权限）

   ![]({{ base.siteurl }}/assets/images/2024-02/11-015-gs-security.png)

3. 点击 **保存**

保存后，未登录用户将无法再直接浏览图层：

- 如果请求界面，通常会被重定向到登录流程
- 如果通过服务访问，可能会收到未授权（401/403）响应
