---
title: forEach与await引发的血案
date: 2023-06-11 16:46:21
tags: js原理
---

<small>原文发布于 [CSDN](https://blog.csdn.net/RaeZhang/article/details/117528885)，本文系近期迁移，想看更多欢迎访问 [我的 CSDN 主页](https://blog.csdn.net/RaeZhang?type=blog)</small>

## 问题描述

前段时间在做需求的时候，遇到一个关于`forEach`和`await`的坑，记录一下。

在业务代码中存在一段很普通的同步循环逻辑

```javascript
function handleValueSync(value) {
  console.log(value);
}

const list = [1, 2, 3, 4, 5, 6, 7, 8];

list.forEach((value) => handleValueSync(value));

// 结果，1，2，3，4，5，6，7，8
```

某一天，需求突然产生变化,原本同步的 handleValueSync 方法变成了异步的，所以我在原先代码基础上进行了简单的修改：

```javascript
function handleValueAsync(value) {
  if (value % 2 === 0) {
    Promise.resolve().then(() => {
      console.log(`${value}`);
    });
  } else {
    console.log(value);
  }
}

let list = [1, 2, 3, 4, 5, 6, 7, 8];

list.forEach(async (value) => await handleValueAsync(value));
```

预期: 每次循环时，一定会 `await handleValueAsync`结束，然后开始下一次循环。

```javascript
// 预期结果 1，2，3，4，5，6，7，8
```

实际输出结果却是
![image](./img/forEach/wrong.png)
await 似乎并没有生效。
换成 map 和 reduce 试一下

```javascript
list.map(async (value) => await handleValueAsync(value));

list.reduce(async ({}, value) => await handleValueAsync(value), {});
```

也都是同样的结果
![image](./img/forEach/wrong2.png)

## 怎样使得 await 生效

网上搜索了一番，发现`forEach`就是不能支持异步调用，解决这个方法就是将 forEach 方法换成普通 for 循环就可以了。

```javascript
function handleValueAsync(value) {
  if (value % 2 === 0) {
    Promise.resolve().then(() => {
      console.log(`${value}`);
    });
  } else {
    console.log(value);
  }
}

let list = [1, 2, 3, 4, 5, 6, 7, 8];

async function awaitLoop() {
  for (let i = 0; i < list.length; i++) {
    await handleValueAsync(list[i]);
  }
}
awaitLoop();
```

结果
![image](./img/forEach/right.png)

这下就对了。。

## Why？

为什么 forEach 无法支持 await，而 for 循环可以呢？
搜了一下 forEach 的实现原理，原来 forEach 是 es6 的新语法。

> forEach() 是在第五版本里被添加到 ECMA-262 标准的；这样它可能在标准的其他实现中不存在，你可以在你调用 forEach() 之前插入下面的代码，在本地不支持的情况下使用 forEach()。
>
> 看下 MDN 上的[forEach 的 pollyfill 源码](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach)

```javascript
// Production steps of ECMA-262, Edition 5, 15.4.4.18
// Reference: http://es5.github.io/#x15.4.4.18
if (!Array.prototype.forEach) {
  Array.prototype.forEach = function (callback, thisArg) {
    var T, k;

    if (this == null) {
      throw new TypeError(" this is null or not defined");
    }

    // 1. Let O be the result of calling toObject() passing the
    // |this| value as the argument.
    var O = Object(this);

    // 2. Let lenValue be the result of calling the Get() internal
    // method of O with the argument "length".
    // 3. Let len be toUint32(lenValue).
    var len = O.length >>> 0;

    // 4. If isCallable(callback) is false, throw a TypeError exception.
    // See: http://es5.github.com/#x9.11
    if (typeof callback !== "function") {
      throw new TypeError(callback + " is not a function");
    }

    // 5. If thisArg was supplied, let T be thisArg; else let
    // T be undefined.
    if (arguments.length > 1) {
      T = thisArg;
    }

    // 6. Let k be 0
    k = 0;

    // 7. Repeat, while k < len
    while (k < len) {
      var kValue;

      // a. Let Pk be ToString(k).
      //    This is implicit for LHS operands of the in operator
      // b. Let kPresent be the result of calling the HasProperty
      //    internal method of O with argument Pk.
      //    This step can be combined with c
      // c. If kPresent is true, then
      if (k in O) {
        // i. Let kValue be the result of calling the Get internal
        // method of O with argument Pk.
        kValue = O[k];

        // ii. Call the Call internal method of callback with T as
        // the this value and argument list containing kValue, k, and O.
        callback.call(T, kValue, k, O);
      }
      // d. Increase k by 1.
      k++;
    }
    // 8. return undefined
  };
}
```

简化一下，其实 `forEach`就是通过`while`循环多次执行 callback，相当于在 for 循环中执行了异步函数,`forEach`可以看成是这样

```javascript
for (let i = 0; i < list.length; i++) {
  (async function (value) {
    if (value % 2 === 0) {
      Promise.resolve().then(() => {
        console.log(`${value}`);
      });
    } else {
      console.log(value);
    }
  })(list[i]);
}
```

所以`await`不起作用

## 参考链接

[当 async/await 遇到 map 和 reduce](https://blog.csdn.net/weixin_33725239/article/details/91390618)
[forEach 和 await/async 的问题](https://www.cnblogs.com/chrissong/p/11247827.html)
[在 foreach 中使用 async/await 的问题](https://blog.csdn.net/yumikobu/article/details/84639025)
