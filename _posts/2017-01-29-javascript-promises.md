---
layout: post
title: "JavaScript Promises"
date: 2017-01-29 12:59:00 +0800
categories: Code
tags: [JavaScript]
comments: true
---

> TLDR: Promise 可以先拿到“还没到的结果”，再约好：**结果到了之后要做什么**。


## 一个非典型的 Promise

姑且不论晦涩的概念，把 Promise 想象成这样一件事：

> 你点了外卖，商家给了你一个订单号。  
> 食物还没到，但你已经可以：
>
> - 关注它的状态（正在制作 / 配送中 / 已送达）
> - 约好送到时要做什么（比如开吃😋）

在 JavaScript 里，Promise 就是用来描述这种 **未来某个时间点才会得到的值**。

先看一个“非典型”、但足够简单的例子：

```javascript
var p = new Promise(function (resolve, reject) {
  // 这里直接给出结果，没有任何异步操作
  resolve(42);
});

p.then(function (value) {
  console.log("结果是：", value); // 结果是： 42
});
```

这里发生了什么？

- `new Promise(...)` 里传入的函数会**立刻执行一次**；
- 我们在里面直接调用 `resolve(42)`，表示：**这个 Promise 成功完成了，结果是 42**；
- 后面的 `.then(...)` 则是在说：

  > “当结果准备好时，请帮我执行这段回调，并把结果传进来。”

从状态的角度看，一个 Promise 大概会经历三种状态：

1. `pending`：还在路上（外卖制作/配送中）
2. `fulfilled`：成功了，有结果了（顺利吃上饭）
3. `rejected`：失败了，有原因（骑手迷路 / 系统炸了）

在上面的例子里，因为我们立刻调用了 `resolve(42)`，所以 Promise 会立即变成 `fulfilled`，然后执行 `.then` 里的回调。

这个例子看起来有点“假异步”——没有真正的网络请求、定时器之类，但它把 Promise 的核心节奏已经演示出来了：**先定义，然后约定成功/失败之后要做什么**。


## 一个更有趣的例子

真正用 Promise 的时候，通常都离不开“等一等”。比如我们用 `setTimeout` 模拟一个异步操作：

```javascript
function wait(ms) {
  return new Promise(function (resolve, reject) {
    setTimeout(function () {
      resolve(ms);
    }, ms);
  });
}

wait(1000).then(function (time) {
  console.log("等了 " + time + " ms");
});
```

这段代码在做什么？

- `wait(ms)` 返回一个 Promise 对象；
- Promise 里面用 `setTimeout` 模拟了一个异步任务；
- `ms` 毫秒后，我们调用 `resolve(ms)`，表示：

  > “好啦，我等完了，结果就是刚才传进来的那个毫秒数。”

于是，当我们写：

```javascript
wait(1000).then(function (time) {
  console.log("等了 " + time + " ms");
});
```

相当于告诉 JavaScript 引擎：

> “帮我等 1 秒，等完之后就打印一行日志。”

因为 Promise 支持链式调用，我们可以把多个异步操作串起来写，让逻辑看起来顺顺溜溜的：

```javascript
wait(500)
  .then(function (t) {
    console.log("Step 1：等了 " + t + " ms");
    return wait(1000); // 再等 1 秒
  })
  .then(function (t) {
    console.log("Step 2：又等了 " + t + " ms");
    return "全部完成";
  })
  .then(function (msg) {
    console.log(msg);
  })
  .catch(function (err) {
    console.error("出错了：", err);
  });
```

这里有几个关键点：

- 每个 `.then` 都可以**返回一个新的值**或者**返回另一个 Promise**；
  - 返回普通值：下一个 `.then` 直接拿到这个值；
  - 返回 Promise：下一个 `.then` 会等这个 Promise 完成。
- `.catch` 可以统一处理前面链条里的异常：
  - 无论是 `reject(...)`，还是回调里抛出的错误，都会一路冒泡到 `.catch`。

你可以把这一整串理解成：

> “先做 A，做完再做 B，做完再做 C，如果中途翻车，就走统一的错误处理通道。”

相对于一层套一层的回调（callback hell），Promise 至少让这件事**长得像一条线，而不是一棵树**。


## 小结：为什么要在意 Promise？

现代 JavaScript 里，只要涉及异步，就绕不开 Promise。
简单总结一下，它的价值主要有这几条：

1. **把“未来的值”包装成一个对象**
   - 不再是“给我一个回调函数，我到时候会调用你”，
   - 而是“给你一个 Promise，你可以在方便的时候再去关心它”。

2. **让异步流程变得可读、可组合**
   - 用 `.then` 链接步骤，用 `.catch` 统一处理错误；
   - 比一层套一层的回调要直观得多。

3. **是 async/await 的基础**
   - `async/await` 本质上就是把 Promise 写得更像同步代码；
   - 但底层仍然是 Promise 在跑。

4. **减少“意外行为”**
   - Promise 一旦从 `pending` 变成 `fulfilled` 或 `rejected`，状态就**不会再变**；
   - 比随意乱调回调函数可靠得多，更容易分析和调试。

下次当你看到：

- 某个 API 返回了一个 `Promise`；
- 代码里出现一串 `.then(...).then(...).catch(...)`；
- 或者你在用 `async/await` 写异步逻辑；

都可以想起本文开头那句话：

> Promise 就是那个“外卖订单号”——
> 结果还没到手，但你已经提前约好了，**到了之后要做什么**。
