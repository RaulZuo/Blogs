---
title: CSS布局技巧 -- 各种居中
date: 2018-12-23 16:23:16
tags:
  - CSS
categories: 前端基础
---

### 多行垂直居中

废话少说，直接上例子!!!

- display:table

  Html代码：

  ```html
  <div class="wrapper">
    <div class="content">
    <span>我是很短的文本</span>
    <span>我是很长的文本，内容非常多多多多多多多多多多多多...</span>
    <div>我只是一个div标签</div>
      <img src="..." style="width:200px;height:200px;"/>
    </div>
  </div>
  ```

  CSS代码：

  ```css
    .wrapper{display:table;height:800px;width:300px;}
    .content{display:table-cell;vertical-align:middle;text-align:center;}
    //文本内容不足一行时居中，内容多行时左对齐
    .content span{display:inline-block;text-align:left;}
  ```

  虽然对于现代浏览器都能够生效，但是在奇葩的IE6-7下就无法正常运行。在这里为了兼容这些特殊的情况，必然要使用另外一种相对定位和绝对定位的方式，在使用IE的特有的条件语法的同时，也可以使用ie hack来写。

  兼容IE的CSS代码：

  ```css
    .wrapper{display:table;height:800px;width:300px;position:relative;}
    .content{display:table-cell;vertical-align:middle;text-align:center;*position:absolute;*top:50%;*left:50%;}
    //文本内容不足一行时居中，内容多行时左对齐
    .content span{display:inline-block;text-align:left;}
    .content *{position:relative;*top:-50%;*left:-50%;}
  ```

  **优点：**
  wrapper的高度没有限制，可以自适应，根据内部元素动态的改变高度
  **缺点：**
  结构复杂，需要增加额外的标签，对于IE6-7浏览器需要额外的兼容

  以上结构能满足所有的多行内容的垂直水平居中，对于单行垂直水平居中的情况，实现相对更加简单，即通过line-height来实现。
  
  ---

- float属性

  在child元素之前插入一个div元素，使其left浮动，高度为parent元素的50%，同时设置margin-bottom为`负`的child元素高度的`一半`，然后再child元素中清除浮动。这样，child元素就相对parent元素垂直居中。

  **No Code No Truth**

  Html代码：

  ```html
    <div class="wrapper">
      <div class="floater"></div>
      <div class="content">Contents</div>
    </div>
  ```

  CSS代码：

  ```css
    .floater{float:left;height:50%;margin-bottom:-100px;}
    .content{clear:both;height:240px;position:relative;}
  ```

  **优点：**

  不存在兼容问题，所有浏览器都适应；内部元素的高度需要固定

  **缺点：**

  需要插入额外的空元素

  ---