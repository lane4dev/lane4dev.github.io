---
layout: post
title:  " Docker Cheat Sheet"
date:   2017-07-26 09:00:00 +0800
categories: Code
tags: [Docker]
banner:
  image: /assets/images/2017-07/docker.png
comments: true
---

Docker 是一个开放平台，帮助开发和运维在笔记本、数据中心虚拟机、甚至云环境里，以统一的方式构建、打包和运行应用。

这篇文章是一份 **Docker 命令速查表**：方便我自己忘了某条命令的时候快速翻一眼就能用。

---

## 安装 Docker（Installation）

先把 Docker 装起来，后面所有命令才有用。不同系统的安装方式略有差异。

### Linux

```bash
curl -sSL https://get.docker.com/ | sh
````

> 使用官方脚本一键安装 Docker。
> 用于本地开发很方便，生产环境建议优先参考各发行版的官方指南。

### Mac

下载 Docker Desktop 的 dmg 安装包：

[下载 Docker for Mac](https://download.docker.com/mac/stable/Docker.dmg)

### Windows

使用 MSI 安装包：

[下载 Docker for Windows](https://download.docker.com/win/stable/InstallDocker.msi)

---

## 镜像仓库与镜像仓库操作（Docker Registries & Repositories）

这一部分是和镜像仓库打交道：登录、搜索、拉取、推送等日常操作。

### 登录到镜像仓库（Login to a Registry）

```bash
docker login
docker login localhost:8080
```

### 退出镜像仓库（Logout from a Registry）

```bash
docker logout
docker logout localhost:8080
```

### 搜索镜像（Searching an Image）

```bash
docker search nginx
docker search nginx --stars=3 --no-trunc busybox
```

> 可以通过星级、是否截断等参数做简单筛选。

### 拉取镜像（Pulling an Image）

```bash
docker pull nginx
docker pull eon01/nginx localhost:5000/myadmin/nginx
```

### 推送镜像（Pushing an Image）

```bash
docker push eon01/nginx
docker push eon01/nginx localhost:5000/myadmin/nginx
```

> 推送前记得先登录对应 registry，并配置好 tag。

---

## 运行容器（Running Containers）

这里是一组把镜像“变成跑起来的容器”的基本操作。

### 创建容器（但不启动）（Creating a Container）

```bash
docker create -t -i eon01/infinite --name infinite
```

> 只创建，不马上运行。适合先创建好容器，再手动调整配置或启动。

### 直接运行容器（Running a Container）

```bash
docker run -it --name infinite -d eon01/infinite
```

> 最常用的一条：基于镜像新建并启动容器。
>
> * `-it`：交互式终端
> * `-d`：后台运行

### 重命名容器（Renaming a Container）

```bash
docker rename infinite infinity
```

### 删除容器（Removing a Container）

```bash
docker rm infinite
```

### 更新容器资源限制（Updating a Container）

```bash
docker update --cpu-shares 512 -m 300M infinite
```

> 运行中也可以调整容器的 CPU / 内存限制。

---

## 启动与停止容器（Starting & Stopping Containers）

日常使用中，最常用的就是这一组启停命令。

### 启动（Starting）

```bash
docker start nginx
```

### 停止（Stopping）

```bash
docker stop nginx
```

### 重启（Restarting）

```bash
docker restart nginx
```

### 暂停（Pausing）

```bash
docker pause nginx
```

### 恢复（Unpausing）

```bash
docker unpause nginx
```

### 阻塞等待容器退出（Blocking a Container）

```bash
docker wait nginx
```

> 等待容器退出，并返回退出码。常用于脚本中做流程控制。

### 发送 SIGKILL（Sending a SIGKILL）

```bash
docker kill nginx
```

> 直接“强杀”。比 `docker stop` 粗暴，除非必要，一般先用 `stop`。

### 附着到已有容器（Connecting to an Existing Container）

```bash
docker attach nginx
```

> 把当前终端“接”到正在运行的容器上，查看输出或交互。

---

## 查看容器信息（Getting Information about Containers）

想知道“有哪些容器在跑”“里面在干嘛”“资源占用如何”，就看这里。

### 查看运行中的容器（Running Containers）

```bash
docker ps
docker ps -a
```

### 查看容器日志（Container Logs）

```bash
docker logs infinite
```

> 排查问题时第一反应通常就是看日志。

### 检查容器详情（Inspecting Containers）

```bash
docker inspect infinite
docker inspect infinite --format '{{ .NetworkSettings.IPAddress }}' $(docker ps -q)
```

> `inspect` 会给出非常详细的 JSON 信息，可以配合 `--format` 提取需要的字段。

### 容器事件（Container Events）

```bash
docker events infinite
```

### 暴露端口（Public Ports）

```bash
docker port infinite
```

### 查看容器进程（Running Processes）

```bash
docker top infinite
```

### 容器资源使用情况（Container Resource Usage）

```bash
docker stats infinite
```

> 类似一个“docker 版的 top/htop”。

### 查看容器文件系统变更（Inspecting Changes to Files or Directories）

```bash
docker diff infinite
```

> 看看容器相对于镜像，哪些文件被新增、修改或删除。

---

## 镜像管理（Manipulating Images）

列出、构建、保存和删除镜像的常用命令都在这里。

### 列出镜像（Listing Images）

```bash
docker images
```

### 构建镜像（Building Images）

```bash
docker build .
docker build github.com/creack/docker-firefox
docker build - < Dockerfile
docker build - < context.tar.gz
docker build -t eon/infinite .
docker build -f myOtherDockerfile .
curl example.com/remote/Dockerfile | docker build -f - .
```

> Docker 支持多种构建上下文：本地目录、tar 包，甚至远程 Dockerfile。

### 删除镜像（Removing an Image）

```bash
docker rmi nginx
```

### 从文件或标准输入加载镜像（Loading a Tarred Repository）

```bash
docker load < ubuntu.tar.gz
docker load --input ubuntu.tar
```

### 将镜像保存为 tar（Saving an Image）

```bash
docker save busybox > ubuntu.tar
```

### 查看镜像历史（Showing the History of an Image）

```bash
docker history
```

### 从容器创建镜像（Creating an Image From a Container）

```bash
docker commit nginx
```

> 适用于“调试完一个临时容器，想把当前状态固化成镜像”的场景。

### 给镜像打标签（Tagging an Image）

```bash
docker tag nginx eon01/nginx
```

### 推送镜像（Pushing an Image）

```bash
docker push eon01/nginx
```

---

## 网络（Networking）

当你开始有多个容器、多个服务，需要它们“互相说话”时，就会用到这些命令。

### 创建网络（Creating Networks）

```bash
docker network create -d overlay MyOverlayNetwork
docker network create -d bridge MyBridgeNetwork
docker network create -d overlay \
  --subnet=192.168.0.0/16 \
  --subnet=192.170.0.0/16 \
  --gateway=192.168.0.100 \
  --gateway=192.170.0.100 \
  --ip-range=192.168.1.0/24 \
  --aux-address="my-router=192.168.1.5" --aux-address="my-switch=192.168.1.6" \
  --aux-address="my-printer=192.170.1.5" --aux-address="my-nas=192.170.1.6" \
  MyOverlayNetwork
