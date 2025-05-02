---
layout: post
title: "在 Arch Linux 上搭建 Android 开发环境"
date: 2019-02-04 21:14:00 +0800
categories: ArchLinux
tags: [Android, Methodology]
comments: true
---

一直以来，我都习惯在 Arch Linux 上干活：滚动更新、包足够新、想折腾随便折腾。
唯一稍微费点劲的，是当我需要搭一套顺手的 Android 开发环境，
既要 Android Studio，又要 SDK / 模拟器，还要真机调试、文件传输样样都搞定。

这篇文章就是把我当时在 Arch 上搭 Android 开发环境的过程整理一下，既当备忘，也给后来人踩坑少一点。

## 系统与字体准备

### 64 位系统 + multilib

Android 开发工具基本都只考虑 64 位环境，这个在 Arch 上默认没问题。
需要做的，是打开 `multilib` 仓库，因为很多 32 位兼容库从这里来。

编辑 `/etc/pacman.conf`，找到这两行，把注释去掉：

```ini
[multilib]
Include = /etc/pacman.d/mirrorlist
```

保存后同步一下软件源：

```bash
sudo pacman -Syy
```

### 字体 & JDK

Android Studio 是基于 Java 的，字体渲染如果不调一下，看着会比较“渣”。比较流行的做法是：

- 安装 **infinality-bundle**（改进字体渲染）
- 安装打过 infinality 补丁的 `jdk8-openjdk-infinality`

这样 Android Studio 里的英文、代码、中文看起来都会顺眼很多。
这里不展开细节，按对应 wiki 做就行，核心是确保 IDE 用的是这套 JDK。


## 安装 Yaourt 作为 AUR 助手

### 启用 archlinuxfr 仓库并安装 yaourt

编辑 `/etc/pacman.conf`，在文件末尾加上：

```ini
[archlinuxfr]
SigLevel = Never
Server = http://repo.archlinux.fr/$arch
```

然后同步并安装：

```bash
sudo pacman -Syy
sudo pacman -S yaourt
```

安装过程中，终端可能会问你选哪一版 JDK，例如：

```text
1) jdk7-openjdk
2) jdk8-openjdk  (default)
```

一般直接用默认的 JDK 8 就行。

### 开启 xz 多线程压缩

用 yaourt 装 AUR 包时，很多包需要本地编译打包，默认压缩会有点慢。可以给 `xz` 开一下多线程：

编辑 `/etc/makepkg.conf`：

```bash
sudo vim /etc/makepkg.conf
```

找到 `COMPRESSXZ` 那一行，改成：

```bash
COMPRESSXZ=(xz -T 0 -c -z -)
```

`-T 0` 的意思是自动使用所有 CPU 线程，打包速度会舒服很多。

### 减少 yaourt 的交互提问

默认 yaourt 比较**话痨**，每个步骤都要你确认一遍。可以用自己的配置稍微收敛一下。

先复制配置：

```bash
cp /etc/yaourtrc ~/.yaourtrc
```

然后编辑 `~/.yaourtrc`，把下面两行的注释去掉并改好值：

```ini
BUILD_NOCONFIRM=1
EDITFILES=0
```

- `BUILD_NOCONFIRM=1`：打包时不再每步都问你“确定吗”
- `EDITFILES=0`：默认不再打开 PKGBUILD 让你编辑

### 4. 让 yaourt 走代理（Shadowsocks）

有些 AUR 包的源码在 GitHub、Google 之类，因此需要科学上网。

如果你本地跑着 Shadowsocks（比如本地监听 `127.0.0.1:1080`），可以在终端临时加个环境变量，
让 yaourt / makepkg 走 socks5 代理：

```bash
export http_proxy=socks5://127.0.0.1:1080
export https_proxy=socks5://127.0.0.1:1080
```

之后在同一个终端里执行 yaourt，就会通过代理去拉源码了。


## 用 yaourt 安装 Android 相关包

准备工作做完，开始装正主。

Android 开发大致需要几类东西：

- Android SDK 本体
- 各种 platform / build-tools / platform-tools
- Google APIs、support / repository 之类的附加包
- Android Studio IDE 本身
- bash 补全等小工具

一个比较“省心”的一次性安装命令如下：

```bash
yaourt -S \
  aur/android-bash-completion \
  aur/android-google-apis \
  aur/android-google-apis-x86 \
  aur/android-google-repository \
  aur/android-platform \
  aur/android-sdk \
  aur/android-sdk-build-tools \
  aur/android-sdk-platform-tools \
  aur/android-sources \
  aur/android-studio \
  aur/android-support \
  aur/android-support-repository
```

然后就是，**漫长的等待**。

- AUR 包需要从源码编译，CPU 会被拉满一阵子；
- 中途如果网速不太给力，下载 Android 相关的大文件也会比较慢。

这一步建议倒杯可乐，顺便刷会儿 B 站。


## 配置 /opt/android-sdk 的权限

