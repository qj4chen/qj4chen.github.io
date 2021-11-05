---
layout: post
title: "【论文解读】GCAN: Graph-aware Co-Attention Networks for Explainable Fake News Detection on Social Media"
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

> 以下根据原文`3 Problem Statement`解读

前面这几行没什么好说的, 定义了两个集合, 一个`推文集合`, 一个`用户集合`, 然后用`单词集合`来表示每一篇推文.

<img src="/img/gcan/image-20211103145810670.png" alt="image-20211103145810670" width=400 />


紧接着, 作者定义了`用户特征`和`传播路径`. 所谓传播路径就是

> a sequence of retweet records.

<img src="/img/gcan/image-20211103150120670.png" alt="image-20211103150120670" width=400  />

某条推特 $s_i$ 的一条传播路径 $R_i\\$ 由一个个包含3个元素的tuple组成. 每个tuple里包含了在  $u_j\\$ 这个用户,以及ta的各种属性 $x_j\\$ , 以及ta转发这条推特的时间 $t_j\\$ . 数据集里包含了 $s_i\\$ 的标签 `fake` or `true`, 注意到, 这个标签随着时间从 $t_1\\$ 变到 $t_K$ (传播路径 $R_i\\$ 上一共有 $K\\$ 个用户 ) 是**保持不变的**.

<img src="/img/gcan/image-20211103150619447.png" alt="image-20211103150619447" width=400  />

<span id="用户特征">

此外, 根据后文, 用户 $u_j\\$ 的特征 $x_j\\$ 主要包括:

* number of words in a user’s self-description
* number of words in $u_j\\$ 's screen name
* number of users who follows $u_j\\$
* number of users that $u_j\\$ is following
* number of created stories for $u_j\\$
* time elapsed after $u_j\\$' s first story
* whether the $u_j\\$ account is verified or not
* whether $u_j\\$ allows the geo-spatial positioning
* time difference between the source tweet’s post time and $u_j\\$ ’s retweet time
* the length of retweet path between $u_j\\$ and the source tweet (1 if $u_j\\$ retweets the source tweet)

作者希望通过**传播路径**和**原始推文**来做一个二分类问题, 即原始推文是真还是假. 并且, 作者希望模型能针对**假新闻的样例**给人们启示:

* 找出那部分少量的可疑用户
* 找出假新闻中独特的词语

<img src="/img/gcan/image-20211105103606361.png" alt="image-20211105103606361" width=400 />

# 模型结构

> 总体来说, 下面的图片概括了原文章提出的深度学习模型.

模型主要包括5个部分:

* 用户特征提取(user characteristics extraction)
  * 对每个用户$u_j\\$ , 都能提取出特征 $x_j\\$ , 具体描述见[上文](#用户特征)
* 推文的表征(news story encoding) //如何翻译encoding比较好呢
  * 设定文本的最大长度$m\\$,  zero padding to maximum length
  * 通过全连接层生成 $d\\$ 维的word embedding $V = [v_1, v_2, ..., v_m]\\$
  * 使用GRU来学习$V\\$里的次序特征, $s_t=GRU(v_t), t=1, ..., m$
  * 最终的推文表征为 $S = [s^{1}, s^{2}, ..., s^{m}]\in \mathbb{R}^{d \times m}\\$
* 传播途径的表征 
  * user propagation representation
  
    > 针对传播路径 $R_i\\$ 中用户特征 $[x_1, ..., x_j, ..., x_K]$ 的处理
  
    * GRU-based representation
    * CNN-based representation (1-Dimension)
  
  * graph-aware propagation representation
  
    * 全连接的图; 用户作为节点; 两个用户的相似度作为边, 通过用户的特征向量$x_i\\$, $x_j\\$ 的余弦相似度计算得出.
    * 两个GCN层叠加 
* 耦合的注意力机制(co-attention mechanisms 用于捕捉`推文`和`传播途径`之间的关系)
* 预测(make prediction)

![模型图](/img/gcan/model_arch.png)

# 总结

用机器学习来研究假新闻的传播, 检测, 是近几年比较流行的话题.

# 参考

- Lu, Y.-J., & Li, C.-T. (2020). GCAN: Graph-aware Co-Attention Networks for Explainable Fake News Detection on Social Media. *Proceedings of the 58th Annual Meeting of the Association for Computational Linguistics*, 505–514. https://doi.org/10.18653/v1/2020.acl-main.48

