---
layout: post
title: "JavaScript callbacks"
date: 2017-02-04 13:15:00 +0800
categories: Code
tags: [JavaScript]
comments: true
---

Callback 函数在 JavaScript 编程中非常常见，这和 JavaScript 本身“事件驱动 + 单线程 + 大量异步”的特性密切相关。

简单说一句：**所谓 callback，本质上就是把一个函数当作参数传给另一个函数，在某个合适的时机再把它“叫出来”执行。**

理解了这件事，其它内容就好办多了。

---

## callback 基础

先看一个非常简单的例子：

```javascript
const x = function () {
  console.log("I am called from inside a function");
};

const y = function (callback) {
  console.log("do something...");
  callback();
};

y(x);
```

在上面的例子中，`x` 被作为参数传给了 `y`，并且在 `y` 的内部被调用。这就是一个典型的 callback 使用场景：**把“要做什么”交给别的函数来决定，自己只负责“在什么时候做”**。

callback 的一个直接好处，就是能让代码变得更灵活。

先看一个不使用 callback 的版本：

```javascript
const calc = function (num1, num2, calcType) {
  if (calcType === "add") {
    return num1 + num2;
  } else if (calcType === "multiply") {
    return num1 * num2;
  }
};

console.log(calc(2, 3, "add")); // 5
console.log(calc(2, 3, "multiply")); // 6
```

这个写法当然可以工作，但扩展起来会有点麻烦，每新增一种计算方式（减法、除法、取模……），都得在 `calc` 里继续加 `if / else if` 分支。

换成 callback 之后，思路就不一样了——`calc` 不关心“你要算什么”，它只负责“帮你调用”：

```javascript
const add = function (a, b) {
  return a + b;
};

const multiply = function (a, b) {
  return a * b;
};

const calc = function (num1, num2, callback) {
  return callback(num1, num2);
};

console.log(calc(2, 3, add)); // 5

const printNum = function (a, b) {
  console.log(`There are two numbers ${a} and ${b}`);
};

calc(2, 3, printNum);
```

现在，`calc` 成了一个通用的“执行器”，
**具体要算什么、要打印什么，全由传进来的 callback 决定**。这样不仅保留了原有的功能，还给了调用者很大的发挥空间——想扩展逻辑，写一个新的函数丢进去就行。

---

## 在 Node.js 中的使用

在 Node.js 里，callback 基本可以算是“标配写法”。原因也很简单：
JavaScript 是非阻塞（Non-blocking）的，当你发一个 HTTP 请求，或者访问数据库时，这些操作通常是异步的。主线程不会傻等，而是继续往下跑。

那么 **等请求“跑完”之后，要做的事情放哪里呢？**

答案就是：放在 callback 里。

下面用一个简单的“模拟访问数据库”的例子来说明：

```javascript
function getUserFromDB(callback) {
  setTimeout(
    () =>
      callback({
        firstName: "Lane",
        lastName: "Tang",
      }),
    1000,
  );
}

function greetUser() {
  getUserFromDB(function (userObject) {
    console.log("Hi " + userObject.firstName);
  });
}

greetUser();
```

这里 `getUserFromDB` 并不会立刻返回用户数据，而是通过 `setTimeout` 模拟了一次耗时 1 秒的异步操作。
当“查库”结束后，它会调用传入的 `callback`，把 `userObject` 作为参数传进去。

从调用者的角度看，逻辑就变成了：

> “先去帮我查一下用户，查完之后再调用我给你的这个函数，在里面打个招呼。”

这种模式在 Node.js 的各种 API 中比比皆是,文件读写、网络请求、数据库操作……都经常会看到 `xxx(args, callback)` 这样的签名。

---

## callback hell 与 Promise 的问题

callback 非常强大，但当嵌套层级越来越多时，代码就容易变成经典的“回调地狱”（[callback hell](http://callbackhell.com/)）：一层套一层、缩进越来越深、错误处理四处飞，读起来让人头皮发麻。

为了解决这种可读性问题，JavaScript 社区逐渐引入了 **Promise**，后来又有了 `async/await`。它们的目标不是替代 `callback`，而是**让异步逻辑以更接近同步代码的方式写出来**，从而摆脱“地狱式缩进”。
