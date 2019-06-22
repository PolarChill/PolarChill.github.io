---
layout: post
title: Promise And EventLoop
subtitle: Promise学习笔记
date: 2019-05-29
author: BY
header-img: img/post-bg-cook.jpg
catalog: true
tags:
  - JavaScript
---

### Promise

#### Promise 是什么

1. 异步编程的解决方案, 将异步操作以同步操作的流程表达出来, 避免了层层嵌套的回调函数
2. 是一个对象, 表示一个异步操作的最终状态,以及结果值
3. Promise 是一个代理（代理一个值），被代理的值在 Promise 对象创建时可能是未知的。它允许你为异步操作的成功和失败分别绑定相应的处理方法（handlers）。 这让异步方法可以像同步方法那样返回值，但并不是立即返回最终执行结果，而是一个能代表未来出现的结果的 promise 对象。

> 本质: Promise 是一个绑定了回调的对象，而不是将回调传进函数内部。MDN(结合理解上述第一点)

#### 特点

1. 三种状态, 一旦从等待状态改为其它状态就再不能改变了
   - pedding 等待中
   - resolved 完成了
   - rejected 拒绝了
2. 构造 Promise 的时候, 构造函数中的代码是立即执行的
3. Promise 实现了链式调用, 每次调用`then`之后都会返回一个全新`Promise`对象, 如果在`then`中使用了`return`, 则`return`值会被`Promise.resolve()`包装

#### 使用方法

Promise 对象时一个构造函数, 当 Promise 实例生成后, 可以调用 then 方法指定 resolved 的状态和 rejected 的回调函数

```js
const promise = new Promise((resolve, reject) => {
    // 构造函数中的代码
    if(/*异步操作成功*/) {
        // 将Promise对象状态从PEDDING改为RESOVLE成功
        resolve(result)
    } else {
        reject(eror)
    }
})

promise.then(successCb, failCb)  // 处理异步返回的结果(第二个失败的回调函数可选)

// 2.原型方法
Promise.prototype.catch(onRejected)
Promise.prototype.then(onFulfilled, onRejected)
Promise.prototype.finally(onFinally) // ES2018

// 3.静态方法
Promise.all(iterable);
//网络超时作竞争
Promise.race(iterable);
//手动创建一个已经resolve或者reject的promise快捷方法
Promise.reject(reason);
Promise.resolve(value);
```

Promise 实例具有`then`方法, 也就是`then`方法是定义在原型对象的 Promise.prototype 上. 作用: 是为 Promise 实例添加状态改变时的回调函数(处理 Promise 返回的结果)

`Promise.prototype.catch` 语法糖, 本质是`.then(null, rejection)`或`.then(undefined, rejection)`的别名，用于指定发生错误时的回调函数。

Promise 的链式调用

#### 实现一个简易的 Promise

关键点: `resolvedCallbacks` 和 `rejectedCallbacks` 用于保存 `then` 中的回调==，因为当执行完`Promise` 时状态可能还是等待中，这时候应该把 `then` 中的回调保存起来用于状态改变时使用==

