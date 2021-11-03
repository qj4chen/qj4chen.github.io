---
layout: post
title: ”【论文解读】GCAN: Graph-aware Co-Attention Networks for Explainable Fake News Detection on Social Media”
categories:
  - 论文解读
  - Fake News
  - 可解释性

---



> 整体概括:  这篇论文研究的基于传播路径的假新闻探测问题, 研究的场景是社交媒体, 以Twitter为例. 本文提出模型, 通过新闻的传播路径(最源头的推文文本信息, 转推者(retweet)的用户信息等)来做二分类问题.
>
> 论文提出的模型叫做**Graph-aware Co-Attention Networks**. 由此可以看出, 模型中用到了`图(graph)`相关的表征模型和`co-attention`的机制. 
>
> 笔者的深度学习知识储备浅薄, 看到文章标题的第一反应是, 模型的可解释性(interpretability) 来自`co-attention`机制, 而图表征的技术手段则提供了一个新的表征维度, 原文作者一定是因为其给模型带来的增益较大, 才将其作为文题. 传统而言, 假新闻的表征手段关注于表征文本(源文本, 转发文本, 评论等等), 而本文表征的是`传播路径`. 
>
> 一起来看看原文作者是怎么做的吧.

# 问题建模

> 根据原文`3 Problem Statement`解读

前面这几行没什么好说的, 定义了两个集合, 一个`推文集合`, 一个`用户集合`, 然后用`单词集合`来表示每一篇推文.

![image-20211103145810670](/img/gcan/image-20211103145810670.png)

紧接着, 作者定义了`用户特征`和`传播路径`. 所谓传播路径就是

> a sequence of retweet records.

![image-20211103150120670](/img/gcan/image-20211103150120670.png)

一条传播路径由一个个包含3个元素的tuple组成. 每个tuple里包含了在 $t_j\\$时刻



![image-20211103150619447](/img/gcan/image-20211103150619447.png)

作者将假新闻探测建模成

模型结构如下:

![模型图](/img/gcan/model_arch.png)

