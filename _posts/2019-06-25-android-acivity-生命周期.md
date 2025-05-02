---
layout: post
title: "Android Activity 生命周期详解"
date: 2019-06-25 10:56:00 +0800
categories: Code
tags: [Android]
comments: true
---

Android 开发里，Activity 生命周期几乎是第一个会被画在白板上的图：从 `onCreate()` 到 `onDestroy()` 的几条箭头，面试和文档都爱用它来开场。问题是，很多人把这张图背得滚瓜烂熟，一到真实场景——按 Home、按 Back、旋转屏幕、两个 Activity 来回跳，Logcat 一刷全是回调，脑子却一片混乱。

这篇文章想做的事很简单：
从「用户在干什么」的场景出发，把那些回调和状态变化串起来。

顺便聊聊：每个回调里适合做什么、不适合做什么，哪些状态要自己管，避免“旋转一次屏幕，状态全没了”。

## 全局视角

先把整个版图过一遍，再用场景细拆。

![]({{ base.siteurl }}assets/images/2018-12/25-001-android-activity-lifecycle.png)

官方给 Activity 定了一套核心回调方法，大致可以想象成几个阶段：

- **创建期**：`onCreate()`
- **变得可见**：`onStart()`
- **在前台、可交互**：`onResume()`
- **准备失去焦点**：`onPause()`
- **完全不可见**：`onStop()`
- **真正结束前的清理**：`onDestroy()`
- **从停止状态重新拉起来**：`onRestart()`，它后面必然跟着 `onStart()`。

除此之外，还有两个专门为了**状态保存/恢复**准备的回调：

- `onSaveInstanceState(Bundle outState)`
- `onRestoreInstanceState(Bundle savedInstanceState)`

它们不一定每次都会被调用，但一旦遇到配置变更、进程在后台被回收，就会是你保住 UI 状态的关键。

## 用典型场景串一遍生命周期

光看方法名还是有点抽象，我们按几个常见操作来过一遍回调顺序。

### 首次启动 & 按 Back 正常退出

**首次从桌面打开应用** 时，大致流程是：

```text
onCreate() → onStart() → onResume()
```

这时 Activity 已经在前台、可以接收触摸和键盘输入。

**用户按 Back 退出当前 Activity** 时，则大致是：

```text
onPause() → onStop() → onDestroy()
```

这里有两个关键点：

按 Back 会把这个 Activity 从返回栈里移除，生命周期会走到 `onDestroy()`；

这种**用户明确退出应用**的场景下，一般不会再去保存什么**待恢复**的临时状态。

### 按 Home 回桌面，再回来

不同点：**没退出应用，只是暂时看不见**。

前台使用 Activity 时，按下 Home：

```text
onPause() → onStop()
```

Activity 进入 Stopped 状态，实例仍在内存里。

稍后从最近任务或桌面图标回到应用：

```text
onRestart() → onStart() → onResume()
```

此时：

Activity 实例通常会被复用，成员变量、部分 View 状态还在。

系统可能在 `onStop()` 之后调用过 `onSaveInstanceState()`，以防在后台挂着时进程被杀，下次可以恢复状态。

如果在后台时进程真的被系统回收了，再次点图标，就会重新走一遍 `onCreate()`，相当于重新启动。

### 旋转屏幕 / 配置变更

很多令人抓狂的 bug 都和这里有关：表单写了一半，手机横过来，界面清空；列表滑到一半，旋转一下，又回到顶部。

默认情况下，当发生“配置变更”（最典型的是横竖屏切换）时，系统会：

**销毁当前 Activity，重新创建一个新实例。**

旧实例的大致流程：

```text
onPause() → onSaveInstanceState() → onStop() → onDestroy()
```

新实例随后登场：

```text
onCreate(savedInstanceState) → onStart() → onRestoreInstanceState() → onResume()
```

几个要点：

- 系统会自动帮你保存一部分 View 状态（比如 `EditText` 的内容、`ListView` 的滚动位置），剩下的需要你在 `onSaveInstanceState()` 里手动存。
- `onCreate()` 收到的 `savedInstanceState` 和 `onRestoreInstanceState()` 里的其实是同一个 Bundle。
- 如果你完全不处理，旋转之后界面会回到“初始状态”，用户之前的操作就都没了。

### 被对话框、多窗口遮挡

还有一类情况：你的 Activity 看起来还**在当前界面**，但上面盖了其他东西，例如：

- 系统弹出的运行时权限对话框
- 某些第三方登录、支付界面
- 进入分屏或浮动窗口模式

