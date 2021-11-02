---
layout: page
title: About Me
description: 关于我
keywords: Chenqijin
comments: true
menu: 关于
permalink: /about/
---

Hello everyone! 

这里是我的个人博客，用来归类整理我在学习生活中遇到的问题和解决方法，请随意浏览。 

## 近期研究兴趣

* 自然语言处理和新闻传播学的交叉应用，包括但不限于：假新闻检测、自动文本摘要、情感分析
* 金融时间序列预测

## 联系

* GitHub：[@qj4chen](https://github.com/qj4chen)
* 博客：[{{ site.title }}]({{ site.url }})

## 技能

#### 机器学习与深度学习
<div class="btn-inline">
    {% for keyword in site.skill_software_keywords %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>

#### 金融与量化
<div class="btn-inline">
    {% for keyword in site.skill_mobile_app_keywords %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>

#### 经历
<div class="btn-inline">
    {% for keyword in site.skill_windows_keywords %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>

<br>
Ps. 星号*为正在学习/加强的技能点。
