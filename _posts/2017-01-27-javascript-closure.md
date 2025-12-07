---
layout: post
title: "JavaScript Closure"
date: 2017-01-27 10:08:00 +0800
categories: Code
tags: [JavaScript]
comments: true
---

> TLDR: Closures are nothing but **functions with preserved data**  
> 简而言之：闭包其实就是：**带着自己环境数据一起工作的函数**。


## 一个非典型的 Closure

所谓闭包（Closure），简单说就是：**一个函数，把它外面作用域里的变量一起“打包带走了”**。

先看一个相对“非典型”的小例子：

```javascript
var outer = 3;

var addTwo = function () {
  var inner = 2;
  return outer + inner;
};

console.dir(addTwo);
```


在这段代码里：

- `outer` 定义在函数外部，是一个全局变量；
- `inner` 是函数内部的局部变量；
- `addTwo` 在执行时，会同时用到 `outer` 和 `inner`。

在 Chrome 里打开控制台，运行上面这段代码，再看看 `console.dir(addTwo)` 的输出效果：

![My helpful screenshot]({{ base.siteurl }}/assets/images/2017-01/closure.png)

你会发现，`outer` 这个变量并没有“消散在空气中”，而是以闭包的一部分，被函数记住了。

这就是闭包最本质的一层含义：**函数可以记住它定义时所在的那片“外部环境”**。


## 一个更有趣的例子

再看一个稍微“实用一点”的版本：

```javascript
var addTo = function (passed) {
  var add = function (inner) {
    return passed + inner;
  };
  return add;
};

console.dir(addTo(2));
```

这里的 `passed` 也是一个外部变量。
当我们调用 `addTo(2)` 时，返回的那个 `add` 函数，会**一直记得**：`passed` 当时是 `2`。

更好玩的是，我们可以用不同的参数，生成一批“定制好的”加法函数：

```javascript
var addThree = addTo(3);
var addFour = addTo(4);

console.log(addThree(1)); // 4
console.log(addFour(1)); // 5
```

这里发生了什么？

- `addTo(3)` 返回了一个函数，这个函数内部“记住了” `passed === 3`；
- `addTo(4)` 返回的另一个函数，则“记住了” `passed === 4`；
- 之后每次调用 `addThree(inner)` 或 `addFour(inner)`，它们都会用到当时各自记住的 `passed` 值。

可以把它想象成：

- `addTo(3)` 做出来的是一个“默认加 3”的函数；
- `addTo(4)` 做出来的是一个“默认加 4”的函数；

它们就像是**预先灌好料的模具**：
模具有点不一样，但用法很统一 —— 只要往里再倒一点新的 `inner`，就能得到不同的结果。


## 小结：为什么要在意闭包？

这个例子虽然简单，但已经展示了闭包的一个常见用途：

- **封装状态**：把某些配置参数（比如这里的 `passed`）藏在闭包里，不暴露到全局；
- **生成“带记忆”的函数**：一次配置，多次复用，代码会更简洁，也更好读。

以后当你在代码里看到：

- 函数里面用到了外层作用域的变量；
- 函数被返回、被当成参数传来传去；

基本就可以猜到：**这里多半有闭包在悄悄帮你干活**。
