---
layout: post
title: "在 Nextcloud 中安装和配置 OnlyOffice"
date: 2024-04-30 00:00:00 +0800
categories: Code
tags: ["NextCloud", "OnlyOffice"]
comments: true
---

OnlyOffice DocumentServer 可以让 Nextcloud 直接在线编辑 Word / Excel / PPT 等文档。
下面以 Docker 方式部署 OnlyOffice，并在 Nextcloud 中完成配置为例，带你一步一步完成集成。


## 前提条件

在开始前，确保你已经具备：

- 一套可正常访问的 **Nextcloud 部署**（建议使用 docker-compose 管理）。
- 已安装 **Docker** 和 **docker-compose**。
- 对自己当前的部署结构有基本了解：

  - 是“多容器 + Docker 网络”的方式，还是 Nextcloud 单机服务。
  - Nextcloud 和 OnlyOffice 是否在同一个 docker-compose 网络里。

---

## 通过 docker-compose 部署 OnlyOffice DocumentServer

在 Nextcloud 的 docker-compose 配置中，增加一个 `onlyoffice` 服务（当然也可以独立部署），例如：

```yaml
onlyoffice:
  image: onlyoffice/documentserver:7.5
  restart: always
  ports:
    - 8887:80
  volumes:
    - onlyoffice_logs:/var/log/onlyoffice
    - onlyoffice_data:/var/www/onlyoffice/Data
  environment:
    - JWT_ENABLED=true
    - JWT_SECRET=n3ECoD8BBnoAdwUGBMYer26DBAgxNKgz
```

### 关键配置说明

- **镜像：**
  `onlyoffice/documentserver:7.5` 为 DocumentServer 正式镜像，版本号可按需调整。

- **端口映射：**
  `8887:80` 表示宿主机的 `8887` 端口转发到容器的 `80` 端口。
  之后在浏览器中访问 `http://<你的服务器IP>:8887/`，可以粗略检查服务是否正常启动。

- **数据卷：**

  - `onlyoffice_logs:/var/log/onlyoffice`：保存日志，方便排查问题。
  - `onlyoffice_data:/var/www/onlyoffice/Data`：保存 DocumentServer 的数据文件。

- **JWT 配置：**

  - `JWT_ENABLED=true`：开启 Token 校验。
  - `JWT_SECRET=...`：JWT 密钥，用于 OnlyOffice 与 Nextcloud 之间的安全通信。
    注意，**这串密钥需要跟 Nextcloud 插件配置中的密钥保持一致**。

保存 docker-compose 文件后，启动或更新服务：

```bash
docker-compose up -d onlyoffice
```

容器正常运行后，可以先用浏览器打开 `http://<服务器IP>:8887/` 进行基本连通性确认。

---

## 在 Nextcloud 中安装 OnlyOffice 插件

接下来，让 Nextcloud 认识这套 DocumentServer。

1. 使用管理员账号登录 Nextcloud 网页端。
2. 点击右上角用户头像 → 进入 **“应用（Apps）”** 页面。
3. 在左侧分类中找到 **“Office & 文档”** 或搜索栏输入 `OnlyOffice`。
4. 找到 **OnlyOffice** 插件，点击 **“下载并启用”**。

插件安装完成后，Nextcloud 菜单中会多出一个 OnlyOffice 配置入口。

---

## 修改 Nextcloud 配置，允许访问本地服务

如果你的 OnlyOffice 和 Nextcloud 在同一台机器或同一内网中，通过内网地址访问时，Nextcloud 默认会有一定限制。
需要在配置中显式允许本地远程服务器访问。

1. 打开 Nextcloud 的配置文件，一般为：

   ```bash
   config/config.php
   ```

2. 增加或修改如下配置项：

   ```php
   <?php
   $CONFIG = array (
     // ...
     'allow_local_remote_servers' => true,
     // ...
   );
   ```

3. 保存后，重启 Nextcloud 容器或服务，让配置生效：

   ```bash
   docker-compose restart nextcloud
   ```

如果你的 Nextcloud 服务名称不是 `nextcloud`，请根据实际服务名调整上面的命令。

---

## 在 Nextcloud 中配置 OnlyOffice 服务

完成上面的准备工作后，就可以在 Nextcloud 里绑定这台 DocumentServer 了。

1. 使用管理员账号登录 Nextcloud。

2. 点击右上角头像 → **“设置”** → 左侧找到 **“ONLYOFFICE”** 配置项。

3. 在配置页面中填写：

   - **文档编辑服务地址（Document Editing Service address）：**
     根据你的部署方式填写，例如：

     - 如果从浏览器访问：
       `http://<服务器IP>:8887/`
     - 如果 Nextcloud 与 OnlyOffice 在同一个 docker-compose 网络中，可以用服务名：
       `http://onlyoffice/` 或 `http://onlyoffice:80/`

   - **JWT 密钥：**
     填入与你在 docker-compose 中配置的 `JWT_SECRET` 完全相同的值：

     ```text
     n3ECoD8BBnoAdwUGBMYer26DBAgxNKgz
     ```

4. 保存配置（一般页面会提供“保存”或“测试连接”按钮）。

如果一切正常，页面会提示连接成功。
此时你可以上传一个 `.docx` 或 `.xlsx` 文件，点击打开，应该会直接进入 OnlyOffice 在线编辑界面。

---

## 简单验证与常见问题

### 无法连接到文档编辑服务

- 检查宿主机端口是否开放：

  ```bash
  curl http://<服务器IP>:8887/
  ```

- 若 Nextcloud 和 OnlyOffice 在同一 docker-compose 中，优先使用服务名 `http://onlyoffice/`，避免被宿主机防火墙影响。

### 提示 JWT 相关错误

- 确认 docker-compose 中的 `JWT_SECRET` 与 Nextcloud OnlyOffice 配置页面中的密钥 **完全一致**（包括大小写）。
- 修改密钥后记得：

  ```bash
  docker-compose restart onlyoffice
  ```

### 内网地址访问被拦截

- 再次确认 `config.php` 中已启用：

  ```php
  'allow_local_remote_servers' => true,
  ```

- 修改后别忘了重启 Nextcloud 容器。

---

## 总结

整个流程可以浓缩为四步：

1. 用 docker-compose 启动 `onlyoffice/documentserver` 容器，配置好端口、数据卷和 JWT。
2. 在 Nextcloud 中安装 OnlyOffice 插件。
3. 修改 `config.php`，允许访问本地远程服务器，并重启 Nextcloud。
4. 在 Nextcloud 设置里填入 OnlyOffice 服务地址和 JWT 密钥，保存并测试。

完成这些操作后，你的 Nextcloud 就摇身一变，从网盘升级为可以在线协作编辑文档的团队工作空间了。
