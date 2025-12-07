---
layout: post
title: "GeoServer 内存优化策略"
date: 2024-06-23 00:00:00 +0800
categories: GIS
tags: ["GeoServer", "优化"]
comments: true
---

折腾 GeoServer 这几年，我踩过最多的坑，大概就是“内存不够用”，
不是 OOM，就是频繁 Full GC，把整台机器拖得跟没吃早饭一样虚。

回头看，其实很多问题并不在于 GeoServer 本身“性能差”，而是我一开始既没照顾好栅格数据的组织方式，
也没给各种请求设好“安全阀”，更不了解 Java 渲染那点门道。

所以这篇笔记，算是我在一线环境里被打脸无数次之后，总结出来的一套“能立刻上手”的内存优化思路：
从启用 Marlin 光栅化引擎，到给 WPS 限流，再到给大 GeoTIFF 做切片和金字塔，
以及用 Control Flow 把整体并发拎起来管一管。

如果你也在为 GeoServer 时不时抽风、内存时不时飙满而头大，希望这篇文章能帮你少走一点弯路。


## 运行环境概览

当前 GeoServer 运行在 Java 11 环境下：

```bash
openjdk version "11.0.19" 2023-04-18
OpenJDK Runtime Environment (build 11.0.19+7-post-Ubuntu-0ubuntu122.04.1)
OpenJDK 64-Bit Server VM (build 11.0.19+7-post-Ubuntu-0ubuntu122.04.1, mixed mode, sharing)
```

在这个基础上，我们主要从以下几个方向做内存优化：

1. 使用更高效的 Marlin 光栅化引擎
2. 限制 WPS 服务的并行度与输入文件大小
3. 针对大体量 GeoTIFF 做数据层面的切片与金字塔优化
4. 配置 Control Flow，控制整体并发与单用户并发

下面逐一展开。

---

## 使用 Marlin Rasterizer 优化渲染内存

GeoServer 在绘制矢量数据、生成地图图片时，会大量用到 Java2D 渲染。
默认渲染器在复杂几何、大量要素叠加时，既耗 CPU，也耗内存。

通过启用 **Marlin** 光栅化引擎，可以在复杂地图渲染场景下更节省内存、提高性能。

### 配置方式（示例：Tomcat 部署）

在 Tomcat 的 JVM 启动参数中加入：

```bash
-Xbootclasspath/a:/opt/apache-tomcat-9.0.75/lib/marlin.jar \
-Dsun.java2d.renderer=sun.java2d.marlin.DMarlinRenderingEngine
```

**说明：**

- `marlin.jar` 需要放在容器可访问的路径（示例为 Tomcat `lib` 目录）。
- `Dsun.java2d.renderer` 指定使用 Marlin 作为渲染引擎。

### 内存优化效果

更高效的几何处理与填充算法，减少渲染过程中的临时对象创建与垃圾回收压力，
对复杂线、面要素（如河网、道路网）有更好的性能表现，
间接降低 GeoServer 在高密度地图渲染任务中的内存峰值。

---

## 控制 WPS 服务资源消耗

WPS（Web Processing Service）通常会执行一些计算量较大、过程较长的任务，
如果不加控制，很容易把内存吃满，甚至拖垮整个 GeoServer 实例。

### 限制并行任务数量

通过 GeoServer 管理界面，可以为 WPS 设置**最大并行任务数**，避免一次性跑太多大作业。

**建议：**

根据机器 CPU 核数、内存大小，适度保守设置。

典型做法：`最大并行 WPS 任务数 < CPU 核数`，并逐步压测微调。

### 限制输入文件大小

对于支持文件上传/处理的 WPS 过程，需要限制单个输入文件的最大体积，防止用户/调用方一次性上传超大文件导致内存爆。

**建议：**

根据典型业务使用场景设置一个合理上限（例如几十 MB 级），超出则直接拒绝。

对于必须处理超大数据的场景，优先走**离线预处理 + 发布数据服务**的方式，而不是直接扔给 WPS 在线计算。

---

## 针对 GeoTIFF 的数据层优化

GeoServer 读取和渲染栅格数据，除了程序本身的配置，**数据文件组织方式**对内存和性能影响非常大。
这里重点针对大体量 GeoTIFF 的优化策略。

### 几百 MB 级 GeoTIFF：Inner Tiling + Overviews

对于体量在“几百兆”级别的 GeoTIFF，官方推荐使用：

- **内部瓦片（Inner tiling）**
- **多级 Overviews（金字塔）**

#### 使用 Inner Tiling

默认情况下，GeoTIFF 往往以“条带（strip）”方式存储。改为 tile 方式后，可按瓦片读取目标区域：
当客户端请求某个小区域时，GeoServer 只需读取覆盖该区域的瓦片，而不是扫过大段无关数据，从而减少 IO 和内存峰值。

