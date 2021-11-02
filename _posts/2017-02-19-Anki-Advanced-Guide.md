---
type: post
title: Anki高阶应用指南
author: Corkine Ma
categories: [Anki操作指南, Project Muninn]
---

> 在之前的博文中，我写过一篇Anki的基础上手指南，介绍了一些基本概念，基本的软件使用方法。工欲善其事，必先利其器，所以，在这篇文章中，我将详细陈述一些Anki的高级使用方法，这些方法之所以高级，是因为开源的Anki可以很好的和其它工具结合起来，帮助我们做的更多，提高我们的记忆效率。

| 相关文章                                     |
| :--------------------------------------- |
| [Anki Guide by Corkine](https://blog.mazhangjing.com/2017/01/22/Anki-Guide/) 本篇讲解基础的Anki概念和操作 |
| 待补充...                                   |

本文的写作得到了很多朋友的支持，非常感谢他们的贡献和付出。

## Anki + Kindle Mate：记单词利器

Kindle用来阅读外文书是最好不过的，遇到生词，直接按下去，字典和释义便会出来，这十分方便。对于英语的入门者而言，如何充分利用在Kindle，让这些外文书的单词得到最牢固的记忆，一直是一个大难题。Kindle提供了一个叫做“生词本”的功能，不过不太好用，这一部分，我们来探究，如何用Anki来帮助我们管理KIndle上的单词。

Kindle的文件目录里，有一个叫做`My Clipping.txt`的文件，这里面保存了我们所有的笔记和生词，通过一个叫做Kindle Mate的软件，我们可以在PC下很方便的查看这些内容，你可以把笔记和标注保存到Onenote或者Evernote中，更棒的是，你也可以把生词导出，保存到Anki中。

### 相关资源：Kindle Mate

[豆瓣小组](https://www.douban.com/group/kindlemate/) | [官方网站和下载地址](http://kmate.me/cn/) | [软件帮助](http://kmate.me/democn/) | [上手指南](http://kmate.me/2016/10/06/kindle-mate-quick-tips-cn/)

### 步骤一： 提取生词

在Windows下，首先，USB连接Kindle，等待Kindle出现驱动器模式，在资源管理器中出现Kindle盘符。

如图操作，确认Kindle剪贴板导入Kindle mate：

![](/media/mate1.png)

按下F12或者点击菜单 编辑 > 管理生词本释义 打开一下窗口：

![](/media/mate2.png)

如果你之前用鼠标点选了一本书（1处所示），那么在3处你可以选择词典的更新范围为此书。

在2处选择一个词典释义来源，看你喜好，4处选择释义的类型，5处可以清楚之前的释义，6处开始，依次点击或选择，生词本释义就会出现在上图右上角的释义部分，如下图所示。

![](/media/mate3.png)

如下图所示，自定义生词本格式，注意选择Anki风格为***已选择字段，无正背面***，这个的自定义程度大，比较方便。勾选你想要的部分，最方便的是按照下图所示勾选，可以直接用Anki的模板，也可以根据自己喜好选择，不过，模板就需要你自己更改下，更改模板需要一定的CSS和HTML基础。

![](/media/mate4.png)

鼠标左键点选你想要导出的书籍，右键选择导出，如图选择保存类型：

![](/media/mate5.png)

使用Notepad++打开这个txt文件，可以看到文件的结构，各部分是用tab制表符隔开的，第一部分是生词，第二部分是其原型，第三部分是例句，第四部分是释义。

![](/media/mate6.png)

在这里选择《...》这个东西，按Ctrl+F替换为![](/media/mate7.png)，这一部不是必选，但可以保持书籍显示美观，Ctrl+S 保存文档即可。

### 步骤二：Anki导入生词

使用一下这个模板，非常方便，此模板根据***KindleMate+Anki之小白记单词v2 (Anki发音升级版) ***这篇文章和 ***The Kite Runner*** 此作者更改，如果需要原始模板，直接点击[这个连接](http://kmate.me/2016/10/10/kindle-mate-anki-card-for-reciter-cn/)查看他的文档。

本文介绍的例子使用的模板：[下载地址](http://info.mazhangjing.com/blog/download/KindleMate.apkg) 请打开资料库删除此记忆库我的实例卡片即可，双击导入，即可查看Kindle Mate记忆库。

在Kindle的这个模板的记忆库下，菜单中选择 文件 > 导入 ：

![](/media/mate8.png)

字段对应上就是，单击导入即可。

### 步骤三：解决发音问题

![](/media/mate9.png)

如上图所示，按Ctrl+A全选你的记忆库笔记，如图添加音频。Awesome TTS的插件参见：

> 安装tts插件：Anki界面最上方的Tools——Add-Ons（插件）——Browse &Install (浏览和安装)——输入301952613，建议自己点一下Browse试一下其他的插件，不需要的卸载也很方便。

![](/media/mate10.png)

如上图所示，按照你的喜好选择，注意右侧发音和单词对应上即可，点击 Generate（创建）继续。

### 结语

最终结果如图所示：

![](/media/mate11.png)

iOS端如图所示：

![](/media/mate12.png)

以上文章内容的编写参考了*The Kite Runner*的[***KindleMate+Anki之小白记单词v2 (Anki发音升级版)***](http://kmate.me/2016/10/10/kindle-mate-anki-card-for-reciter-cn/)，这篇文章发表在KindleMate的博客里，因为iOS用这位作者的模板字体太小，所以我简单的修改了一下CSS样式，如果有什么软件的Bug，可以去这篇文章的帖子下回复，作者应该就是Kindle Mate的作者，非常感谢他的贡献，开发出了这么好用的软件！