默认情况下，Android SDK 会装在 `/opt/android-sdk/` 目录。这个目录属于 root，普通用户没写权限，
而 Android Studio 在更新 SDK / 下载新平台时，需要往里面写东西。

比较稳妥的做法是：把这个目录的属组改成 `wheel`，给组可写权限，
这样所有 `wheel` 组用户（一般是桌面登录用户）都能维护 SDK：

```bash
sudo chown -R :wheel /opt/android-sdk/
sudo chmod -R g+w /opt/android-sdk/
```

如果你图省事，也可以直接粗暴一把：

```bash
sudo chmod -R 777 /opt/android-sdk
```

但这种做法不太安全，**不推荐**，知道有这么一招就好。


## 配置 JDK 与 JAVA_HOME

为了让 Android Studio 和命令行工具都能找到 JDK，可以在 `/etc/profile` 里加一行：

```bash
sudo vim /etc/profile
```

在文件末尾添加：

```bash
export JAVA_HOME=/usr/lib/jvm/java-7-openjdk
```

如果你选的是 JDK 8 或带 infinality 补丁的版本，对应地改成：

```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk
# 或者具体的 infinality 路径
```

保存后重新登录，或者手动 `source /etc/profile` 一下。


## 首次启动 Android Studio

一切准备完成，接下来：点开 Android Studio。

1. 在菜单里找到 **Android Studio**（或者直接终端里运行 `android-studio`）。
2. 第一次启动会弹出向导，建议选 **Custom**，因为要手动把 SDK 目录改成刚才装好的 `/opt/android-sdk/`。
3. 设置好 SDK 路径后，你可以再切回 **Standard** 模式，让它自动帮你选一套默认配置。
4. 一路 `Next` → `Finish`。

接着可以随便建立一个 HelloWorld 项目，这时候 Android Studio 会下载 Gradle、依赖等内容，又是一段安静的等待时间。


## 模拟器的硬件加速（KVM）

没硬件加速的 Android 模拟器，体验基本等于“截图浏览器”。在 Arch 下启用 KVM 的步骤大致是：

1. 安装 KVM 相关包：

   ```bash
   sudo pacman -S qemu libvirt
   ```

2. 在 BIOS 中确认已经打开虚拟化（Intel VT-x / AMD-V）。

3. 把自己的用户加入相应的虚拟化用户组（视当时文档而定，一般是 `kvm`、`libvirt` 等）。

在 Android Studio 里创建 AVD（虚拟设备）时，选择带硬件加速的镜像，就能明显感到流畅度不同。

（更细的 libvirt / virt-manager 配置，这里就不展开了。）


## 真机调试与 MTP 文件传输

### 真机调试：adb 设备识别

有时候 Android Studio 识别不到手机，这时需要装几个 udev / MTP 相关的包：

```bash
sudo pacman -S android-udev
yaourt -S simple-mtpfs
```

安装完 `android-udev` 后拔插手机、重新 `adb kill-server && adb start-server` 一下，通常就能在 `adb devices` 里看到设备了。

### 不同桌面环境下的 MTP 支持

不同桌面环境里，挂载 Android 手机（MTP）略有差别：

- **GNOME：**

  ```bash
  sudo pacman -S gvfs gvfs-mtp
  ```

- **KDE：**

  ```bash
  sudo pacman -S kio-mtp
  ```

- **Xmonad（纯窗口管理器）：**

  如果你不用完整桌面，只跑 Xmonad，可以用一个简单工具：

  ```bash
  sudo pacman -S android-file-transfer
  ```

装完之后，重启一下系统或者重启会话，一般文件管理器就能直接看到 Android 设备了。


## 在 Xmonad 下解决 Android Studio 显示空白的问题

如果你用的是 Xmonad 之类的平铺窗口管理器，Android Studio 可能会出现一个经典问题：
弹出对话框时内容空白、看不见任何东西，只剩一行小提示，比如：

> Did you know?
> …（一堆快捷键介绍）…

这通常是 Java Swing/IntelliJ 系列程序和“非传统 WM”之间的小冲突。

有两步可以缓解：

1. 安装 `wmname` 并伪装窗口管理器

```bash
sudo pacman -S wmname
wmname LG3D
```

`LG3D` 是一个常见的“假名字”，很多 Java 程序会把它当成兼容的 WM，行为会正常很多。

你可以把 `wmname LG3D` 写进 Xmonad 启动脚本或 `.xinitrc` 里，保证每次启动 X 时自动生效。

2. 在 Xmonad 配置里增加焦点处理

修改你的 `~/.xmonad/xmonad.hs`，加上：

```haskell
import qualified XMonad.Hooks.ICCCMFocus as ICCCMFocus
```

然后在配置里加上：

```haskell
logHook = ICCCMFocus.takeTopFocus
```

重新编译并重启 Xmonad 后，Android Studio 的各种对话框基本就能正常显示和获取焦点了。
