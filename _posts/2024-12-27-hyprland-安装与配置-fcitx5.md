---
layout: post
title: "Hyprland 下安装与配置 fcitx5"
date: 2024-12-27 00:00:00 +0800
categories: ArchLinux
tags: ["hyprland", "fcitx5"]
comments: true
---

Hyprland 下安装与配置 fcitx5 的小段备忘（Arch + Wayland）,
这样每次~~换环境 / 重装系统~~（但愿别）的时候，可以按这篇一步一步走，
大概率不用再折腾搜索引擎。

## 场景说明

环境：

- 发行版：Arch / Arch 系（已有 `paru`）
- 桌面：Hyprland（Wayland）
- 目标：在 Hyprland 里正常使用 **fcitx5 + 拼音输入法**，并简单美化一下主题。

---

## 1. 安装 fcitx5 及中文扩展

先把输入法本体装上：

```bash
paru -S fcitx5-im
```

`fcitx5-im` 是一个安装组，会顺带把常用的前端和配置工具装好（GTK/Qt 支持之类）。

如果需要中文输入（肯定需要 😄），再装中文扩展：

```bash
paru -S fcitx5-chinese-addons
```

装完这一步，fcitx5 已经在系统里有了，只是还没被桌面环境“接上”。

---

## 2. 配置环境变量（让应用知道用 fcitx5）

在 `~/.bash_profile` 中追加下面几行（如果没有这个文件就新建一个）：

```bash
export XMODIFIERS=@im=fcitx
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
```

简单记忆：

- `XMODIFIERS`：老传统，告诉输入法框架用 fcitx。
- `GTK_IM_MODULE` / `QT_IM_MODULE`：让 GTK / Qt 应用把输入焦点交给 fcitx5。

备注：

- 如果平时用的是 `zsh` / `fish`，仍然建议在 `~/.bash_profile` 或显示管理器加载的环境文件里写这些变量，保证图形会话都能拿到。
- 改完后需要**重新登录会话**才会生效（单纯 `source` 有时不够）。

---

## 3. 在 Hyprland 启动时自动拉起 fcitx5

打开 Hyprland 的配置文件（默认）：

```bash
~/.config/hypr/hyprland.conf
```

在合适的位置（通常是其他 `exec-once` 附近）加上一行：

```ini
exec-once = fcitx5 -d
```

含义：

- `exec-once`：Hyprland 启动时执行一次。
- `fcitx5 -d`：以后台模式（daemon）启动 fcitx5。

提示：
如果你后面发现 fcitx5 启动了但没有托盘图标，检查一下：

- 是否有系统托盘 / bar 支持（Waybar 等）
- 是否连上了 `fcitx5-qt` / `fcitx5-gtk`（一般 `fcitx5-im` 已覆盖）

---

## 4. 添加拼音输入法 & 快捷键

重登 Hyprland 会话后：

1. 系统托盘里应该能看到 fcitx5 的图标（小键盘 / 方块）。
2. 右键点击该图标 → 打开配置界面（`Configure` / `配置`）。
3. 在「输入法」列表中添加 **Pinyin**（例如 `Pinyin` / `Pinyin (fcitx5)`）。

常用切换快捷键（可在配置里改）：

默认：`Ctrl + Space` 在中英文之间切换。

注意：

如果 `Ctrl + Space` 没反应：

  - 看看 Hyprland 的键位绑定有没有抢这个组合。
  - 在 fcitx5 配置里改一下快捷键（比如 `Super + Space`），避免冲突。

---

## 5. 主题与配色简易美化

fcitx5 支持主题，可以稍微打扮一下候选词界面，眼睛会更舒服。

简单路线：

1. 通过 fcitx5 配置界面打开「外观 / Appearance」。
2. 在主题列表里选一个自己顺眼的（比如常见的暗色主题）。
3. 应用后测试一下：

   - 在任意文本框中切到中文输入，看看候选条样式是否变化。

如果想折腾更花的主题：

- 可以从 AUR / 其他仓库安装主题包（例如常见的 `fcitx5-theme-*`），
- 然后在外观设置里切换到对应主题。