这时：

Activity 可能会收到 `onPause()`，有时不会立即 `onStop()`，因为界面仍部分可见；

同一应用内的普通对话框（`AlertDialog`、`DialogFragment`）通常不会触发 Activity 进入 Paused 状态，生命周期保持不变。

理解这一点，对处理“被遮挡但还活着”的场景很重要，比如暂停动画、控制相机等。


## 3. 状态保存与恢复：到底该交给谁管？

想真正用顺生命周期，先把一个问题想清楚：
哪些状态要跨“重建”保留？谁来帮你保？

### 先分两类状态

一个实用的划分方式：

1. **长期业务数据**

   - 用户信息、配置项、缓存列表、草稿内容……
   - 生命周期可以跨多次 app 启动。
   - 应该放在数据库、SharedPreferences、本地文件、网络等“持久层”，不要绑死在 Activity 上。

2. **这次会话里的临时 UI 状态**

   - 当前选中的 Tab、是否展开某个 Panel、滚动位置、一次性提示是否已经弹过……
   - 只需要在这轮会话里活着，即便发生配置变更也希望被恢复。
   - 适合通过 `onSaveInstanceState()` → `onCreate()` / `onRestoreInstanceState()` 来保存和恢复。

很多“旋转后状态消失”的 bug，其实就是第二类状态没人管。

### 3.2 onSaveInstanceState() 何时会被调用？

系统在“有可能要重建 Activity”时，会调用 `onSaveInstanceState()`：

典型场景包括：

- 配置变更：横竖屏切换、语言切换、主题模式变化等；
- Activity 退到后台后，系统需要回收内存，可能干掉所在进程。

调用顺序通常大致是：

```text
onPause() → onSaveInstanceState() → onStop()
```

但是要注意：

**按 Back 退出、主动 `finish()` 通常不会触发这个回调**，因为系统默认认为你已经“不要再恢复这页了”。

Bundle 是为了“同一轮会话里的重建”准备的，不是为了持久存储。真正要跨进程、跨多次启动的东西，还是要靠持久层。

### onCreate() 和 onRestoreInstanceState()

当 Activity 被重建时，系统会把之前 `onSaveInstanceState()` 里存的 Bundle 传回来：

- `onCreate(Bundle savedInstanceState)`：
  Activity 创建时最早拿到这份 Bundle。通常用来把 UI 恢复到一个“至少不违和”的初始状态。

- `onRestoreInstanceState(Bundle savedInstanceState)`：
  在 `onStart()` 之后调用，这时 View 层级已经完全创建完了，适合做依赖控件的恢复，比如滚动位置、焦点等。

一个常见模式是：

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)

    // 初始化 View

    // 如果有历史状态，先做一层基础恢复
    if (savedInstanceState != null) {
        // 恢复不依赖复杂 UI 的状态
    }
}

