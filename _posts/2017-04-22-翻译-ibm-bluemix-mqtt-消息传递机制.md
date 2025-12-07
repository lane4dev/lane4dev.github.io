---
layout: post
title:  "IBM Bluemix MQTT 消息传递机制"
date:   2017-04-22 14:37:00 +0800
categories: IOT
tags: [MQTT, Bluemix, Translate]
comments: true
---

最后更新：2017-03-14  [在GitHub上编辑](https://github.com/IBM-Bluemix/docs/blob/master/services/IoT/reference/mqtt/index.md)

MQTT是设备和应用程序用于与IBM Watson™IoT平台通信的主要协议。它是一种发布和订阅消息的传输协议，旨在有效交换传感器和移动设备之间的实时数据。

MQTT可以通过TCP / IP运行，并且可以直接对TCP / IP进行编码，还可以选择使用一个库来处理详细的MQTT协议信息。有各种MQTT客户端库可以使用。IBM帮助开发和提供支持了一些客户端库，包括以下站点中可用的库：  

[MQTT community wiki](https://github.com/mqtt/mqtt.github.io/wiki)
[Eclipse Paho project](http://eclipse.org/paho/)


## 版本支持

想了解有关Watson IoT Platform支持的MQTT版本的信息，请参阅 [Standards and requirements](https://console.ng.bluemix.net/docs/services/IoT/reference/standards_and_requirements.html#mqtt)。

## 应用程序，设备和网关客户端

在Watson IoT平台中，主要的是设备和应用。网关是设备的子类。

您的MQTT客户端将自己标识为Watson IoT Platform的服务。该服务决定了客户端连接时的功能。同时也决定了客户端认证的机制。

应用程序和设备与不同的MQTT主题域名配合使用。设备在设备范围的主题域名内工作，而应用程序可以完整访问整个组织的主题域名。有关详细信息，请参阅以下主题：

[MQTT messaging for devices](https://console.ng.bluemix.net/docs/services/IoT/devices/mqtt.html)

[MQTT messaging for applications](https://console.ng.bluemix.net/docs/services/IoT/applications/mqtt.html)

[MQTT messaging for gateways](https://console.ng.bluemix.net/docs/services/IoT/gateways/mqtt.html)

## 信息保留

Watson IoT平台为MQTT消息传递的保留消息功能提供了有限的支持。如果在从设备，网关或应用程序发送到Watson IoT Platform的MQTT消息中将保留的消息标志设置为true，则将该消息作为未保留的消息处理。Watson IoT平台机构无权发布保留的消息。当Watson IoT Platform服务设置为true时，它将覆盖保留的消息标志，并处理消息，就像保留的消息标志设置为false一样。

## Quality of service 等级

MQTT协议提供三种服务质量，用于在客户端和服务器之间传递消息：“最多一次”，“至少一次”和“一次”。当您可以使用服务质量级别发送事件和命令时，您必须仔细考虑适合您的需要的服务级别。服务质量“2”并不总是比“0”级更好的选择。

### 最多一次（QoS0）

“最多一次”服务质量水平（QoS0）是最快的传输模式，有时被称为“触发然后忘记”。消息最多只能传递一次，或者可能根本不会传送。通过网络传送不需要被接收方确认送达，并且该消息未被存储。如果客户端断开连接或服务器发生故障，则该消息可能会丢失。

MQTT协议不需要服务器将服务质量“0”的发布转发给客户端。如果客户端在服务器接收到发布时断开连接，则发布可能会被丢弃，具体取决于服务器实现。

提示：在间隔发送实时数据时，请使用服务质量等级0。如果单个消息丢失，则不重要，因为包含较新数据的另一个消息将在不久之后发送。在这种情况下，使用更高质量服务的额外成本并没有产生任何实质的好处。

### 至少一次（QoS1）

当服务质量等级为1（QoS​​1），消息总是至少传递一次。如果在发送方收到确认之前发生故障，则可以多次发送消息。消息必须在本地存储在发送方，直到发送方收到由接收方发布的确认消息。该消息被存储，以防再次发送消息。

### 仅仅一次（QoS2）

“仅仅一次”的服务质量2级（QoS2）是最安全，但最慢的传输方式。该消息总是传递一次，并且还必须在存储在本地发送方，直到发送方接收到由接收方发布的确认消息。该消息被存储，以防再次发送消息。使用服务质量2级，使用比第1级更复杂的握手和确认顺序，以确保消息不被重复。

提示：发送命令时，如果您希望确认只有指定的命令将被执行，并且只会执行一次操作，请使用服务质量级别2。这是一个2级服务质量的额外开销比其他级别更有利的例子。

## 缓冲区订阅和会话清理

来自设备或应用程序的每个订阅都会分配5000个消息的缓冲区。缓冲区允许任何应用程序或设备落后于正在处理的实时数据，并且还为每个订阅创建多达5000个未决消息的积压。当缓冲区已满，收到新消息时，最旧的消息将被丢弃。

使用MQTT clean session选项访问订阅缓冲区。当会话清理设置为false时，订阅者从缓冲区接收消息。当会话清理设置为true时，缓冲区被重置。

注意：无论使用的服务质量设置如何，都可以使用订阅缓冲区限制。使用1级或2级发送的消息可能不会传递到无法跟上其所做的订阅的消息速率的应用程序。

## 消息有效载荷限制

Watson IoT平台支持任何以MQTT标准格式发送和接收的消息。MQTT是数据无关的，所以可以以任何编码，加密数据、图像、文本——几乎每种类型的数据都以二进制格式发送。但是，具体使用时有一些限制。

Watson IoT平台上的消息有效载荷有大小限制。

## 消息有效载荷格式限制

消息有效载荷可以包含任何有效的字符串，但是JSON（“json”），text（“text”）和binary（“bin”）格式比其他格式类型更常用。

下表概述了不同格式类型的消息有效载荷限制：

| 有效载入格式 | 具体用例指南                                   |
| :----: | :--------------------------------------- |
|  JSON  | JSON是Watson IoT Platform的标准格式。如果您打算使用内置的Watson IoT平台仪表板，板卡和分析，请确保消息有效载荷格式符合格式正确的JSON文本。 |
|   文字   | 使用有效的UTF-8字符编码。                          |
|  二进制   | 无限制                                      |

## 最大消息有效载荷大小

重要提示：Watson IoT Platform的最大有效载荷大小为131072个字节。有效载荷大于限制的消息将被拒绝。连接客户端也会断开连接，并在诊断日志中显示一条如下的消息：

``` text
Closed connection from x.x.x.x. The message size is too large for this endpoint.
```

## MQTT 保持活跃间隔

MQTT 保持活动间隔（以秒为单位）测量，定义了客户端和代理之间通信的最大时间。MQTT客户端必须确保在没有与 Broker 的任何其他通信的情况下，发送PINGREQ数据包。保持活跃间隔允许客户端和代理检测到网络发生故障，导致连接中断。而无需等待TCP / IP超时时间。

如果您的Watson IoT Platform MQTT客户端使用共享订阅，则保持活动间隔值只能设置在1到3600秒之间。如果请求值为0或大于3600的值，则Watson IoT Platform代理将keep alive间隔设置为3600秒。

----

原文地址：[MQTT messaging](https://console.ng.bluemix.net/docs/services/IoT/reference/mqtt/index.html#ref-mqtt)
