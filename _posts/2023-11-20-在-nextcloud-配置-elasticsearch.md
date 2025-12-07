---
layout: post
title: "在 Nextcloud 中配置 ElasticSearch 全文检索"
date: 2024-05-06 00:00:00 +0800
categories: Code
tags: ["NextCloud", "ElasticSearch"]
comments: true
---

本文记录如何在 Docker 环境下，为 Nextcloud 配置 ElasticSearch，并通过 Full Text Search 系列应用实现文件内容检索，包括：

- 使用 `docker-compose` 启动 ElasticSearch 服务
- 安装 `ingest-attachment` 与 `analysis-ik` 插件
- 在 Nextcloud 中启用全文检索相关应用
- 使用 `occ` 建立索引
- 解决 `occ fulltextsearch:index` 执行时的 NULL Pointer 报错问题


## 整体思路概览

Nextcloud 自带的搜索更偏向**文件名、标签**层面。要想实现**文件内容级别**的全文检索，需要借助：

- **ElasticSearch**：负责存储索引、响应搜索请求
- **Full Text Search 系列应用**：负责把 Nextcloud 的文件信息推送到 ElasticSearch，并提供 UI 搜索入口

流程可以简单理解为：

Nextcloud 文件 → Full Text Search → ElasticSearch 建索引 → Nextcloud 发起搜索 → ElasticSearch 返回结果

下面按步骤展开。


## 通过 docker-compose 启动 ElasticSearch

在现有的 `docker-compose.yml` 中添加 ElasticSearch 服务，例如：

```yaml
elastic_search:
  image: elasticsearch:7.17.6 # 注意：elasticsearch-analysis-ik 插件版本需与 ES 匹配
  restart: always
  volumes:
    - elasticsearch:/usr/share/elasticsearch/data
  environment:
    - discovery.type=single-node
    - "ES_JAVA_OPTS=-Xms128m -Xmx128m"
```

参数说明：

- `discovery.type=single-node`：单节点模式，适合个人或小型部署。
- `ES_JAVA_OPTS`：限制 ElasticSearch JVM 的内存占用，根据实际机器内存酌情调整。
- `elasticsearch` 卷用于持久化数据，避免容器重启后索引丢失。

修改完成后，重新启动相关服务：

```bash
docker compose up -d elastic_search
```

确认容器正常运行后再进行下一步。


## 安装 ElasticSearch 插件

为了支持：

- **二进制文件内容解析**（如 PDF、Office 文档）

- **中文分词**

需要为 ElasticSearch 安装以下插件：

1. `ingest-attachment`：用于解析附件内容（通过 Tika）。
2. `analysis-ik`：提供中文分词能力（IK 分词器）。

在 ElasticSearch 容器中执行：

```bash
# 进入 ElasticSearch 容器（示例名自行替换）
docker exec -it elastic_search bash

# 安装附件解析插件
elasticsearch-plugin install ingest-attachment

# 安装 IK 分词器
elasticsearch-plugin install \
  https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.17.6/elasticsearch-analysis-ik-7.17.6.zip
```

安装完两个插件后，重启 ElasticSearch 容器：

```bash
exit
docker restart elastic_search
```

提示：插件版本必须与 ElasticSearch 版本对应，升级 ES 前先确认插件是否有对应版本。


## 在 Nextcloud 中启用 Full Text Search 相关应用

登录 Nextcloud Web 管理界面，使用管理员账号操作：

1. 打开 **应用（Apps）** 页面。

2. 搜索并安装以下三个应用：

   - **Full text search**
   - **Full text search – Elasticsearch Platform**
   - **Full text search – Files**

3. 安装完成后，在 Nextcloud 的设置中，检查 Full Text Search 的配置是否指向你的 ElasticSearch 服务（地址、端口等，根据自己的部署填写）。

这一步的目标是：
让 Nextcloud 知道**全文检索由谁来做**，并建立与 ElasticSearch 的连接。


## 使用 occ 建立全文索引

应用安装好之后，需要让 Nextcloud 把现有文件喂给 ElasticSearch，建立初始索引。

在运行 Nextcloud 的应用容器中执行 `occ` 命令，例如：

```bash
docker exec -u www-data next_cloud-app-1 \
  php ./occ fulltextsearch:index
```

参数说明：

- `-u www-data`：以 Web 服务器用户身份执行，避免权限问题（具体用户视镜像而定）。
- `next_cloud-app-1`：为 Nextcloud 应用容器名，请按实际名称替换。
- `fulltextsearch:index`：触发全文索引构建。

首次构建索引时，耗时取决于：文件数量、文件大小、文件类型以及机器性能。之后可以按需定期重建或增量更新（比如通过 cron 或计划任务）。


## 解决 `occ fulltextsearch:index` NULL Pointer 报错

在执行 `occ fulltextsearch:index` 时，可能会遇到类似：

> NULL Pointer on `isGloba`（`isGlobal`）

该问题与 `MountPoint` 类的属性为空处理不当有关，需要对相关 PHP 文件做一点小修复。

### 1. 问题文件位置

报错通常来自 `MountPoint.php`（具体路径可能随版本变化，可参考 Issue：
[https://github.com/nextcloud/files_fulltextsearch/issues/125](https://github.com/nextcloud/files_fulltextsearch/issues/125)）

### 2. 修复方式（补丁）

将原始文件按下面的补丁进行修改（仅示意关键差异）：

```diff
--- MountPoint.orig.php	2021-07-11 12:14:05.690177923 +0000
+++ MountPoint.php	2021-07-11 12:13:30.439026756 +0000
@@ -100,7 +100,7 @@
  	 * @return bool
  	 */
  	public function isGlobal(): bool {
--		return $this->global;
++		return $this->global === TRUE;
  	}

  	/**
@@ -119,7 +119,7 @@
  	 * @return array
  	 */
  	public function getGroups(): array {
--		return $this->groups;
++		return $this->groups ? $this->groups : array();
  	}

  	/**
@@ -138,7 +138,7 @@
  	 * @return array
  	 */
  	public function getUsers(): array {
--		return $this->users;
++		return $this->users ? $this->users : array();
  	}
```

核心改动有两点：

1. `isGlobal()`：

原来直接返回 `$this->global`，当属性为 `null` 时会导致问题。
修改为只在值为 `TRUE` 时才返回 `true`，其它情况为 `false`。

2. `getGroups()` / `getUsers()`：

原来直接返回 `$this->groups` / `$this->users`，可能为 `null`。
修改为：如果为空则返回空数组 `array()`，保证调用方拿到的一定是数组。

修改保存后，重新执行索引命令：

```bash
docker exec -u www-data next_cloud-app-1 \
  php ./occ fulltextsearch:index
```

如果不再报 NULL Pointer，说明修复生效。


## 参考资料

- Docker + NextCloud + OnlyOffice + ElasticSearch 搭建示例（CSDN）：
  [https://blog.csdn.net/qq_32014795/article/details/121612093](https://blog.csdn.net/qq_32014795/article/details/121612093)

- Nextcloud + ElasticSearch Full Text Search 详细教程：
  [https://fariszr.com/en/nextcloud-fulltextsearch-elasticsearch-docker-setup/](https://fariszr.com/en/nextcloud-fulltextsearch-elasticsearch-docker-setup/)
