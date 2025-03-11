---
layout: post
title: "ES8（ES2017）新特性指南"
date: 2017-09-03 17:33:00 +0800
categories: Code
tags: [JavaScript]
comments: true
---

# ES8（ES2017）新特性指南

ES8 已经在这个夏天正式发布，也被称作 **ES2017**，为 JavaScript 带来了不少好用的新写法。

---

## Object.values()

现在 `Object.values()` 可以**直接拿到对象里所有属性的值**，不用再自己写循环去遍历。

一个简单的例子：

```javascript
const countries = {
  BR: "Brazil",
  DE: "Germany",
  RO: "Romania",
  US: "United States of America",
};
Object.values(countries);
// ['Brazil', 'Germany', 'Romania', 'United States of America']
```

这里 `Object.values(countries)` 会返回一个数组，里面按顺序装着对象中每个属性的“值”。适合只关心“值”，不太在意 key 的场景。

---

## Object.entries()

如果不仅想拿到值，还想一起拿到 **key + value**，那就用 `Object.entries()`。

它会把对象“拆”成一个由 `[key, value]` 组成的数组：

```javascript
const countries = {
  BR: "Brazil",
  DE: "Germany",
  RO: "Romania",
  US: "United States of America",
};
Object.entries(countries);
// [['BR', 'Brazil'], ['DE', 'Germany'], ['RO', 'Romania'], ['US','United States of America']]
```

返回结果是一个二维数组，每一项都是 `[国家代码, 国家名称]`。这在需要对对象做 `map` / `filter` 这类数组操作时，非常顺手。

---

## 字符串填充：padStart 与 padEnd

有时候我们希望把字符串**补齐到固定长度**，比如对齐输出数字、金额等。ES8 新增了两个方法：

```javascript
"string".padStart(targetLength, padString);
"string".padEnd(targetLength, padString);
```

- `padStart`：在字符串**前面**补东西
- `padEnd`：在字符串**后面**补东西
- `targetLength`：补完之后的总长度
- `padString`：用来补的内容（不写的话默认是空格）

看一组实际的例子：

```javascript
"0.10".padStart(10); // 返回一个长度为 10 的字符串，前面补空格
"hi".padStart(1); // 'hi'  (本来就比 1 长，不补)
"hi".padStart(5); // '   hi'
"hi".padStart(5, "abcd"); // 'abchi'
"hi".padStart(10, "abcd"); // 'abcdabcdhi'
"loading".padEnd(10, "."); // 'loading...'
```

一个常见的用途是 **让输出更整齐**，比如对齐小数点：

```javascript
"0.10".padStart(12); //      '       0.10'
"23.10".padStart(12); //     '      23.10'
"12,330.10".padStart(12); // '  12,330.10'
```

这样打印日志或在控制台看数字时会舒服很多，一眼就能看出列对齐。

---

## Object.getOwnPropertyDescriptors()

`Object.getOwnPropertyDescriptors()` 会返回一个对象中**所有“自有”（非继承）属性的完整描述信息**。

这些描述信息里可能包含：

- `value`
- `writable`
- `get`
- `set`
- `configurable`
- `enumerable`

举个例子：

```javascript
const obj = {
  name: "Pablo",
  get foo() {
    return 42;
  },
};
Object.getOwnPropertyDescriptors(obj);
//
// {
//  "name": {
//     "value": "Pablo",
//     "writable": true,
//     "enumerable": true,
//     "configurable": true
//  },
//  "foo": {
//     "enumerable": true,
//     "configurable": true,
//     "get": function foo()
//     "set": undefined
//  }
// }
```

可以看到，`name` 是一个普通属性，而 `foo` 是通过 getter 定义的访问器属性，它们的描述信息就不一样。

---

### 为什么这个有用？

JavaScript 提供了一个拷贝属性的方法：`Object.assign()`。

它的核心逻辑类似：

```javascript
const value = source[key]; // get
target[key] = value; // set
```

但它只拷贝“值”，不会把 **getter / setter / 不可写属性** 这些“特殊属性特征”一起拷过去。

举个有问题的场景：

```javascript
const objTarget = {};
const objSource = {
  set greet(name) {
    console.log("hey, " + name);
  },
};
Object.assign(objTarget, objSource);
objTarget.greet = "love"; // 实际上变成了给 greet 赋值 'love'
```

原本 `greet` 是一个 setter，用来打印 “hey, xxx”。结果通过 `Object.assign` 拷贝后，`greet` 变成了一个普通属性，行为变了，这就容易踩坑。

---

### 用 Object.getOwnPropertyDescriptors 解决

正确做法是：把属性的“描述”也一起拷贝过去：

```javascript
const objTarget = {};
const objSource = {
  set greet(name) {
    console.log("hey, " + name);
  },
};
Object.defineProperties(objTarget, Object.getOwnPropertyDescriptors(objSource));

objTarget.greet = "love"; // 打印 'hey, love'
```

这样，`greet` 在目标对象上依然是一个 setter，行为完全保留。

---

## 函数参数列表中的尾逗号

这是一条**语法层面的小改动**，现在函数的参数列表和调用时，**最后一个参数后面可以保留逗号**，也算合法写法。

例如：

```javascript
getDescription(name, age,) { ... }
```

这样在多人协作或频繁增删参数时，Git diff 会更干净，也少了一点手改逗号的小烦恼。（实际还是建议尽量不要保留最后的小逗号）

---

## 异步函数：async 和 await

这是 ES8 中最让人开心的特性之一，它让写异步代码时，不再被回调和 then 链拉扯，可以用接近同步的写法来表达异步逻辑。

先看一个例子：

```javascript
function loadExternalContent() {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve("hello");
    }, 3000);
  });
}

async function getContent() {
  const text = await loadExternalContent();
  console.log(text);
}

console.log("it will call function");
getContent();
console.log("it called function");

// 控制台输出顺序：
// 'it will call function' // 同步
// 'it called function'    // 同步
// 'hello'                 // 异步（约 3 秒后）
```

这里的关键点是：

- `async function` 声明了一个异步函数，返回值会被自动包成 `Promise`。
- 在 `async` 函数内部，可以用 `await` 等待一个 Promise 的结果，看起来像同步代码，但不会阻塞整个线程。

读起来的感觉是，“先做这件事，等它结果回来，再继续往下执行”，符合人类的思维顺序。

---

## 共享内存与 Atomics

最后这个特性稍微硬核一点，主要面向更底层的并发场景。

根据规范的说明（[链接在此](https://tc39.github.io/ecmascript_sharedmem/shmem.html)）：

> 共享内存通过新的 `SharedArrayBuffer` 类型暴露出来；
> 新的全局对象 `Atomics` 提供了一组原子操作，可以在共享内存上实现阻塞式的同步原语。

可以理解为：

- **SharedArrayBuffer**：
  允许多个线程共享同一块内存区域，对其中的数据进行读写。

- **Atomics 对象**：
  提供一组原子操作，保证对共享内存的读写不会在中途被打断。
  换句话说：某个原子操作要么“全部完成”，要么“完全没发生”，不会出现“只写了一半”的尴尬情况。

这两个能力组合起来，可以让你在多线程环境中更安全地操作共享数据，构建更底层的同步工具（比如锁、信号量等等）。
日常前端场景不一定会直接用到，但在高性能、并发计算相关的场景里非常关键。