```

### 删除网络（Removing a Network）

```bash
docker network rm MyOverlayNetwork
```

### 列出网络（Listing Networks）

```bash
docker network ls
```

### 查看网络详情（Getting Information About a Network）

```bash
docker network inspect MyOverlayNetwork
```

### 将运行中的容器接入网络（Connecting a Running Container）

```bash
docker network connect MyOverlayNetwork nginx
```

### 容器启动时指定网络（Connecting a Container When it Starts）

```bash
docker run -it -d --network=MyOverlayNetwork nginx
```

### 将容器从网络中断开（Disconnecting a Container）

```bash
docker network disconnect MyOverlayNetwork nginx
```

---

## 清理 Docker 环境（Cleaning Docker）

这一节大多是“重锤操作”，可以帮你快速清理环境，但也很容易删多了。尤其在生产环境，动手前务必三思。

### 删除运行中的容器（Removing a Running Container）

```bash
docker rm nginx
```

### 删除容器及其卷（Removing a Container and its Volume）

```bash
docker rm -v nginx
```

### 删除所有已退出容器（Removing all Exited Containers）

```bash
docker rm $(docker ps -a -f status=exited -q)
```

### 删除所有停止容器（Removing All Stopped Containers）

```bash
docker rm `docker ps -a -q`
```

### 删除镜像（Removing a Docker Image）

```bash
docker rmi nginx
```

### 删除悬空镜像（Removing Dangling Images）

```bash
docker rmi $(docker images -f dangling=true -q)
```

### 删除所有镜像（Removing all Images）

```bash
docker rmi $(docker images -a -q)
```

### 停止并删除所有容器（Stopping & Removing all Containers）

```bash
docker stop $(docker ps -a -q) && docker rm $(docker ps -a -q)
```

### 删除悬空卷（Removing Dangling Volumes）

```bash
docker volume rm $(docker volume ls -f dangling=true -q)
```

---

## Docker Swarm 简要速查（Docker Swarm）

最后是一小节 `docker swarm` 的命令，用于快速搭建和管理一个简单的 Docker 集群。

### 安装 Docker Swarm（Installing Docker Swarm）

```bash
curl -ssl https://get.docker.com | bash
```

### 初始化 Swarm（Initializing the Swarm）

```bash
docker swarm init --advertise-addr 192.168.10.1
```

### 获取 worker 加入集群的命令（Getting a Worker to Join）

```bash
docker swarm join-token worker
```

### 获取 manager 加入集群的命令（Getting a Manager to Join）

```bash
docker swarm join-token manager
```

### 查看服务（Listing Services）

```bash
docker service ls
```

### 查看节点（Listing Nodes）

```bash
docker node ls
```

### 创建服务（Creating a Service）

```bash
docker service create --name vote -p 8080:80 instavote/vote
```

### 查看 Swarm 任务（Listing Swarm Tasks）

```bash
docker service ps
```

### 扩缩容服务（Scaling a Service）

```bash
docker service scale vote=3
```

### 更新服务（Updating a Service）

```bash
docker service update --image instavote/vote:movies vote
docker service update --force --update-parallelism 1 \
  --update-delay 30s nginx
docker service update --update-parallelism 5 \
  --update-delay 2s --image instavote/vote:indent vote
docker service update --limit-cpu 2 nginx
docker service update --replicas=5 nginx
```
