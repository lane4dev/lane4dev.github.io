---
layout: post
title: "Nextcloud OSS 存储配置指南"
date: 2024-01-20 00:00:00 +0800
categories: Code
tags: ["NextCloud", "OSS"]
comments: true
---

这篇文档带你一步步把阿里云 OSS 接入到 Nextcloud，覆盖三种常见方式：

1. 通过 **External Storage 插件**，把 OSS Bucket 挂成一个目录
2. 把 **OSS 配置成主存储（Primary Storage）**
3. 用 **ossfs 挂载到本地，再由 Nextcloud 访问**（不推荐，只做补充说明）


## 准备工作

在开始配置之前，你需要准备好：

- 一个可以正常访问的 Nextcloud 实例
- 一个已经创建好的 OSS Bucket
- OSS 的子账号或 RAM 用户（AccessKey ID / AccessKey Secret）
- OSS Endpoint（建议使用内网 Endpoint，如果 Nextcloud 部署在阿里云 ECS 上）

后续示例中出现的以下字段，请替换为自己的实际值：

- `xxx`：Bucket 名称、密钥等具体值
- `xxx.aliyuncs.com`：OSS 的 Endpoint，例如 `oss-cn-hangzhou.aliyuncs.com`


## 方式一：使用 External Storage 插件挂载 OSS（推荐入门）

适用场景：
希望 **保留 Nextcloud 本地存储**，只是把 OSS 当成一个额外的共享目录使用。

挂载后，OSS 的 Bucket 会 **以目录的形式出现在 Nextcloud 主页面** 中，像一个共享网盘。

### 启用 External Storage Support 插件

1. 使用管理员账号登录 Nextcloud Web 界面。
2. 点击右上角头像，选择 **「应用」**。
3. 在应用列表中找到 **“External storage support”**，点击 **启用**。

![]({{ base.siteurl }}/assets/images/2024-01/20-001-nc-setting.png)

![]({{ base.siteurl }}/assets/images/2024-01/20-002-nc-plugin.png)


### 进入 External Storage 管理页面

1. 再次点击右上角头像，进入 **「设置」**。
2. 在左侧菜单的最下方，可以看到管理员分区。
3. 点击 **「External storage」**（外部存储）。

![]({{ base.siteurl }}/assets/images/2024-01/20-003-nc-es-setting.webp)

### 添加 S3 兼容存储（OSS）

在 **External storage** 页面中，新增一条外部存储配置：

1. 在存储配置列表中，新建一行：

   - **Folder name**：
     在 Nextcloud 中看到的目录名，例如：`oss-share`
   - **External storage** 类型：
     选择 **Amazon S3** 或 **S3 compatible**（具体名称取决于 Nextcloud 版本）

2. 在下方的配置项中填写 OSS 信息（典型示例）：

   - Bucket：`xxx`（你的 OSS Bucket 名称）
   - Hostname / Endpoint：`xxx.aliyuncs.com`
   - Port：`443`
   - Enable SSL：勾选（使用 HTTPS）
   - Access key：`xxx`（AccessKey ID）
   - Secret key：`xxx`（AccessKey Secret）
   - Region：可留空或填写实际 Region ID（例如 `oss-cn-hangzhou`），视插件要求而定
   - 使用 Path Style 与否：根据 Nextcloud 版本及 OSS 兼容情况，通常可关闭 path style（与你后面 `use_path_style` 一致）

3. 在右侧可以设置访问权限：

   - 仅管理员可见
   - 指定用户或用户组可访问

配置保存后，Nextcloud 会尝试连接 OSS。如果配置正确，状态会显示为绿色对勾。

### 在 Nextcloud 中访问 OSS 目录

返回 Nextcloud 文件主界面，你会看到一个新出现的文件夹，例如 `oss-share`。打开后，即可像浏览普通文件夹一样访问 OSS 中的对象。

![]({{ base.siteurl }}/assets/images/2024-01/20-004-nc-es-result.webp)

