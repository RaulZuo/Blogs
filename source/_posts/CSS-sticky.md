---
title: CSS布局技巧 -- STICKY属性
date: 2018-12-23 00:21:58
tags:
  - CSS
categories: 前端基础
---

在一些很长的表格中，往往需要使用表头悬浮的设计以方便用户使用，例如H5电商页面通过下滑展示大量商品列表时，顶部的导航栏需要在离开屏幕时，需要固定在屏幕顶部以方便用户筛选类别。这种效果一直以来需要通过JavaScript来实现，此外，由于设置对应的DOM对象为fixed时会脱离常规流，会导致下部元素对象瞬间向上移动，影响用户体验。CSS3中的position:sticky为解决这些问题而生。

### position:sticky用法

sticky是CSS3的一个新属性，对象在常态时遵循常规流，如relative属性。但是，当对象滑动到屏幕外时则吸附在屏幕中设置的固定位置，如fixed属性。简而言之，sticky属性就像relative和fixed的合体，同时很好的解决对象脱离常规流时带来的下部对象瞬间偏移的问题。

```css
.sticky {
    position: -webkit-sticky;
    position:    -moz-sticky;
    position:     -ms-sticky;
    position:         sticky;
    top: 1px;
}
```

> 由于样式尚未进入标准，必须使用私有前缀。

### 浏览器兼容性

虽然很多浏览器不支持sticky属性，但是可以通过JavaScript简单实现这种效果：

![浏览器兼容性](/images/css_sticky.png)

```css
.sticky {
    position: -webkit-sticky;
    position:    -moz-sticky;
    position:     -ms-sticky;
    position:         sticky;
}

.header {
    width: 100%;
    height: 20px;
    background: #fcc;
}

.empty {
    width: 100%;
    height: 20px;
}

.hidden {
    display: none;
}
```

### 实例

```html
<section>
    <div class="header"></div>
    <div class="empty hidden"></div>
</section>
```

```js
var $win = $(window),
      $header = $('.header'),
      $empty = $('.empty');

$win.on('scroll', function(event) {
     if($win.scrollTop() > $header.offset().top) {
         $header.addClass('sticky');
         $empty.removeClass('hidden');
     } else if($win.scrollTop() < $empty.offset().top) {
         $header.removeClass('sticky');
         $empty.addClass('hidden');
     }
});
```

其中，当header对象脱离常规流时，使用empty元素占位，消除下部元素快速向上移动的问题。

如有错误，各位大牛敬请指出！