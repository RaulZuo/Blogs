---
title: CSS布局技巧 -- 内凹圆角
date: 2018-12-23 16:34:01
tags:
  - CSS
categories: 前端基础
---

圆角，相信每一个了解CSS属性的都知道，通过border-radius实现圆角（外凸圆角），但是如果需要实现**内凹圆角**怎么办呢？比如四角内凹的元素，比如如下所示这样的内凹圆角

![内凹圆角](http://images2015.cnblogs.com/blog/936737/201612/936737-20161204143241099-912365423.png)

对于这种问题，很多人的反应都是采用CSS的伪类或者子元素的**绝对定位**来覆盖，但是这样做的后果就是被覆盖部分是不透明的。在不同样色的背景下，会出现非常突兀的覆盖元素，这样一来，视觉上将会非常难看，自适应行不强。

如果需要实现透明的效果，很多人又会说，**切图**！对，切图作为background-image是一个很好的解决方法。如果只能使用CSS属性来实现，是不是就懵逼了。。。

下边就来介绍一种使用**纯CSS实现这种背景透明的内凹圆角**效果

首先先介绍两个CSS3的属性

- 线性渐变 **linear-gradient()**

  从该属性名中可以看出，这是一个生成颜色渐变图片的CSS方法。根据渐变基准方向（Gradient line）和颜色点，其中变化颜色点可以有多个。

  示例：

  ```css
  // 两种颜色渐变
  background: linear-gradient(90deg, #F6327C, #DF3DF0);

  // 两种以上颜色渐变，三个颜色点，在50%处有颜色#FF0，剩余的两个分别是起始点和结束点
  background: linear-gradient(90deg, #F6327C, #FF0 50%, #DF3DF0);
  ```

> 渐变基准方向若使用 to left/to right/to top/to bottom 这样属性时，注意-webkit-等内核兼容性方面的语法的变化，细节可以查看[文档](https://developer.mozilla.org/en-US/docs/Web/CSS/linear-gradient)

- 径向渐变 **radial-gradient()**

  顾名思义，从文字可以明白所谓的径向渐变就是以某点为圆心，固定直径内颜色渐变。它的属性包含了起始位置、方向、颜色渐变梯度，径向梯度允许变化的形状和大小。详细内容可以查看[文档](https://developer.mozilla.org/en-US/docs/Web/CSS/radial-gradient)

  其语法为：

  ```css
  radial-gradient([[ circle || <length>] [at <position> ]?, | [ ellipse || [ <length> | <percentage> ]{2} [ at <position> ]?, | [ [ circle | ellipse] || <extent-keyword> ] [at <position> ]?, | at <position> ,]? <color-stop> [, <color-stop>] +)
  where <extent-keyword> = closest-corner | closest-side | farthest-corner | farthest-side and <color-stop>     = <color> [ <percentage> | <length> ]? 
  ```

  示例：

  ```css
  // 椭圆渐变
  background-image: radial-gradient(ellipse farthest-corner at 45px 45px , #00FFFF 0%, rgba(0, 0, 255, 0) 50%, #0000FF 95%);

  // 固定半径的圆渐变
  background-image:  radial-gradient(16px at 60px 50% , #000000 0%, #000000 14px, rgba(0, 0, 0, 0.3) 18px, rgba(0, 0, 0, 0) 19px); 
  ```

### 言归正传，回到内凹圆角

首先主元素左右两边留有固定大小的margin值，然后使用圆角元素覆盖对应的margin区域。若圆角元素不大于2个，可以使用**::after**和**::before**两个伪类元素。若大于4个，如四角内凹元素，则采用自元素的绝对定位方式。

DOM元素结构（采用伪类）：

```html
  <div class="main"></div>
```

CSS结构分以下2个方面来进行设计：

1. 主体元素（背景颜色渐变）

  ```css
    .main {
      position: relative;
      width: 200px;
      height: 40px;
      margin: 0 5px;
      background: -webkit-linear-gradient(left, #F6327C, #DF3DF0);
      background: linear-gradient(to right, #F6327C, #DF3DF0);
    }
  ```

2. 内凹圆角元素（使用伪类）

  ```css
  .main::before {
    position: absolute;
    content: "";
    display: block;
    position: absolute;
    top: 0;
    left: -5px;
    width: 5px;
    height: 40px;
    border-radius: 2px 0 0 2px;
    background: -webkit-radial-gradient(10px at left,transparent 50%,#F6327C 50%);
    background: radial-gradient(10px at left,transparent 50%,#F6327C 50%);
  }
  .main::after {
    position: absolute;
    content: "";
    display: block;
    position: absolute;
    top: 0;
    right: -5px;
    width: 5px;
    height: 40px;
    border-radius: 0 2px 2px 0;
    background: -webkit-radial-gradient(10px at right,transparent 50%,#F6327C 50%);
    background: radial-gradient(10px at right,transparent 50%,#F6327C 50%);
  }
  ```

根据以上代码就能够实现如图所示的带透明内凹圆角效果。