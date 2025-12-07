---
layout: post
title:  "我的 ArchLinux 配置"
date:   2016-03-17 12:45:00 +0800
categories: ArchLinux
banner: 
  image: /assets/images/2016-03/archlinux.png
comments: true
---

对我来说，Arch 做得最让人上瘾的一点，就是它的包管理系统。  
用 `pacman` 和 `yaourt`，基本能在仓库里找到我想装的所有软件，不用到处找源码、手动编译，也不用自己记一堆安装路径，卸载更新都很省心。再加上滚动更新（rolling release），系统和应用一直保持在相对新的状态，这点对喜欢折腾的人非常友好。

当然，Arch 也不是“人人可上手”的发行版。它对用户的前置知识要求不低，尤其是安装阶段，全程都是命令行，没有接触过 Linux 的同学可能会有点懵。不过好在 [ArchWiki](https://wiki.archlinux.org/) 写得非常详细，只要愿意一步步看文档，不光能把系统装起来，对操作系统的一些基本概念也会更有感觉。


## yaourt 安装与使用

在我完成基础系统安装后，通常会先把 `yaourt` 装上。

`yaourt`（Yet Another User Repository Tool）是社区为 Arch 做的一套工具，用来扩展 `pacman` 对 AUR（Arch User Repository）的支持。它可以自动编译和安装来自 AUR 和官方仓库的软件。语法基本沿用了 `pacman` 的风格，还带了一个简单的高亮交互界面，用起来比纯命令行更顺手一点。

### 安装 yaourt

安装 `yaourt` 的步骤很简单，先编辑 `pacman` 的配置文件：

```bash
$sudo vim /etc/pacman.conf
````

在文件末尾添加一个新的源：

```text
[archlinuxfr]
SigLevel = Never
Server = http://repo.archlinux.fr/$arch
```

保存退出后，就可以直接用 `pacman` 安装 `yaourt` 了：

```bash
$sudo pacman -Sy yaourt
```

到这里，AUR 的世界就向你打开了。

### 使用 yaourt

`yaourt` 的基本语法结构是：

```bash
$yaourt <操作符> [操作] [应用名]
```

更新整个 Arch 系统，只需要一条命令：

```bash
$yaourt -Syu
```

安装某一个特定的包：

```bash
$yaourt -S <应用名>
```

升级已存在的包：

```bash
$yaourt -U <应用名>
```

从系统中删除一个包：

```bash
$yaourt -R <package-name>
```

安装本地的包：

```bash
$yaourt -P <应用所在的文件夹>
```

查看当前系统中安装应用的统计信息：

```bash
$yaourt --stats
```

日常使用下来，基本就是围绕这几条命令在转，熟悉之后会感觉包管理这块非常顺手。

---

## 关于 GNOME 的介绍

Gnome 是 Linux 下比较主流的一种桌面环境。和 Windows 不同，Linux 的桌面环境是和内核分开的：桌面对系统来说只是一个“应用”，不装桌面，系统照样可以跑起来。

Arch 本身并不自带桌面环境，装系统之后需要根据自己的喜好选择。目前常见的桌面环境有 Gnome、KDE、Xfce、Budgie 等等。想要了解更多，可以参考 ArchWiki 上的页面：[Desktop environment](https://wiki.archlinux.org/index.php/Desktop_environment#List_of_desktop_environments)。

我个人比较偏爱 Gnome。它算是比较老牌的桌面环境，整体相对稳定，背后也有比较成熟的开发和维护团队。

![Gnome](https://www.gnome.org/wp-content/uploads/2015/10/activities_overview.png)

默认的 Gnome 看起来可能有点“素”，但这也正是 Linux 的乐趣所在——强大的可定制性。
通过主题、插件和各种小工具，不仅可以把界面打扮得更顺眼，还能根据自己的习惯优化操作流程，真正用起来顺手、顺眼。

---

## 我的 Gnome 配置

### 1. 准备工作

第一件事是安装 `gnome-tweak-tool`，有了这件“扳手”，Gnome 的各种设置都会直观很多。

`gnome-tweak-tool` 默认包含在 `gnome-extra` 里，如果之前没有装过 `gnome-extra`（其实挺推荐装一套的，里面带了不少实用小工具），可以单独安装：

```bash
$pacman -S gnome-tweak-tool
```

### 2. 插件管理

打开 `Tweak Tool`，在 `Extensions` 标签页里就可以看到插件列表了。
部分插件是系统自带的，更多的可以从 [官网](https://extensions.gnome.org/) 安装。除了 Chrome 以外，大多数浏览器都可以直接在网页上安装和管理插件；如果你用的是 Chrome，可以参考这篇 Wiki：[GnomeShellIntegrationForChrome](https://wiki.gnome.org/Projects/GnomeShellIntegrationForChrome)。

下面是我自己常开的几款插件，简单说说它们的大致用途：

* [User Themes](https://extensions.gnome.org/extension/19/user-themes/)：
  必开插件之一，否则不能自定义 Shell 主题。

* [Activities Configurator](https://extensions.gnome.org/extension/358/activities-configurator/)：
  用来调整左上角“活动概览（Activities）”按钮的显示方式，比如改成图标、改显示文字等。
  ![Activities Configurator](https://extensions.gnome.org/extension-data/screenshots/screenshot_358_0tEVXqg.png)

* [AlternateTab](https://extensions.gnome.org/extension/15/alternatetab/)：
  通过 `Alt + Tab` 更灵活地在不同应用之间切换。
  ![AlternateTab](https://extensions.gnome.org/extension-data/screenshots/screenshot_15.png)

* [Applications Menu](https://extensions.gnome.org/extension/6/applications-menu/)：
  在左上角增加一个按类别归类的应用菜单，更适合喜欢菜单式启动的同学。
  ![Applications Menu](https://extensions.gnome.org/extension-data/screenshots/screenshot_6.png)

* [Caffeine](https://extensions.gnome.org/extension/517/caffeine/)：
  临时“关掉”屏幕自动休眠，追番、看文档或跑任务时很实用。
  ![Caffeine](https://extensions.gnome.org/extension-data/screenshots/screenshot_517.png)

* [Dash to Dock](https://extensions.gnome.org/extension/307/dash-to-dock/)：
  非常好用，可以把默认只在“概览”里出现的 Dock 固定到屏幕侧边，兼顾快速启动和多桌面切换。
  ![Dash to Dock](https://extensions.gnome.org/extension-data/screenshots/screenshot_307_VW5dorQ.png)

* [Dynamic Panel Transparency](https://extensions.gnome.org/extension/1011/dynamic-panel-transparency/)：
  控制顶部面板的透明度：普通窗口时保持自定义的半透明效果，全屏时自动变成不透明，视觉效果会更统一。
  ![Dynamic Panel Transparency](https://extensions.gnome.org/extension-data/screenshots/screenshot_1011_pWGnOl9.png)

* [Impatience](https://extensions.gnome.org/extension/277/impatience/)：
  加快“概览”动画的速度，让切换体验更干脆利落。

* [Native Window Placement](https://extensions.gnome.org/extension/18/native-window-placement/)：
  在“概览”中更合理地摆放各个应用窗口。
  ![Native Window Placement](https://extensions.gnome.org/extension-data/screenshots/screenshot_18.png)

* [Removable Drive Menu](https://extensions.gnome.org/extension/7/removable-drive-menu/)：
  插入 U 盘、手机等外接设备时，在顶部面板显示状态，并提供快捷卸载入口。
  ![Removable Drive Menu](https://extensions.gnome.org/extension-data/screenshots/screenshot_7.png)

上面这些插件加起来，大概就勾勒出了一个“好看又好用”的 Gnome 桌面基础形态。

### 3. 自定义主题

`Numix-themes` 是一套基于 Material Design 风格的主题，包含 [GTK+](https://en.wikipedia.org/wiki/GTK%2B) 主题和图标主题，看起来比较干净现代。更多信息可以去 [Numix 官方网站](https://numixproject.org/) 逛一逛。

![Numix](http://pre07.deviantart.net/4d48/th/pre/f/2014/223/d/3/numix_circle_linux_desktop_icon_theme_by_me4oslav-d6uxcka.png)

安装也很简单：

```bash
$pacman -S numix-themes
```

安装完成后，打开 `Tweak Tool`，在 `Appearance` 标签页中就可以选择 GTK 主题和图标主题；`Extensions` 标签页则可以开关和配置前面提到的插件（里面还有不少零碎但实用的设置）。

另外再推荐几套我喜欢的主题和图标：

* [Adapta-theme](https://github.com/adapta-project/adapta-gtk-theme)

  ![Materials](https://github.com/adapta-project/adapta-github-resources/raw/master/images/Materials.png)

* [Arc-theme](https://github.com/horst3180/arc-theme)

  ![githubusercontent](https://camo.githubusercontent.com/4c0001cbfe222446c4b3af91027b716daec7d3d7/687474703a2f2f692e696d6775722e636f6d2f5068354f624f612e706e67)

* [Paper-icon-theme](https://github.com/snwh/paper-icon-theme)

  ![screenshot](https://snwh.org/images/paper/screenshot.png)

* [Capitaine-cursors](https://github.com/keeferrourke/capitaine-cursors)

  ![capitaine\_cursors\_by\_krourke](http://pre06.deviantart.net/4408/th/pre/f/2016/208/4/7/capitaine_cursors_by_krourke-dabmjtm.png)

* [La-Capitaine-icon-theme](https://github.com/keeferrourke/la-capitaine-icon-theme)

  ![preview](https://github.com/keeferrourke/la-capitaine-icon-theme/raw/master/preview.svg.png)

这些主题和图标混搭起来，基本可以把 Gnome 换上你自己那一身“皮”。

---

## 关于 Shell 的选择

大多数 Linux 发行版（包括 macOS）默认使用的 Shell 是 Bash。
但实际用下来，我更喜欢 Zsh —— 尤其是自动补全、插件生态这两块，体验会明显好很多。配合 Oh-My-Zsh 这个配置框架，只需要几行命令，就能把终端变得既好看又好用。

具体怎么玩主题和插件，可以到 [Oh-My-Zsh 官方网站](http://ohmyz.sh/) 看详细说明，这里只简单记录一下安装步骤。

Arch 默认不带 Zsh，需要先手动安装：

```bash
$pacman -S zsh
```

然后用一行命令安装 Oh-My-Zsh：

```bash
$sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

最后，把默认 Shell 从 Bash 切换到 Zsh：

```bash
$chsh -s /usr/local/bin/zsh
```

搞定之后，打开一个新的终端，你就能看到一个截然不同的命令行世界。

---

按照上面的步骤折腾一圈下来，一个比较顺手、看着顺眼的 Gnome 桌面基本就成型了。
接下来就可以根据自己的习惯慢慢微调：换主题、调动画、加快捷键……Arch + Gnome 的搭配，折腾空间非常大，也非常适合喜欢控制细节的人。
