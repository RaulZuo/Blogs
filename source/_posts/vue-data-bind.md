---
title: Vue源码解析---数据的双向绑定
date: 2018-12-23 17:02:23
tags:
  - vue
categories:
  - 源码解析
---

> 本文主要抽离Vue源码中数据双向绑定的核心代码，解析Vue是如何实现数据的双向绑定
> 核心思想是ES5的**Object.defineProperty()**和**发布-订阅**模式

### 整体结构

1. 改造Vue实例中的data，通过Object.defineProperty()将其所有属性设置为**访问器属性**
2. 对每个属性添加**Observer**，并在observer中添加订阅者对象序列**Dep**
3. 添加订阅者对象**Watcher**，每次初始化的时候添加到对应data属性中的**Dep**之中

所有，我们从代码的角度将整体分为三个部分：**监听数据变化**、**管理订阅者**、**订阅者**

### 监听数据变化

使用ES5中的**Object.defineProperty**将data中的属性修改为**访问者属性**

```js
// Dep用于订阅者的存储和收集，将在下面实现
import Dep from 'Dep'
// Observer类用于给data属性添加set&get方法
export default class Observer{
  constructor(value){
    this.value = value
    this.walk(value)
  }
  walk(value){
    Object.keys(value).forEach(key => this.convert(key, value[key]))
  }
  convert(key, val){
    defineReactive(this.value, key, val)
  }
}
export function defineReactive(obj, key, val){
  // 用于存放某个属性的所有订阅者
  var dep = new Dep()
  // 给当前属性的值添加监听
  var chlidOb = observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: ()=> {
      console.log('get value')
      // 如果Dep类存在target属性，将其添加到dep实例的subs数组中
      // target指向一个Watcher实例，每个Watcher都是一个订阅者
      // Watcher实例在实例化过程中，会读取data中的某个属性，从而触发当前get方法
      if(Dep.target){
        dep.addSub(Dep.target)
      }
      return val
    },
    set: (newVal) => {
      console.log('new value setted')
      if(val === newVal) return
      val = newVal
      // 对新值进行监听
      chlidOb = observe(newVal)
      // 通知所有订阅者，数值被改变了
      dep.notify()
    }
  })
}
export function observe(value){
  // 当值不存在，或者不是复杂数据类型时，不再需要继续深入监听
  if(!value || typeof value !== 'object'){
    return
  }
  return new Observer(value)
}
```

### 管理订阅者

对订阅者进行收集，存储和通知

```js
export default class Dep{
  constructor(){
    this.subs = []
  }
  addSub(sub){
    // 在收集订阅者的时候，需要对subs中的订阅者进行去重，这边不详细解析
    this.subs.push(sub)
  }
  notify(){
    // 通知所有的订阅者(Watcher)，触发订阅者的相应逻辑处理
    this.subs.forEach((sub) => sub.update())
  }
}
```

### 订阅者

每个watcher对象都是对data中每个属性的订阅，是多对一的关系，每个watcher只能对应一个data属性，而一个data属性可以对应多个watcher

```js
import Dep from 'Dep'
export default class Watcher{
  constructor(vm, expOrFn, cb){
    this.vm = vm // 被订阅的数据一定来自于当前Vue实例
    this.cb = cb // 当数据更新时想要做的事情
    this.expOrFn = expOrFn // 被订阅的数据
    this.val = this.get() // 维护更新之前的数据
  }
  // 对外暴露的接口，用于在订阅的数据被更新时，由订阅者管理员(Dep)调用
  update(){
    this.run()
  }
  run(){
    const val = this.get()
    if(val !== this.val){
      this.val = val;
      this.cb.call(this.vm)
    }
  }
  get(){
    // 当前订阅者(Watcher)读取被订阅数据的最新更新后的值时，通知订阅者管理员收集当前订阅者
    Dep.target = this
    const val = this.vm._data[this.expOrFn]
    // 置空，用于下一个Watcher使用
    Dep.target = null
    return val;
  }
}
```

### 实例

下边我们创建一个简易的Vue来实际运行下对数据的监听

```js
import Observer, {observe} from 'Observer'
import Watcher from 'Watcher'
export default class Vue{
  constructor(options = {}){
    // 简化了$options的处理
    this.$options = options
    // 简化了对data的处理
    let data = this._data = this.$options.data
    // 将所有data最外层属性代理到Vue实例上
    Object.keys(data).forEach(key => this._proxy(key))
    // 监听数据
    observe(data)
  }
  // 对外暴露调用订阅者的接口，内部主要在指令中使用订阅者
  $watch(expOrFn, cb){
    new Watcher(this, expOrFn, cb)
  }
  _proxy(key){
    Object.defineProperty(this, key, {
      configurable: true,
      enumerable: true,
      get: () => this._data[key],
      set: (val) => {
        this._data[key] = val
      }
    })
  }
}
```

```js
import Vue from './Vue';
let demo = new Vue({
  data: {
    'a': {
      'ab': {
        'c': 'C'
      }
    },
    'b': [
      'bb': 'BB',
      'bbb': 'BBB'
    ],
    'c': 'C'
  }
});
demo.$watch('c', () => console.log('c is changed'));
// get value
demo.$watch('a.ab', () => console.log('a.ab is changed'));
demo.$watch('b', () => console.log('b is changed'));
// get value
demo.c = 'CCC';
// new value setted
// get value
// c is changed
demo.a.ab = 'AB';
// get value
// new value setted
demo.b.push({'bbbb': 'BBBB'});
// get value
```

根据实例的输出结果，我们很奇怪的发现，只有对简单的数据监听才能实现数据双向绑定。

1. `demo.$watch('a.ab', () => console.log('a.ab is changed'))`注册订阅者并没有调用`getter`
2. `demo.a.ab = 'AB'`有监听到数据的变化，并没有调用对应的callback
3. `demo.b.push({'bbbb': 'BBBB'})`对数值进行操作，并没有调用对应的callback

这是为什么呢？因为我们对数据的监听的实现，目前仅限于简单对应，对于某个属性内部有更多复杂属性时，就无能为力了。

为了实现进一步对数据和复杂对象的监听，请戳[Vue源码解析---数组的双向绑定]()和[Vue源码解析---复杂队形的双向绑定]()
