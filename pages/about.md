---
layout: page
title: About Me
description: 关于我
keywords: corkine，马章竞
comments: true
menu: 关于
permalink: /about/
---

Hello everyone! 

这里是我的个人博客，用来归类整理我在学习生活中遇到的问题和解决方法，请随意浏览。 

## 坚信

* Life is a struggle.
* 生活就像西西弗斯的神话。

## 联系

* GitHub：[@corkine](https://github.com/corkine)
* 博客：[{{ site.title }}]({{ site.url }})
* 主站：[www.mazhangjing.com](http://www.mazhangjing.com)

## 技能

#### 平面设计与影视制作
<div class="btn-inline">
    {% for keyword in site.skill_software_keywords %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>

#### 全栈工程师(伪)
<div class="btn-inline">
    {% for keyword in site.skill_mobile_app_keywords %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>

#### 生活技能和职业发展
<div class="btn-inline">
    {% for keyword in site.skill_windows_keywords %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>

<br>
Ps. 星号*为正在学习/练习的技能点。
