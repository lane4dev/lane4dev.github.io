---
layout: post
title: "在 Arch Linux 下挂载加密的 LVM 分区"
date: 2024-10-31 00:00:00 +0800
categories: ArchLinux
tags: ["LVM", "磁盘"]
comments: true
---

鉴于我上一台 Arch Linux 系统的固态硬盘不是简单的 ext4、xfs，
而是那种套了两层壳的分区：**LUKS 加密 + LVM 管理**。
所有当我将它拆下来，通过移动硬盘盒读取时，直接 `mount` 一下就报错，
提示 "unknown filesystem type 'crypto_LUKS"，为了能够读取里面的文件着实花费了我不少功夫。

这篇文章记录我在 Arch Linux 下，手动挂载加密的 LVM 分区的整个过程，顺带解释下每一步在干嘛，方便之后复用或救命。


## 什么是 LVM？

如果你已经熟悉了可以跳过这一小节。

LVM（Logical Volume Manager）是 Linux 下的一套磁盘管理机制。
它的核心思想是把一块或多块物理磁盘整合成一个统一的大空间（称为“卷组”，Volume Group），
然后再从这个空间里灵活划分出多个逻辑卷（Logical Volume），用来挂载、格式化、存储文件。

你可以把 LVM 想象成一个大抽屉柜，而每个逻辑卷就是你从中拉出的抽屉：
外面看起来就是一整个柜子，里面则可以自由分隔、调整大小，甚至临时增加新的抽屉。

当你在 `lsblk` 里看到某个分区的类型是 `LVM2_member`，
那说明这块磁盘只是那个“抽屉柜”的一部分板材，还没组装完、也看不到抽屉。
你不能直接 `mount` 它，而是需要告诉系统：

1. 去识别这块板子属于哪个柜子（卷组）；
2. 把它组装好；
3. 然后你才能打开你要用的抽屉（逻辑卷），也就是挂载真正的文件系统。

---

## 我的磁盘状态

我需要挂载的是一个 U 盘镜像恢复的系统盘，`/dev/sda2` 显示如下：

```bash
$ lsblk -f
NAME            FSTYPE      LABEL       UUID                                 MOUNTPOINT
sda
├─sda1          vfat        BOOT        A1B2-C3D4                            /boot
└─sda2          crypto_LUKS             6a5e...                              # 这就是目标
```

显然这是一个 **LUKS 加密分区**。我接下来要做的是：

1. 解锁加密分区（得到一个 mapper 设备）
2. 识别并激活其中的 LVM 卷组和逻辑卷
3. 挂载逻辑卷的根文件系统

---

## 步骤一：解锁 LUKS 加密分区

先确认系统有 `cryptsetup`：

```bash
sudo pacman -S cryptsetup
```

然后解锁：

```bash
sudo cryptsetup luksOpen /dev/sda2 crypt_data
```

它会提示输入密码。输入正确后，会看到 `/dev/mapper/crypt_data` 出现了。
这是已经解密后的“原始内容”，但还不是我能挂载的根分区，因为它里面套了个 LVM。

---

## 步骤二：识别并激活 LVM 卷组

确认系统已经装了 LVM 工具：

```bash
sudo pacman -S lvm2
```

加载相关模块：

```bash
sudo modprobe dm-mod
```

然后扫描一下系统有哪些 LVM 组件：

```bash
sudo pvscan   # 扫描物理卷
sudo vgscan   # 扫描卷组
sudo lvscan   # 扫描逻辑卷
```

如果看到类似：

```
  Found volume group "vg0" using metadata type lvm2
  ACTIVE            '/dev/vg0/root' [20.00 GiB] inherit
```

那就说明一切正常。我接下来激活它：

```bash
sudo vgchange -ay
```

这一步会让 `/dev/mapper/vg0-root` 这种逻辑卷路径生效。

---

## 步骤三：挂载文件系统

确认文件系统类型（不是必须，但稳一点）：

```bash
sudo blkid /dev/mapper/vg0-root
```

应该能看到 `TYPE="ext4"` 或 `xfs` 之类的字样。

然后挂载：

```bash
sudo mount /dev/mapper/vg0-root /mnt/data
```

搞定！现在就可以访问 `/mnt/data` 里的内容了。

---

## 如果要卸载怎么办？

当处理完数据，可以通过以下命令清理现场：

```bash
sudo umount /mnt/data
sudo vgchange -an          # 关闭卷组
sudo cryptsetup luksClose crypt_data  # 关闭加密设备
```

---

## 小结

* `crypto_LUKS` ≠ 普通文件系统，要先解锁才能访问
* `LVM2_member` ≠ 可挂载设备，要先激活逻辑卷
* `lvm2` 和 `cryptsetup` 是关键组件，没装啥也干不了
* 最终要挂载的，不是 `/dev/sda2`，而是 `/dev/mapper/vgname-lvname`