使用 `gdal_translate` 转为瓦片存储：

```bash
gdal_translate -of GTiff -co TILED=YES source.tif tiled.tif
```

**好处：**

1. 缩小单次读取的栅格块大小
2. 减少无效数据加载
3. 降低 GeoServer 处理某一个地图请求时的内存占用

#### 为 GeoTIFF 生成 Overviews（内嵌金字塔）

Overviews 是嵌入原始 GeoTIFF 的“低分辨率版本”，
客户端在缩放较小、全图浏览时，不必每次都读取原始分辨率栅格。

```bash
gdaladdo -r average tiled.tif 2 4 8 16
```

这里生成了多级金字塔：1/2、1/4、1/8、1/16 分辨率。

**好处：**

1. 浏览小比例尺地图时，优先读取低分辨率栅格，显著减小单次请求的数据量
2. 降低 GeoServer 内部重采样和内存消耗
3. 提升缩放/平移时的交互体验

---

### 几 GB 级超大 GeoTIFF

当文件体量达到“几个 G”时，即使做了内部瓦片和 Overviews，单文件仍然过大。

这种情况下，推荐为不同级别建立**平铺金字塔**（目录级金字塔 + mosaic）。

目录结构示例：

```text
rootDirectory
    +- pyramid.properties
    +- 0
       +- mosaic metadata files
       +- mosaic_file_0.tiff
       +- ...
       +- mosiac_file_n.tiff
    +- ...
    +- 32
       +- mosaic metadata files
       +- mosaic_file_0.tiff
       +- ...
       +- mosiac_file_n.tiff
```

常见做法：

每个目录代表一个缩放级别（如 0、2、4、8、16、32 …），
每级内部再用瓦片化的小 GeoTIFF 组成 mosaic，
GeoServer 通过 ImageMosaic/金字塔机制按需读取对应级别的小文件。

**优点：**

1. 单个文件不至于巨大，利于缓存与重建
2. 请求对应某个缩放级别时，只访问这一层的瓦片，内存占用更可控
3. 有利于集群环境下的分片存储与扩展

---

## 配置 Control Flow：给 GeoServer 加“安全阀”

Control Flow 插件可以限制不同类型请求的并发数量，
相当于给 GeoServer 加了一层“流量闸门”，防止瞬间高并发把内存、CPU 一次性打满。

示例配置：

```properties
# 如果请求在队列中等待超过 60 秒，就不再执行（客户端大概率已经超时放弃）
timeout=60

# 全局：最多允许 32 个请求并行
ows.global=32

# 针对 WMS GetMap：最多允许 16 个并发
ows.wms.getmap=16

# 针对内存敏感的 Excel 导出：最多允许 4 个并发
ows.wfs.getfeature.application/msexcel=4

# 针对单个用户：最多允许 4 个并发请求
#（当时 Firefox 默认并发为 6，这里略低一些，避免单用户“刷爆”服务）
user=4

# 针对 GWC 切片请求：最多允许 4 个并发
#（经验：吞吐峰值通常在 核心数 x 4 左右，可按机器配置调整）
ows.gwc=4
```

### 内存优化思路

**限制全局并发**：控制整体资源占用上限，避免“人人都能抢到一点内存，最后谁也跑不完”。

**区分请求类型**：

  - 地图渲染（GetMap）一般较耗 CPU 和内存
  - 大文件输出（如 Excel）高度内存敏感
  - 切片生成（GWC）又是另一类计算/IO 模式

**限制单用户并发**：防止单个用户（或脚本）发起大量并发请求，拖垮整个服务。

---

## 实际效果对比（示意）

在应用上述策略前后，可以从以下几个维度感受到明显差异：

- 内存使用峰值降低
- OOM 和频繁 Full GC 情况减少
- 缩放/浏览大影像时，响应速度更稳定
- 高并发访问下，整体服务更“抗揍”

---

## 实施建议与排查思路

如果你准备在生产环境中落地这些优化，建议循序渐进：

1. **先改数据，再调服务**

   - 优先对 GeoTIFF 做 Inner tiling + Overviews
   - 超大数据集使用平铺金字塔和 mosaic

2. **再启用 Marlin + Control Flow**

   - Marlin 基本是“低风险，高收益”，可以优先尝试
   - Control Flow 根据压测逐步调参，不要一下子收得太紧

3. **最后再收紧 WPS 限制**

   - 限制并行度和输入文件大小
   - 对确有需求的大任务，考虑单独部署专用 WPS 实例或改成离线任务

4. **结合监控工具观察效果**

   - 关注 JVM 堆内存、GC 次数与耗时
   - 关注 GeoServer 响应时间、错误率
   - 有条件的话，对关键图层/服务做定期压测，对比优化前后的曲线
