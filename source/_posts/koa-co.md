---
title: Koa源码分析（二） -- co的实现
date: 2018-12-23 16:53:58
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

## co

大名鼎鼎的co是什么？它是[TJ大神](https://github.com/tj)基于ES6的一些新特性开发的异步流程控制库，基于它所开发的[koa](https://github.com/koajs/koa)被视为未来主流的web框架。

koa基于co实现，而co又是使用了ES6的generator和promise特性。如果还不理解，可以查看阮一峰老师的[《ECMAScript 6 入门 --- Generator》](http://es6.ruanyifeng.com/#docs/generator)和[《ECMAScript 6 入门 --- Promise》](http://es6.ruanyifeng.com/#docs/promise)。目前co升级为4.X版本事，代码进行了一次颇有规模的重构，我们主要关注co（4.X）的实现思路和源码分析。

## 使用示例

```js
co(function* (){
    var a = yield Promise.resolve('one');
    console.log(a);
    var b = yield Promise.reslove('two');
    console.log(b);
    return 'three';
}).then((value) => console.log(value));
// one
// two
// three
```

```js
co(function* (){
    var res = yield [Promise.resolve(1), Promise.resolve(2), Promise.resolve(3)];
    return res;
}).then((value) => console.log(res));
// [1, 2, 3]
```

根据co的功能，它作为异步流程控制的作用，自动调用generator对象的next()方法，实现generator函数的运行，并返回最终运行的结果。

如果要涉及到co的实现细节，我们就会存在以下几个疑问：

1. 如何依次调用next()方法
2. 如何将**yield**后边运算异步结果返回给对应的变量
3. co自身如何返回generator函数最后的return值

接下来我们正对以上问题，分析TJ大神的源码

## 源码解析

### co源码的流程控制

```js
function co(gen) {
  // 保持当前函数的上下文
  var ctx = this;
  // 截取co输入的参数，剔除arguments中的第一个参数，即gen对象，剩余参数作为gen的入参
  var args = slice.call(arguments, 1);

  // 返回一个Promise对象，即最外围Promise对象
  return new Promise(function(resolve, reject) {
    // 判断传入的gen是否为函数，若是则执行，将结果赋值给gen对象
    // 若不是，则不执行
    if (typeof gen === 'function') gen = gen.apply(ctx, args);
    // 根据generator函数执行结果是否存在next字段，判断gen是否为generator迭代器对象
    // 若不是，则调用resolve返回最外围Promise对象的状态
    if (!gen || typeof gen.next !== 'function') return resolve(gen);

    // 若是generator迭代器对象，开始控制gen.next()方法的调用
    onFulfilled();

    // 两个用途
    // 一、generator函数的执行入口
    // 二、当做所有内部Promise对象的resolve方法，处理异步结果，并继续调用下一个Promise
    function onFulfilled(res) {
      var ret;
      try {
        // gen运行至yield处被挂起，开始处理异步操作，并将异步操作的结果返回给ret.value
        ret = gen.next(res);
      } catch (e) {
        // 若报错，直接调用reject返回外围Promise对象的状态，并传出错误对象
        return reject(e);
      }
      // 将gen.next的执行结果传入next函数，实现依次串行调用gen.next方法
      next(ret);
      return null;
    }

    // 当做所有内部Promise对象的reject方法，处理异步结果，并继续调用下一个Promise
    function onRejected(err) {
      var ret;
      try {
        ret = gen.throw(err);
      } catch (e) {
        // 若报错，直接调用reject返回外围Promise对象的状态，并传出错误对象
        return reject(e);
      }
      // 将gen.throw的执行结果传入next函数，实现依次串行调用gen.next方法
      next(ret);
    }

    // 实现串行调用gen.next的核心
    function next(ret) {
      // 判断内部Promise是否全部执行完毕
      // 若执行完毕，直接调用resolve改变外围Promise的状态，并返回最终的return值[问题3]
      if (ret.done) return resolve(ret.value);
      // 若未执行完毕，调用toPromise方法将上一个Promise返回的值转化为Promise对象
      // 具体参见toPromise方法
      var value = toPromise.call(ctx, ret.value);
      // 根据value转化后的Promise对象的两个状态，执行下一个next方法
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
      // 抛出不符合转化规则的类型的值
      return onRejected(new TypeError('You may only yield a function, promise, generator, array, or object, '
        + 'but the following object was passed: "' + String(ret.value) + '"'));
      }
    });
  }
```

源码分析完了，我们可以把co串行调用generator函数中yield的过程总结如下：

1. 进入外围Promise
2. 通过入口onFilfilled()方法，将generator函数运行至第一个yield处，执行该yield后边的异步操作，并将结果传入next方法
3. 如果next中传入结果的done为true，则返回外围Promise的resolve
4. 如果next中传入结果的done为true，则返回value（即yield后边的对象）是否可以转化为内部Promise对象。如无法转化，则抛出错误，返回外围Promise的reject
5. 若能转化为Promise对象，将所有内部Promise并行执行，通过then(onFilfilled, onRejected)开始执行
6. 在onFilfilled()或者onRejected()内部调用再次调用next()方法，实现串行执行yield，并肩yield后边的对象传递给next()，依次重复。
7. 所有yield执行返回，将最后的return值返回给外围Promise的resovle方法，结束co对generator函数的调用

### yield后面对象转化为Promise

能够在co中实现generator函数的逐步调用next()方法，转化为内部Promise将至关重要，而源码是如何转化的呢？哪些对象又是能够转化的呢？接下来，我们看下源码。

```js
function toPromise(obj) {
  // 确保obj有意义
  if (!obj) return obj;
  // 若是Promise对象，则直接返回
  if (isPromise(obj)) return obj;
  // 若是generator函数或者generator对象，则传入一个新的co，并返回新co的外围Promise
  // 作为当前co的内部Promise，这样实现多层级调用
  if (isGeneratorFunction(obj) || isGenerator(obj)) return co.call(this, obj);
  // 若是函数，则返回thunk规范的函数
  if ('function' == typeof obj) return thunkToPromise.call(this, obj);
  // 若是数组，把数组中每个元素转化为内部Promise，返回Promise.all并行运算
  if (Array.isArray(obj)) return arrayToPromise.call(this, obj);
  // 若是对象，遍历对象中的每个key对应的value，转化成Promise.all并行运算
  if (isObject(obj)) return objectToPromise.call(this, obj);
  return obj;
}

function thunkToPromise(fn) {
  var ctx = this;
  return new Promise(function (resolve, reject) {
    fn.call(ctx, function (err, res) {
      if (err) return reject(err);
      if (arguments.length > 2) res = slice.call(arguments, 1);
      resolve(res);
    });
  });
}

function arrayToPromise(obj) {
  // Array.map并行计算返回每一个元素的Promise
  return Promise.all(obj.map(toPromise, this));
}

function objectToPromise(obj){
  var results = new obj.constructor();
  var keys = Object.keys(obj);
  var promises = [];
  for (var i = 0; i < keys.length; i++) {
    var key = keys[i];
    var promise = toPromise.call(this, obj[key]);
    if (promise && isPromise(promise)) defer(promise, key);
    else results[key] = obj[key];
  }
  // Promise链式调用，后续的then能偶获取此处的results
  return Promise.all(promises).then(function () {
    return results;
  });

  function defer(promise, key) {
    // key对应的元素成功转化为Promise对象后，构造这些Promise的resovle方法
    // 以便在results中获取每个Promise对象成功执行后结果
    results[key] = undefined;
    promises.push(promise.then(function (res) {
      results[key] = res;
    }));
  }
}

```

结合上述分析，我们可以得到，yield后面只能是**函数、Promise对象、Generator函数、Generator迭代器对象、数组（元素仅限之前的4类）和Object(对应value仅限定之前的4类)**