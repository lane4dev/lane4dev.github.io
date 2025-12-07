---
layout: post
title:  "通过命令行设置 Raspberry Pi 无线上网"
date:   2017-02-14 14:37:00 +0800
categories: Hardware
tags: [Raspberry Pi]
comments: true
---

如果没法通过图形界面给 Raspberry Pi 配置 WiFi，或者手边压根没接显示器、也没有有线网，这篇教程就能派上用场。  

好消息是：**不用额外装软件**，所有需要的工具系统里都自带了，只要能进命令行就能搞定。

这篇教程介绍如何一步步用命令行把 Raspberry Pi 连上无线网络。


## 一、先把无线网络信息搞清楚

首先，第一步要弄清楚周围有哪些 WiFi、用的是什么加密方式，以及你要连的那一个网络叫什么。

在终端中运行：

```bash
sudo iwlist wlan0 scan
```

这条命令会扫描当前能看到的所有 WiFi，并打印一大串信息。有两项需要重点关注：

* `ESSID:"test"`：这里的 `test` 就是 WiFi 名称（SSID）。
* `IE: IEEE 802.11i / WPA2 Version 1`：表示使用的加密/认证方式。
  常见的是 **WPA** 或 **WPA2**。本教程对 WPA 和 WPA2 都适用，不过 **WPA2 Enterprise**（企业级认证）不在本文范围内。

你还需要知道这个 WiFi 的密码，对大多数家用路由器来说，密码通常写在机身背面的贴纸上。

在下面的例子里，我们假设：

* 无线网络名（`ESSID` / `ssid`）：`testing`
* 密码（`psk`）：`testingPassword`


## 二、用 `wpa_passphrase` 生成安全配置

如果不想在配置文件里直接写明文密码，可以用 `wpa_passphrase` 生成一段加密后的 PSK。

示例命令：

```bash
wpa_passphrase "testing"
```

执行后，树莓派的终端会让你输入密码，然后输出一段可以直接用的配置片段，比如：

```text
network={
    ssid="testing"
    #psk="testingPassword"
    psk=131e1e221f6e06e3911a2d11ff2fac9182665c004de85300f9cac208a6a80531
}
```

* `ssid`：网络名称
* `#psk="testingPassword"`：注释里保留了明文密码
* `psk=...`：真正用于连接的十六进制 PSK

这里可以把这段配置复制到 `wpa_supplicant` 的配置文件里，或者直接让 `wpa_passphrase` 把输出写入文件：

```bash
wpa_passphrase "testing" "testingPassword" >> /etc/wpa_supplicant/wpa_supplicant.conf
```

注意以下几点：

* 这条命令需要有写入配置文件的权限，可以先 `sudo su` 切到 root。

* 或者用 `sudo` 和 `tee` 来追加写入，而不用切换用户：

  ```bash
  wpa_passphrase "testing" "testingPassword" | \
    sudo tee -a /etc/wpa_supplicant/wpa_supplicant.conf > /dev/null
  ```

* 一定要用 `>>` 或 `tee -a` **追加写入**，不要用 `>`，不然会把原配置文件整个覆盖掉。

* 如果你有一个纯文本文件里存着密码，也可以用重定向把它喂给 `wpa_passphrase`：

  ```bash
  wpa_passphrase "testing" < password.txt
  ```

处理完记得删除明文密码文件，避免安全隐患。


## 三、把网络信息写入 Raspberry Pi 配置

Raspberry Pi 使用 `wpa_supplicant` 来管理无线网络。因此需要编辑它的配置文件：

```bash
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
```

把光标移动到文件末尾，增加一个 `network` 段，例如：

```text
network={
    ssid="The_ESSID_from_earlier"
    psk="Your_wifi_password"
}
```

把上面的值替换成你自己的 WiFi 名称和密码，比如前面的示例网络，就可以写成：

```text
network={
    ssid="testing"
    psk="testingPassword"
}
```

> 说明：
>
> * 如果直接使用明文密码，保持 `psk="..."` 这种形式即可。
> * 如果使用 `wpa_passphrase` 生成的十六进制 `psk=...`，也可以按生成结果粘贴使用，不必再加引号。

编辑完按下：

1. `Ctrl + X` 退出
2. 按 `Y` 保存
3. 回车确认文件名

通常几秒钟内，`wpa_supplicant` 会自动加载新配置并尝试连接 WiFi。

如果没有动静，可以手动让它重新加载：

```bash
sudo wpa_cli reconfigure
```


## 四、如何确认 Raspberry Pi 已连上 WiFi？

无论你是手动写的 `network` 段，还是用 `wpa_passphrase` 生成的配置，都可以用下面的方式确认连接情况：

```bash
ifconfig wlan0
```

如果在输出中看到类似：

```text
inet addr:192.168.1.23
```

说明 `wlan0` 已经拿到 IP 地址，Raspberry Pi 成功连上无线网络。

如果没有 `inet addr` 字段，通常是没连上。请按顺序检查：

1. `ssid` 是否拼写正确（大小写也必须一致）
2. 密码是否输错
3. 路由器是否限制了接入设备（比如 MAC 地址过滤等）


## 五、连接不需要密码的网络

有些场景（例如某些开放网络或临时热点）可能不需要密码，这时 `wpa_supplicant` 配置里要显式声明不使用密钥管理：

```text
network={
    ssid="testing"
    key_mgmt=NONE
}
```

保存后，按前面的方法 `sudo wpa_cli reconfigure`，再通过 `ifconfig wlan0` 检查是否获得 IP 地址。


## 六、连接隐藏 SSID 的网络

如果路由器隐藏了 SSID（也就是 WiFi 不会在列表中直接显示），需要在配置里加上 `scan_ssid` 选项：

```text
network={
    ssid="yourHiddenSSID"
    scan_ssid=1
    psk="Your_wifi_password"
}
```

`scan_ssid=1` 的意思是：即便网络没有在广播 SSID，也主动去尝试连接。重载配置并检查连接方式与前面相同。


## 七、配置多个无线网络

在较新的 Raspbian 版本中，你可以在同一个 `wpa_supplicant.conf` 里配置多个可选网络。这样 Raspberry Pi 在不同地方（比如家里、学校）就能自动选择可用网络。

示例：

```text
network={
    ssid="SchoolNetworkSSID"
    psk="passwordSchool"
    id_str="school"
}

network={
    ssid="HomeNetworkSSID"
    psk="passwordHome"
    id_str="home"
}
```

当多个配置都在范围内时，还可以用 `priority` 字段控制优先级。数值越大，优先级越高：

```text
network={
    ssid="HomeOneSSID"
    psk="passwordOne"
    priority=1
    id_str="homeOne"
}

network={
    ssid="HomeTwoSSID"
    psk="passwordTwo"
    priority=2
    id_str="homeTwo"
}
```

在这个例子中，如果两个网络都能连到，Raspberry Pi 会优先连接 `HomeTwoSSID`，因为它的 `priority=2` 更高。


无论你身在家里、办公室还是教室，只要提前把常用 WiFi 配好，Raspberry Pi 插上电就能自动上线，省心又省事。
