---
layout: post
title: 动态公告牌 - 使用 Vue 和 JavaScript 实现
categories:
  - JavaScript
  - Vue
---

> 本文介绍了一种... 极其无聊的 Vue 单页应用 —— 公告牌 —— 包含一个漂亮的动态渐变背景和动态数字大小的单页应用。可以用来计数，以及任何对外显示数字、简要信息的地方。关键是，动态渐变背景很漂亮。

JavaScript 是 Web 前端的核心，尤其是当你希望自己的 CSS 渐变背景能自己动，但是使用 CSS 内置的动画又无法完成的时候。从某种角度来看，这是 HTML 和 CSS 标准 —— 更多的是历史的问题。Web 文档陈旧、笨重，但是似乎不可或缺 —— 从来没有过这样的便捷性，在几乎任何设备上，通过网址就可访问一个应用程序 —— Java 试图做，但是失败了。

这个公告牌的展示如下所示：

![](http://static2.mazhangjing.com/20190331/e7f6781_cm_image2019-03-3118.46.53.png)

# 面临的问题

## 背景问题

动态背景参照了 [渐变精选](http://go.mazhangjing.com/color) 这个网站。但是，如何动态的展示渐变，让它有种流动的感觉，我开始想要使用的是 CSS 的 Translate 动画，当 hover 时自动触发背景的变换，但是发现渐变过渡是被当做 background-image 处理的，这意味着不能通过动画实现。

最后我解决方法是使用 JavaScript 定时器，定时重写 CSS 样式。

至于背景平铺的问题，因为 Vue 难以获取 HTML 元素，因此，我使用的是 Absolute 定位。

## 数字问题

至于垂直水平居中，这个问题，我使用的是 Flex 来解决的。同时为溢出的部分进行裁剪。

## 循环动画实现问题

因为动画是用 JavaScript 模拟的，因此，需要手动完成循环往复的动画效果，而不能每次动画完毕之后，立刻回到起始位置，我们想要有一种流畅感 —— 使用了一个中间状态来执行循环往复的判断。

## 根据屏幕自适应问题

中间的数字要根据屏幕大小自适应，这个不好办，我使用的简单粗暴的方法，直接通过 ComputedStyle 获取在 Window 显示的整个 div 高度，然后重写数字的大小。

# 最终的实现

代码如下：

```vue
<template>
  <div id="app">
    <div id="number">
      {{number()}}
    </div>
    <span class="link">
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Host by abc.mazhangjing.com
    </span>
  </div>
</template>

<script>
import moment from 'moment';
export default {
  name: 'App',
  data() {
    return {
      a: 0,
      left_to_right: true
    }
  },
  methods: {
    number() {
      return moment().diff(moment('20180210','YYYYMMDD'), 'days');
    },
    updateBackground() {
      const app = document.querySelector("#app");
      if (this.a === 30) {
        this.left_to_right = false
      } else if (this.a === 0) {
        this.left_to_right = true
      }
      if (this.left_to_right) {
        this.a = this.a + 2;
      } else {
        this.a = this.a - 2;
      }
      app.style.background='linear-gradient(120deg, #a6c0fe ' + this.a + '%, #f68084 ' + (this.a + 100) + '%)';
    },
    setNumberSize() {
      const number = document.querySelector("#number");
      const appHeight = parseInt(window.getComputedStyle(document.querySelector("#app")).height);
      if (appHeight !== null && appHeight !== 0) number.style.fontSize = appHeight/2 + 'px'
    }
  },
  mounted() {
    setInterval(this.updateBackground, 500);
    this.setNumberSize();
  }
}
</script>

<style>
  #app {
    position: absolute;
    bottom: 0;
    top: 0;
    left: 0;
    right: 0;
    display: flex;
    justify-content: center;
    flex-direction: column;
    align-items: center;
    background: linear-gradient(120deg, #a6c0fe 0%, #f68084 100%);
    overflow: hidden;
  }
  #number {
    font: 30em sans-serif;
    text-align: center;
    color: white;
    opacity: .5;
  }
  .link {
    color: white;
    opacity: 0.3;
    font: 1.2em sans-serif;
    letter-spacing: 1px;
  }
</style>
```

# 吐槽

没什么吐槽的，HTML 和 CSS 是 Web 标准，虽然，居中这种事情，在现代浏览器上已经很方便了，但是，动态调整高度，更加强大的动画，相比较客户端技术，比如 Qt/C++/Python 和 Flutter/Dart、JavaFx/Java/Scala/Kotlin/Conjure/Groovy 还是很差。