override fun onRestoreInstanceState(savedInstanceState: Bundle) {
    super.onRestoreInstanceState(savedInstanceState)
    // 这里可以安全地操作所有 View（比如 RecyclerView 的滚动位置）
}
```

这样既能保证“冷启动时 UI 正常”，又能在“重建时尽量贴近重建前的状态”。

---

## 每个生命周期方法里适合做什么？

下面换个角度，不再强调**何时被调用**，而是聊聊**这个回调里你可以考虑做什么，不该做什么**。

### onCreate()

适合做：

1. 一次性的初始化：`setContentView()`、`findViewById()` / ViewBinding、初始化 ViewModel / Presenter；
2. 拿 `Intent` 里的参数、解析基础配置；
3. 建立一些“整个生命周期都在”的依赖关系（例如注入某些组件）。

不太适合做：

依赖“每次可见都要刷新”的操作，比如运行时权限检查、根据用户登录状态调整 UI，
这些更适合放在 `onStart()`，否则从设置里改权限再回来时可能不会重新跑逻辑。

### onStart()

适合做：

每次 Activity 即将可见时的检查和刷新：

  - 再次校验权限；
  - 根据最新配置（主题、语言）更新 UI；
  - 注册只在可见时才需要的监听（例如某些广播）。

可以理解成：**“你马上要被用户看到，先把 UI 整理一下”**。

### onResume()

适合做：

真正依赖“在前台”的工作：

  - 开始动画、渲染密集的效果；
  - 开启摄像头预览、传感器监听；
  - 开始播放音视频。

这些东西通常需要在 `onPause()` 里对称地停掉，不然就容易浪费资源。

### 4.4 onPause()

`onPause()` 通常是生命周期里**最短、最敏感**的一个回调。系统希望它尽快返回，好让下一个前台界面顺利切入。

适合做：

1. 暂停正在运行的动画；
2. 停止相机预览、麦克风采集；
3. 在需要时简单地保存一下轻量级状态。

不适合做：

重 IO：大规模数据库操作、大文件写入、复杂网络请求。

这些更适合放到后台任务里，或者提前在业务层做。

### 4.5 onStop()

当 Activity 完全不可见时，系统会调用 `onStop()`。

适合做：

释放“用户看不见就应该被关掉”的资源：

  - 注销广播接收器；
  - 取消各种观察者/订阅；
  - 断开不再需要的网络连接。

很多时候，如果你想**尽量早一点清资源**，可以选择在 `onStop()` 而不是 `onDestroy()` 里做处理。

### onDestroy()

`onDestroy()` 是 Activity 生命周期里“真正的最后一站”，但它**不是总能保证走得到**：系统直接杀进程时不会再特地回调它。

可以在这里做的：

释放和 Activity 绑定得很死、只有在 Activity 真正销毁时才需要清理的东西，例如某些 native 资源。

但更安全的习惯是：

不要把关键的持久化、业务清理都压在 `onDestroy()`，它更像是“加一层保险”，而不是“唯一依赖点”。

---

## 实用建议

聊完流程，再看几个典型的坑。

### 所有逻辑都挤在 onCreate() / onDestroy()

刚开始项目不大时，把初始化全写在 `onCreate()`、清理全写在 `onDestroy()` 似乎没什么问题。时间久了，很容易出这种情况：

- 权限只在 `onCreate()` 检查：用户从系统设置改了权限再回来，逻辑没跑到；
- 关键清理只放在 `onDestroy()`：进程在后台被系统杀掉时，这段代码根本没有执行的机会。

比较健康的做法是：

- 初始化拆到 `onCreate()`、`onStart()`、`onResume()` 几层；
- 清理拆到 `onPause()`、`onStop()`，`onDestroy()` 只做必要的收尾工作。

### 忽略配置变更，旋转就丢状态

旋转一次屏幕，整个界面回到初始状态，其实正是默认配置变更行为的体现：旧 Activity 被销毁，新 Activity 重建，但你没有任何状态恢复逻辑。

实用建议：

- 明确哪些状态要恢复，哪些不恢复；
- 在 `onSaveInstanceState()` 里只保存必要的 UI 状态（比如当前选项、已输入但未提交的内容）；
- 在 `onCreate()` / `onRestoreInstanceState()` 里恢复 UI 时，注意和“冷启动”的逻辑兼容。

如果业务上必须自己接管配置变更，可以通过 manifest 或 `onConfigurationChanged()` 来处理，但那是另一整块话题了。

### 分不清 “Back 退出” 和 “后台被杀”

按 Back 退出时的典型调用顺序：

```text
onPause() → onStop() → onDestroy()
```

而 Activity 在后台被系统杀掉时，常见的流程是：

```text
onPause() → onStop() → （进程被干掉，不再调用 onDestroy）
```

这就是说：

如果你想确保某些清理一定发生，就不要只写在 `onDestroy()`；

要区别对待“用户主动退出”和“系统被动回收”，重要数据尽量在更早的时机存好。

### 5.4 生命周期逻辑散落一地

随着功能增多，很容易出现这样的 Activity：

```kotlin
override fun onResume() {
    super.onResume()
    analytics.trackScreen()
    player.resume()
    locationManager.start()
    // ...
}

override fun onPause() {
    super.onPause()
    player.pause()
    locationManager.stop()
    // ...
}
```

生命周期回调里塞满各种业务调用，后期维护会很吃力。

一个思路是：让“需要感知生命周期的组件”自己观察生命周期（例如通过某种生命周期感知机制），而不是把所有东西都粘在 Activity 回调里。这样 Activity 更多只负责 UI 和导航，其他组件自己处理 `ON_START` / `ON_STOP` 之类的事件，代码会清爽很多。

## 参考资料

- [Android Activity Lifecycle](https://blog.mindorks.com/android-activity-lifecycle/)
- [The Android Lifecycle cheat sheet — part I: Single Activities](https://medium.com/androiddevelopers/the-android-lifecycle-cheat-sheet-part-i-single-activities-e49fd3d202ab)
- [Activity Lifecycle](https://guides.codepath.com/android/Activity-Lifecycle)
- [The activity lifecycle](https://developer.android.com/guide/components/activities/activity-lifecycle)
