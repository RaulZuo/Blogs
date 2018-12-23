---
title: Koa源码分析（三） -- middleware机制的实现
date: 2018-12-23 16:54:05
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
3. [Koa源码分析（三） -- middleware机制的实现](/koa-middleware/)

## Koa概括

Koa是基于generator与co之上的新一代中间件框架，它的优势主要集中在以下几个方面

1. 中间件机制
2. 封装了request/response, context对象
3. 使用yield，方便异步编程进行流程控制
4. 在忽略同步或者异步的情况下，使用try catch可以获取程序运行中的异常（错误处理是服务端程序的核心）

## 示例代码

```js
var Koa = require('koa');
var app = new Koa();
//添加中间件1
app.use(function *(next){
  var start = new Date;
  console.log("start=======1111");
  yield next;
  console.log("end  =======1111");
  var ms = new Date - start;
  console.log('%s %s - %s', this.method, this.url, ms);
});
//添加中间件2
app.use(function *(){
  console.log("start=======2222");
  this.body = 'Hello World';
  console.log("end  =======2222");
});

app.listen(3000);
/*
start=======1111
start=======2222
end  =======2222
end  =======1111
GET / - 10
start=======1111
start=======2222
end  =======2222
end  =======1111
GET /favicon.ico - 5
*/
```

从上述代码中，我们添加了两个middleware，其中第一个middleware中有一个输入参数`next`，并通过`yield`进行调用。通过分析输出的log信息，不难发现，先运行middelware1中的`yield`之前的代码，然后进入到middleware2中运行，待middleware2运行结束后又回到middleware1中，并运行`yield`之后的代码。

由于`app.use`输入的是generator函数，如果熟悉generator函数的同学，或许会说，这是将middleware2作为middleware1中的next参数，依次调用多个generator函数。对，没错，实际运行就是这样的，但是koa框架是如何组织代码实现这样方面的调用，将**地狱式调用**的异步编程编程这样清晰的结构？请看下文的源码分析

## 源码分析

### Application初始化

```js
function Application() {
  if (!(this instanceof Application)) return new Application;
  this.env = process.env.NODE_ENV || 'development';
  this.subdomainOffset = 2;
  // 用于存放中间件，即generator对象
  this.middleware = [];
  this.proxy = false;
  // 获得封装的上下文对象
  this.context = Object.create(context);
  // 获取封装的请求对象
  this.request = Object.create(request);
  // 获取封装的响应对象
  this.response = Object.create(response);
}
```

### 启动服务

```js
listen() {
  debug('listen');
  // 调用node原生中的创建服务
  // 其中callback()是服务创建的核心，具体见下面分析
  const server = http.createServer(this.callback());
  // 开启服务的监听
  return server.listen.apply(server, arguments);
}
```

### 添加中间件

```js
app.use = function(fn){
  if (!this.experimental) {
    // es7 async functions are not allowed,
    // so we have to make sure that `fn` is a generator function
    assert(fn && 'GeneratorFunction' == fn.constructor.name, 'app.use() requires a generator function');
  }
  debug('use %s', fn._name || fn.name || '-');
  // 将输入的fn依次push到middleware数组中
  this.middleware.push(fn);
  // 返回this，以便链式调用
  return this;
};
```

### node native 创建服务

```js
app.callback = function(){
  if (this.experimental) {
    console.error('Experimental ES7 Async Function support is deprecated. Please look into Koa v2 as the middleware signature has changed.')
  }
  // 将中间件按照加入的顺序，实现yield的链式调用，即组织异步调用结构，详细见下面的compose
  // co.wrap方法将generator函数转化为Promise
  var fn = this.experimental ? compose_es7(this.middleware) : co.wrap(compose(this.middleware));
  var self = this;

  if (!this.listeners('error').length) this.on('error', this.onerror);

  // 返回node native的请求处理函数
  return function handleRequest(req, res){
    res.statusCode = 404;
    var ctx = self.createContext(req, res);
    onFinished(res, ctx.onerror);
    fn.call(ctx).then(function handleResponse() {
      respond.call(ctx);
    }).catch(ctx.onerror);
  }
};
```

### 中间件异步构建

```js
// 返回一个启动函数
function compose(middleware){
  return function *(next){
    if (!next) next = noop();
    var i = middleware.length;
    // 对中间件队列从后遍历，逐个获取对应的generator对象
    while (i--) {
      // 将后面的generator对象传递给前面中间件的generatorFunction
      next = middleware[i].call(this, next);
    }
    // 返回一个yield，next指向第一个中间件的generator
    return yield *next;
  }
}
function *noop(){}
```

这样，我们就从返回的启动函数（generator函数）的yield处指向第一个中间件，然后从之前`while`循环构成的从前往后的调用链，依次调用下一个中间件，直至最后一个中间件然后再返回。

这边我们再次回到`callback()`这个启动函数处，调用`co.wrap()`实现对generator函数的逐步调用。
