---
layout: post
title: "VSCode 配置 Kotlin 开发环境"
date: 2020-09-18 18:14:00 +0800
categories: Code
tags: [VSCode, Kotlin]
comments: true
---

由于自己偶尔想写一写 Kotlin，又不愿意使用 Intellij 这么重的编辑器，
好在目前 VSCode 对 Java/Kotlin 支持还算不错，下面提供一个最小可用配置流程。

## 必装插件

VSCode 里需要首先安装这几个扩展：

- **Kotlin Language**
- **Extension Pack for Java**
- **Gradle for Java**

---

## 新建 Gradle 项目（选 Kotlin DSL）

1. `Ctrl + Shift + P` 打开命令面板
2. 输入 `Gradle`
3. 选择 **“Create a gradle java project”**
4. 选目录、项目名
5. 弹出 DSL 选项时，记得选 **Kotlin** 作为 DSL

> 这里先按 Java 项目建，后面再给它加 Kotlin 支持。

---

## 创建 `tasks.json`，方便一键 build/run

1. `Ctrl + Shift + P`
2. 输入 `Tasks`，随便选一个（如 **Configure Default Build Task**）
3. 让 VSCode 帮你生成 `.vscode/tasks.json`，然后把里面核心任务改成这样：

```jsonc
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "build",
      "type": "shell",
      "command": "./gradlew build -x test",
      "problemMatcher": [],
      "group": {
        "kind": "build",
        "isDefault": true
      }
    },
    {
      "label": "run",
      "type": "shell",
      "command": "./gradlew run",
      "problemMatcher": []
    }
  ]
}
```

> 之后就可以用 VSCode 的任务面板来跑 Gradle，不用每次敲命令。

---

## 先确认 Java 版本能跑起来

1. `Ctrl + Shift + P`
2. 输入 `Tasks: Run Task`
3. 选 **`run`** 或 **`build`**

正常的话，终端里会看到类似输出：

```text
> Task :app:run
Hello World!

BUILD SUCCESSFUL in 1s
2 actionable tasks: 1 executed, 1 up-to-date
 *  Terminal will be reused by tasks, press any key to close it.
```

> 这一步只是确认：Gradle 项目本身是 OK 的，问题别出在环境上。

---

## 给项目加上 Kotlin 支持

### 在 `app/build.gradle.kts` 中启用 Kotlin

在原有 `plugins` 里加上 Kotlin JVM 插件，例如：

```kotlin
plugins {
    // Java CLI 应用
    application
    // Kotlin 支持
    kotlin("jvm") version "1.7.10"
}
```

> 版本号按需要替换，这里只是一个参考。

### 建 Kotlin 源码目录

在 `src/main` 下新建 `kotlin` 目录，让 Java / Kotlin 并存：

```text
project
  └─ src
     └─ main
        ├─ java
        └─ kotlin
```

把 Kotlin 源码放到 `src/main/kotlin/...` 里。

构建一次项目，看 `app/build/classes` 里是否多出 Kotlin 的 `.class` 文件，说明编译无问题。

### 修改应用入口到 Kotlin

在 `app/build.gradle.kts` 中设置 Kotlin 入口类，比如：

```kotlin
application {
    // Kotlin 的入口类（Kotlin 文件名 + Kt）
    mainClass.set("gradleexample.KotlinAppKt")
}
```

然后再跑一次：

```text
> Task :app:run
Hello World in Kotlin!

BUILD SUCCESSFUL in 4s
3 actionable tasks: 1 executed, 2 up-to-date
 *  Terminal will be reused by tasks, press any key to close it.
```

> 至此：VSCode + Gradle + Kotlin 基本开发环境就设置好了。

---

## 备忘

- 换机器/目录出问题，优先检查：

  - JDK 版本
  - Gradle wrapper 是否可执行（`chmod +x gradlew`）

- `mainClass` 写不对时，任务会报 “找不到 main class”，优先检查包名 + `Kt` 后缀。

- 插件装完重启 VSCode 一下，很多奇怪的红线就会自己消失。
