---
title: CSS布局技巧 -- 纯CSS让子元素的宽度总和决定其父元素的宽度
date: 2018-12-23 16:19:26
tags:
  - CSS
categories: 前端基础
---

### 使用场景

在移动端屏幕宽度有限的前提下，使用横向滚动的方式展示更多的内容。在这样的需求下，希望父元素作为容器，其宽度可以又横向排列资源的总宽度**动态撑开**，超过祖父元素的宽度；在不超过祖父元素时，自动继承100%的宽度。

DOM结构如下：

```html
<div class="grantparent">
  <div class="parent">
    <div class="child"></div>
    <div class="child"></div>
    <div class="child"></div>
    <div class="child"></div>
    <div class="child"></div>
  </div>
</div>
```

### 一般处理方法

* 将子元素设为`float`或者`inline-block`，然后再通过js计算子元素的个数和其宽度，从而设置父元素的宽度
* 不利因素
  * 增加DOM操作
  * js重新设定属性增加渲染重绘次数
  * float在渲染时计算量比较大

### 纯CSS处理方法

* 设置父元素的属性

```css
white-space: nowrap;
display: inline-block;
```

* 设置子元素的属性

```css
display: inline-block;
```