更多细节可参考官方文档：
[Configuring External Storage (GUI) — Nextcloud Admin Manual](https://docs.nextcloud.com/server/27/admin_manual/configuration_files/primary_storage.html)


## 方式二：将 OSS 配置为 Nextcloud 主存储（Primary Storage）

适用场景：
希望 **所有用户文件都直接存储在 OSS 上**，实现计算与存储分离。

和前一种方式不同，这里不是“多一个目录”，而是直接把 OSS 当成 Nextcloud 的底层存储。一般建议在 **新部署** 时就启用该配置，避免数据迁移的复杂度。

### 在 config.php 中开启对象存储

编辑 Nextcloud 的配置文件：

- 典型路径：`/var/www/html/config/config.php`

在配置数组中添加 `objectstore` 配置段：

```php
'objectstore' => [
    'class' => '\\OC\\Files\\ObjectStore\\S3',
    'arguments' => [
        // 填写 OSS 的 Bucket 名称
        'bucket' => 'xxx',
        'autocreate' => true,

        // 填写创建的子账户的 AccessKey ID
        'key'    => 'xxx',

        // 填写创建的子账户的 AccessKey Secret
        'secret' => 'xxx',

        // 填写 OSS 的内网 Endpoint 域名（建议使用内网）
        'hostname' => 'xxx.aliyuncs.com',

        'port' => 443,
        'use_ssl' => true,

        // 是否使用 Path Style，根据实际情况调整
        'use_path_style' => false,
    ],
],
```

提示：

- `autocreate => true` 表示如果 Bucket 不存在，会尝试自动创建（需要子账号有权限）。
- `hostname` 建议使用内网 `*.aliyuncs.com`，可以显著降低延迟、节省流量成本。

### 通过环境变量配置（可选）

如果你使用 Docker 或容器编排，Nextcloud 通常支持通过环境变量传入对象存储配置。下面是对应于 S3/OSS 的变量示例：

```bash
OBJECTSTORE_S3_HOST=xxx.aliyuncs.com
OBJECTSTORE_S3_BUCKET=xxx
OBJECTSTORE_S3_REGION=
OBJECTSTORE_S3_KEY=xxx
OBJECTSTORE_S3_SECRET=xxx
OBJECTSTORE_S3_USEPATH_STYLE=false
OBJECTSTORE_S3_AUTOCREATE=true
OBJECTSTORE_S3_SSL=true
OBJECTSTORE_S3_PORT=443
```

在容器启动时注入这些环境变量，Nextcloud 会据此连接 OSS 并将其用作主存储。

注意：

- 在已有大量数据的生产实例上切换主存储，风险较大，建议在全新环境测试验证后再推广。
- 任何涉及存储迁移的操作，都建议提前做快照或备份。

参考文档：
[Configuring Object Storage as Primary Storage — Nextcloud Admin Manual](https://docs.nextcloud.com/server/27/admin_manual/configuration_files/primary_storage.html)


## 方式三：通过 ossfs 将 OSS 挂载为本地文件系统（不推荐）

**注意：** 官方说明中明确提到：ossfs 在某些场景下会有兼容性与性能问题，更适合作为简单文件访问或迁移工具，而不是高强度在线服务的后端存储。

这里介绍是为了完整性——如果你只是想临时把 Bucket 挂到系统上看一眼内容，或者做一些一次性的迁移，这种方式也可以考虑。

### 安装 ossfs

以 Debian/Ubuntu 环境为例：

```bash
apt-get update
apt-get install gdebi-core
gdebi ossfs_1.91.1_ubuntu18.04_amd64.deb
```

### 安装 MIME 支持（可选）

为了让上传文件的 Content-Type 与扩展名匹配，可以安装：

```bash
apt-get install mime-support
```

### 配置 OSS 账号信息

在 `/etc/passwd-ossfs` 中写入 Bucket 与账号信息：

```bash
echo BucketName:yourAccessKeyId:yourAccessKeySecret > /etc/passwd-ossfs
chmod 640 /etc/passwd-ossfs
```

这一步相当于给 ossfs 提供访问 OSS 的“钥匙”，务必限制文件权限，避免泄露。

### 挂载 Bucket 到本地目录

```bash
mkdir /mnt/ossfs
ossfs pengfeiyun-test /mnt/ossfs -o url=http://pengfeiyun-test.oss-cn-zhangjiakou.aliyuncs.com
```

如果你的 Nextcloud 部署在阿里云 ECS 上，可以使用 **内网域名**：
如将 Endpoint 改为 `oss-cn-hangzhou-internal.aliyuncs.com`，既省钱又更稳定。

### 取消挂载

不再需要时，可以取消挂载：

```bash
fusermount -u /mnt/ossfs
```

⚠️ 再强调一次：
把 ossfs 当作 Nextcloud 的长期后端存储存在兼容性风险，不建议用于生产环境的核心存储。
如果是为了“看一看 Bucket 里有什么”或者做一次性迁移，是 ok 的。


## 如何选择合适的接入方式？

- **只想多一个共享目录、简单挂个 Bucket：**，
用 **External Storage 插件**，最直观、最易回滚。

- **希望彻底上云，所有文件都放到 OSS：**，
配置 **对象存储为主存储**，但一定要在测试环境跑通、再慎重上生产。

- **想临时挂载 OSS、做迁移或脚本处理：**，
可以用 **ossfs**，但不推荐作为 Nextcloud 的长期后端存储方案。
