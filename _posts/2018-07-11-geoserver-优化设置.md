---
layout: post
title: "Geoserver 优化配置"
date: 2018-07-11 10:56:00 +0800
categories: Code
tags: [GIS]
comments: true
---

在把 GeoServer 跑进容器、接到线上环境之后，大家往往会先关注功能是否可用，但真正影响体验的，往往是那些“看不见”的细节：

- JRE 版本兼容不兼容？
- JAI 和 ImageIO 装没装对？
- JVM 和服务策略是不是在帮你省内存、提性能，还是在拖后腿？

下面这份笔记，就从运行环境（JRE）、图像处理组件（JAI/ImageIO）、容器 JVM 调优、服务配置以及数据组织方式 几个方面，汇总了一些在 GeoServer 调优时常用、但又容易被忽略的要点，方便自己按需查阅和逐步优化。

## GeoServer 支持的 JRE 版本

- **Java 8**：GeoServer 2.9.x 及以上版本（已在 OpenJDK 和 Oracle JRE 上测试）
- **Java 7**：GeoServer 2.6.x ~ 2.8.x（已在 OpenJDK 和 Oracle JRE 上测试）
- **Java 6**：GeoServer 2.3.x ~ 2.5.x（已在 Oracle JRE 上测试）
- **Java 5**：GeoServer 2.2.x 及更早版本（已在 Sun JRE 上测试）


## 使用 JAI 和 ImageIO 插件

为 Docker 镜像安装原生 JAI 和 ImageIO 扩展。

**Java Advanced Imaging API（JAI）** 是 Oracle 提供的高级图像处理库。GeoServer 依赖 JAI 处理栅格覆盖数据（coverages），并在生成 WMS 输出时大量用到它。JAI 的性能对所有栅格处理都非常关键，而这些操作在 WMS 和 WCS 中会被频繁使用，例如重采样、裁剪以及重投影栅格数据。

**Java Image I/O Technology（ImageIO）** 用于读取和写入栅格数据。
ImageIO 会影响 WMS 和 WCS 对栅格数据的读取；即便没有栅格数据，WMS 在输出 PNG/GIF/JPEG 等图片时也需要依赖 ImageIO 进行编码。

---

## 针对容器的优化

### JAVA_OPT 设置

`jvm` 参数示例：

- `-Xms128m`：初始堆内存大小 128M
- `-Xmx756M`：最大堆内存大小 756M
- `-XX:SoftRefLRUPolicyMSPerMB=36000`
- `-XX:+UseParallelGC`
- `--XX:+UseParNewGC`
- `–XX:+UseG1GC`

> 注：以上为示例参数，实际使用时需根据容器资源进行调整。

### 启用 Marlin 光栅化器（Marlin rasterizer

`jvm` 参数示例：

- `-Xbootclasspath/a:$MARLIN_JAR`
- `-Dsun.java2d.renderer=org.marlin.pisces.MarlinRenderingEngine`

---

## 针对配置的优化

### 设置服务策略（Service Strategy）

通过修改 GeoServer 实例的 `web.xml` 来设置服务策略。

文件路径示例：

```text
geoserver/src/web/app/src/main/webapp/WEB-INF/web.xml
```

可选策略说明：

| Strategy           | 描述                                                                                                                              |
| :----------------- | :-------------------------------------------------------------------------------------------------------------------------------- |
| **SPEED**          | 立即开始返回输出。该策略速度最快，但通常不会返回完整规范的 OGC 错误信息。                                                         |
| **BUFFER**         | 将完整结果先缓存在内存中，待生成完成后再返回。这可以保证 OGC 错误信息完整，但会明显增加响应延迟；如果响应非常大，还可能耗尽内存。 |
| **FILE**           | 和 BUFFER 类似，但会将结果写入文件而不是内存。比 BUFFER 慢一些，但可以避免内存问题。                                              |
| **PARTIAL-BUFFER** | 在 BUFFER 和 SPEED 之间折中：先在内存中缓存几 KB 的响应，再开始向客户端输出完整内容。                                             |

### 配置服务限制（Configure service limits）

- 为每个 WFS `GetFeature` 请求设置返回要素数量上限
  - 也可以按具体要素类型进行配置，直接修改对应的 `info.xml` 文件

- 为 WMS 请求设置资源限制，避免单次请求占用过多内存或时间

### 缓存数据（Cache your data）

通过合理的数据缓存策略，减少重复计算和 IO，提升整体响应性能。

---

## 针对数据的优化

### 使用外部数据目录（Use an external data directory）

将 GeoServer 的数据目录放在容器外部或独立存储中，便于升级、迁移和备份。

### 使用空间数据库（Use a spatial database）

将矢量数据存储在空间数据库（如 PostGIS）中，一般能获得更好的查询和渲染性能。

### 选择性能更好的覆盖数据格式（Pick the best performing coverage formats）

#### 选择合适的格式（Choose the right format）

- **面向数据交换设计的格式**：如 GeoTIFF、ECW、JPEG 2000 和 MrSID 等。
- **更适合渲染和服务的数据格式**：如 ArcGrid 等。

#### 为 Geotiff 配置高性能渲染（Setup Geotiff data for fast rendering）

- 启用 **内部切片（inner tiling）**
- 配置多级 **overviews（缩略金字塔）**

#### 处理超大数据集（Handling huge data sets）

使用 **ImagePyramid 插件** 来管理和服务大规模栅格数据。
