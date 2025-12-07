---
layout: post
title:  "连接 Raspberry Pi 到 Bluemix IOT 服务"
date:   2017-03-20 14:37:00 +0800
categories: IOT
tags: [Raspberry Pi, Bluemix]
comments: true
---

这篇文章带你一步步把 Raspberry Pi 接入 IBM Bluemix 的物联网服务，让小板子也能把自己的状态实时“上报到云端”。


## 必要的软件包安装

如果你想顺便看看 Raspberry Pi 的 GPIO 状态，可以先安装一下 `wiringPi`（可选，但挺方便）。

简单流程如下：

```bash
$ git clone git://git.drogon.net/wiringPi  # 获取 wiringPi 源代码
$ cd wiringPi
$ ./build                                  # 编译并安装
$ gpio readall                             # 安装完成后查看各 GPIO 引脚状态
````

这一步不是必须的，只是让后面调试更直观。

接下来是连接 Bluemix 真正需要的部分——安装 IOT 库。

在 Raspberry Pi 上执行：

```bash
$ curl -LO https://github.com/ibm-messaging/iot-raspberrypi/releases/download/1.0.2/iot_1.0-1_armhf.deb  # 下载安装包
$ sudo dpkg -i iot_1.0-1_armhf.deb                                  # 安装 IOT 软件
$ service iot getdeviceid                                            # 获取设备 ID
The device ID is XXXXXXXXXXXXXXXX
...
```

记一下这里输出的 `device ID`，后面在 Bluemix 上添加设备时会用到。

## 在 Bluemix 上创建 IOT 服务

接下来切换到 IBM Bluemix 平台，创建对应的物联网服务。

1. 新建一个 **“Internet of Things”** 服务。
2. 在 **App 状态** 中选择 `Leave Unbound`。
3. 在 **Service name** 填写一个容易记的名字，例如：`your-service-name`。

创建完成后，进入服务页面，点击 **`Launch Dashboard`** 打开物联网仪表盘。

## 添加 Raspberry Pi 设备

在 Dashboard 中：

1. 切换到 **`Device`** 标签页。
2. 点击 **`Add Device`**，开始添加一个新设备。
3. 按向导填写设备相关信息（包括刚才拿到的 `device ID` 等），然后点击 **`Continue`**。

完成后，Bluemix 会给出一段配置内容，大致长这样：

```text
org=your_org_name
id=your_device_id
type=apikey
auth-key=your_auth_key
auth-token=your_auth_token
```

这些字段就是 Raspberry Pi 连到 Bluemix 所需要的认证信息。

将以上内容写入本机配置文件：

```bash
sudo nano /etc/iotsample-raspberrypi/device.cfg
```

把那几行配置粘进去，保存退出即可。

## 启动 IOT 服务并查看效果

最后，在终端中执行：

```bash
sudo service iot restart
```

重启 `iot` 服务。

如果一切配置正确，你就可以在 Dashboard 的 **Device** 标签页里看到这块 Raspberry Pi 的实时状态信息了，例如：

* CPU 温度
* 内存占用
* 其他运行状态……

到这里，Raspberry Pi 到 Bluemix IOT 服务的基础连接就打通了。后面你可以在此基础上继续接入自己的传感器、业务逻辑或可视化应用，让这块小板子为你的物联网项目真正“开口说话”。 🙂
