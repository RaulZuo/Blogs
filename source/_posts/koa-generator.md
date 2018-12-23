---
title: Koa源码分析（一） -- generator
date: 2018-12-23 16:44:20
tags:
  - nodejs
  - koa
categories:
  - 源码解析
---

## Abstract

本系列是关于Koa框架的文章，目前关注版本是**Koa v1**。主要分为以下几个方面：
  
  1. [Koa源码分析（一） -- generator](/koa-generator/)
  2. [Koa源码分析（二） -- co的实现](/koa-co/)
  3. [Koa源码分析（三） -- middleware机制的实现](/koa-middleware)

## Genetator函数

Generator函数是ES6提供的一种异步编程解决方案，其语法行为完全不同于传统函数。

> 详细解析可见阮一峰老师的[《ECMAScript 6 入门 --- Generator》](http://es6.ruanyifeng.com/#docs/generator)

### 语法

两大特征:

1. `function` 关键字与函数名之间的 `*`
2. 函数体内部使用 `yield` 语句

我们定义一个generatorFunction示例：

```js
function* firstGenerator() {
  var one = yield 'one';
  console.log(one);
  var two = yield 'two';
  concole.log(two);
  var third = yield 'third';
  console.log(third);
  return 'over'
}
```

带有 `*` 的函数声明就代表 `firstGenerator` 函数是一个generator函数，函数里面的 yield 关键字可以理解为在当前位置设置断点，这一点如有疑问，可以看后续。

### 语法行为

那generator函数的语法行为究竟与传统函数不同在哪里呢？下边我们来梳理下generator函数的运行步骤。

```js
var it = firstGenerator();
console.log(it.next(1));  // {value: "one", done: false}
console.log(it.next(2));  // {value: "two", done: false}
console.log(it.next(3));  // {value: "third", done: false}
console.log(it.next(4));  // {value: "over", done: true}
```

首先通过执行 `firstGenerator` 函数，我们可以得到一个generator对象 `it`，它是一个迭代器对象。此时， `firstGenerator` 函数并**未执行**，只是返回了迭代器对象 `it` ，我们可以通过 `it` 对象中 `next` 方法触发 `firstGenerator` 函数开始执行，此时我们调用 `it.next(1)`，注意 `next` 注入的参数 `1` 并没有任何效果。当 `firstGenerator` 函数执行到 `yield` 语句时，被打了断点，停留在此处，并将 `yield` 后的变量传递给 `it.next(1)` 结果对象中的 `value` 字段，另外其中的 `done` 字段表示 `firstGenerator` 函数尚未完全执行完，还停留在断点。以此同时，将执行权交换给 `it.next(1)`。

执行第二次 `it.next(2)`，执行权再次交给 `firstGenerator` 函数，并将 `next` 带入的参数传递给函数中的变量 `one`，此时输出 `2`。当运行到 `yield 'two'` 语句时，再次将执行权返回给 `it.next(2)` 并传值。

第三次执行 `it.next(3)`的过程与第二次完全一样。

最后一次执行 `it.next(4)` 时，在此之前， `firstGenerator` 函数断点在 `var third = yield 'third'`，当 `it.next(4)` 将执行权交给 `firstGenerator` 函数时，将 `4` 传递给变量 `third`，此刻输出 `4`。当执行到 `return` 语句时，整个函数已经执行完了，并将 `'over'`传递给 `it.next(4)` 返回结果中的 `value` 字段，且 `done` 字段为 `true`。若没有 `return` 语句，则 `value` 字段返回 `null`。

这样下来，整个 `firstGenerator` 整个函数执行完毕。我们可以将Generator函数比喻成**懒惰的癞蛤蟆**，每次都需要使用**it.next()**方法戳一下，才会有对应的行动。如果大家了解python中协程的概念，应该很好理解Generator函数的语法行为。

### 进阶语法

在Generator函数中 `yield` 的语法，其后边的值也可以是函数、对象等等。 `yield` 后边可以是另一个Generator对象。

```js
function* subGen() {
  console.log('step in sub generator');
  var b = yield 'sub 1';
  console.log(b);
  console.log('step out sub generator');
}
var subGenerator = new subGen();
function* mainGen() {
  var a = yield 'main 1';
  console.log(a);
  var b = yield *subGenerator;
  console.log(b);
  var c = yield 'main 2';
  console.log(c);
  return 'over';
}
var it = mainGen();
console.log(it.next(1));
// {value: 'main 1', done: false}
console.log(it.next(2));
// 2
// step in sub generator
// {value: 'sub 1', done: false}
console.log(it.next(3));
// 3
// step out sub generator
// null
// {value: 'main 2', done: false}
console.log(it.next(4));
// 4
// {value: 'over', done: true}
```

`yield` 后面跟着 `*subGenerator` 对象，这等同于断点就进入 `subGenerator` 的 `subGen`里面，等待 `subGen` 全部执行完后再回来继续执行，类似于递归。
直白点说，就是将 `subGen` 内嵌到 `mainGen`中。

**在`subGen`函数中的return语句并不会起到断点的作用**

### 结束语

Generator函数作为ES6的新特性，通过它可以很好的解决JavaScript中的**恶魔**回调问题。Koa框架使用这特性很好组织了异步代码的结构。