```js
const PEDDING = 'pedding';
const RESOLVED = 'resolved';
const REJECTED = 'rejected';
function MyPromise(fn) {
  // 代码可能会异步执行, 保存this
  const that = this;
  that.state = PEDDING; //初始状态
  that.value = null; //保存resolve或reject返回的值
  that.resolvedCallBacks = []; // 保存then中的回调
  that.rejectedCallBacks = [];

  // resolve函数
  function resolve(value) {
    if (that.state === PEDDING) {
      that.state = RESOLVED;
      that.value = value;
      that.resolvedCallBacks.map(cb => cb(that.value));
    }
  }
  // rejecte函数
  function reject(value) {
    if (that.state === PEDDING) {
      that.state = REJECTED;
      that.value = value;
      that.rejectedCallBacks.map(cb => cb(that.value));
    }
  }
  // 执行
  try {
    fn(resolve, reject);
  } catch (error) {
    reject(e);
  }
}

MyPromise.prototype.then = function(onFulfilled, onRejected) {
  const that = this;
  // 判断传入的是否为函数, 如果不是函数, 需要手动创建, 防止穿透现象
  // 只是作为一个透传的例子
  // Promise.resolve(4).then().then((value) => console.log(value)) 本例子中会报错
  onFulfilled = Object.prototype.toString.call(onFulfilled) === '[object Function]' ? onFulfilled : v => v;
  onRejected =
    Object.prototype.toString.call(onFulfilled) === '[object Function]'
      ? onRejected
      : r => {
          throw r;
        };
  // 当状态是等待状态, 就将回调函数放到回调数组中, 等待执行
  if (that.state === PEDDING) {
    that.resolvedCallBacks.push(onFulfilled);
    that.rejectedCallBacks.push(onRejected);
  }

  if (that.state === RESOLVED) {
    onFulfilled(that.value);
  }

  if (that.state === REJECTED) {
    onRejected(that.value);
  }
};

new MyPromise((resolve, reject) => {
  setTimeout(() => {
    resolve(1);
  }, 0);
}).then(value => {
  console.log(value);
});
```

- value 返回值 onFulFilledCb onRejctedCb 数组 resovle 函数 reject 函数
- 原型的 then 方法实现, 注意判断类型, 根据 promise 状态执行对应的回调函数, 如果还在 padding 中, 将回调方法放入到回调数组中, resolve 或 reject 后, 执行回调方法

- [ ] 待完善 Promise A+的实现

### EventLoop

#### 什么是 EventLoop

> 一种机制用来处理程序中多个快的执行,且执行每个块时调用 JavaScript 引擎, 这种机制被称为事件循环 (你不知道的 JS p.142)

执行栈: 函数的调用执行,形成了一个栈帧(保存执行的相关信息), 压栈出栈操作

```js
while (queue.waitForMessage()) {
  queue.processNextMessage();
}
```

同步任务正常执行, 遇到异步任务先挂起放到待执行的 Task 队列中, 可以分为宏任务队列`macrotask`和微任务队列`microtask`, ==**一旦执行栈为空**==, 就从任务队列中拿出需要执行的代码, 并放入执行栈中执行

![事件循环](media/15602655206229/15603448663256.jpg)

#### 微任务和宏任务

微任务:`promise`, `process.nextTick` ，`MutationObserver`，其中 process.nextTick 为 Node 独有。

宏任务: `script`, `setTimeout`, `setInterval`, `setImmediate`, `IO`, `UI rendering`

#### EventLoop 执行顺序

1. **首先执行同步代码块**
2. **当执行完同步代码块后, 执行栈为空, 查询是否有异步任务需要执行**
3. ==执行所有微任务==
4. **当执行完所有微任务后, 有必要会渲染页面**
5. **开始下一轮 EventLoop, 执行宏任务中的代码**

总的来说: 每一次事件循环中，主进程都会先执行一个 MacroTask 任务，这个任务就来自于所谓的 MacroTask Queue 队列；当该 MacroTask 执行完后，Event loop 会立马调用 MicroTask 队列的任务，直到消费完所有的 MicroTask，再继续下一个事件循环。这也就是 setTimeOut 会比 Promise 后执行的原因。

#### 待完善 Node 中的 EventLoop

### Promise 面试题

```js
// 写出运行结果
async function async1() {
  console.log('async1 start');
  await async2();
  console.log('async1 end');
}

async function async2() {
  console.log('async2');
}

console.log('script start');

setTimeout(function() {
  console.log('settimeout');
});

async1();

new Promise(function(resolve) {
  console.log('promise1');
  resolve();
}).then(function() {
  console.log('promise2');
});

console.log('script end');
```

### 参考

- [MDN Promise](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)
- [MDN 使用 Promises](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Using_promises)
- [ECMAScript 6 入门 Promise](http://es6.ruanyifeng.com/?search=map&x=0&y=0#docs/promise)
- [你真的会用 Promise 吗？](https://juejin.im/post/5cf7857ff265da1bac4005b2#heading-8)
