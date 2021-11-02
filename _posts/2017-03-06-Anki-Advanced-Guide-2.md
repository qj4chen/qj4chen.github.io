---
type: post
title: Anki高阶应用指南（二）：Anki + 词典 打造终极记单词利器
author: Corkine Ma
categories: [Anki操作指南, Project Muninn]
---

> 在之前的博文中，我写过一篇Anki的基础上手指南，介绍了一些基本概念，基本的软件使用方法。工欲善其事，必先利其器，在这篇文章中，我将详细陈述一些Anki的高级使用方法，这些方法之所以高级，是因为开源的Anki可以很好的和其它工具结合起来，帮助我们做的更多，提高我们的记忆效率。

| 相关文章                                     |
| :--------------------------------------- |
| [Anki Guide by Corkine](https://blog.mazhangjing.com/2017/01/22/Anki-Guide/) 本篇讲解基础的Anki概念和操作 |
| [Anki高阶应用指南（一）：Anki + Kindle 打造记单词利器](/2017/02/19/Anki-Advanced-Guide/) |
| [Anki高阶应用指南（二）：Anki + 词典 打造终极记单词利器](/2017/02/19/Anki-Advanced-Guide-2/) |
| 待补充...                                   |

本文的写作得到了很多朋友的支持，非常感谢他们的贡献和付出。

之前写过一篇利用Kindle和Kindle Mate来导出英语单词，然后在Anki中进行复习的方法，得到了大家的喜欢和支持，这一篇文章，我决定在之前的那篇文章上改进一下，扩大在Anki学习和记忆单词来源的广泛性，并且提高制作卡片的效率。

**具体来说，本文将告诉你：**

- 如何从**Chrome**/类Chrome浏览器添加生词到Anki
- 如何从有道**词典**导入生词本到Anki
- 如何从欧路**词典**导入生词本到Anki并且使用<u>你喜爱的字典内容</u>作为卡片来源

**关于词典软件的推荐：**

我推荐大家使用欧陆词典，因为这个软件支持`Mdict`词典扩展，所以<u>你可以用它来查各种你喜爱的词典</u>，比如牛津或者朗文（欧路词典内置了一些比较热门的词典，打开即用），欧路词典支持Mac、Windows、iOS和Android，设计明快，没有广告，不卡顿。此外，<u>将文件导入词典中即可实现点词查字典的功能</u>，和Kindle类似。

如果你比较喜欢社交化的词典APP，有道词典也不错；

如果你喜欢浏览网页，那么使用Chrome的一款插件叫做划词助手，它会自动查词，并且在你打开Anki的时候顺便添加生词到你的Anki中，简直没有更方便。

## 有道词典导入Anki

使用有道词典导入Anki，需要将有道的生词本转换格式，使用这个网址：[http://corkine.github.io/yd2anki/](http://corkine.github.io/yd2anki/)

步骤如下，非常简单：

1. 把有道词典导出的`XX.xml`选择用记事本打开后粘贴到网站右边的红色文本框中。
2. 点击右边的蓝色表格会出现Anki格式的文本。全选后另存为txt文件即可（注意编码为<u>**utf-8**</u>）。
3. 将TXT格式转换后的生词本导入到Anki中，并选择一个笔记类型，对应好项目即可。（*具体可见上一篇的Kindle Mate生成的TXT格式文档转换成Anki卡片的教程，附送还有一个漂亮的模板*）

> 注：本转换网站使用的代码版权归知乎用户[易小猫](https://www.zhihu.com/people/qu-le-82-6)。这里是原文 [→ 知乎专栏](https://zhuanlan.zhihu.com/p/24288262)

另外一种方法是，把生词本文件导入到欧路词典中，访问[欧陆词典导入页面](https://dict.eudic.net/studylist/import)进行下一步操作。当然，你需要一个欧路词典账户，强烈推荐此账户，可同步多平台的生词。此网页的FAQ更加详细的教程，可以从金山词霸或者有道词典导入生词本。

![](/media/import.png)

## 欧路词典导入Anki

欧路词典是一个很老的软件了，当年在Android 2.1的年代，就有这种说法：安卓词典用深蓝，苹果词典用欧路。到现在为止，欧路词典已经扩张到了包括Android的各个平台，还是一如既往的好用。

这个词典最大的优势是——它包含多部词典，或者，你可以说，这是一个词典管理软件。更为难得的是，它没有广告，很轻量，可以选择各种自定义的词典。它很自由，也很强大。

欧路词典生词本导入Anki需要你有一个欧路词典的同步账户，打开 [https://dict.eudic.net/studylist](https://dict.eudic.net/studylist) 这个网页，登录

![](/media/out2.png)

选择你想要导出的生词本，在工具选项卡中点选“打印当前列表”。（<u>需要注意的是，**欧路词典的打印当前列表功能默认打印所有的单词**，你不能够选择某个生词本来指定范围，所以，一般来说，我把它们都导入Kindle，然后用标签来区分不同的生词本</u>）

![](/media/oulu2.png)

如果你使用的是Chrome或者360安全/急速/QQ/UC等浏览器，点击打印按钮旁边的“取消”，然后复制整个网页（按Crtl+A），如图：

![](/media/oulu3.png)

打开“记事本”程序，或者“gedit/Notepad++”等类记事本程序，粘贴内容（Ctrl+V），按Ctrl+S，或者在菜单选择 文件  > 保存命令，注意填写文件名以及选择编码为**UTF-8**（非常重要）。

![](/media/jishiben.png)

打开Anki，选择你一个目标记忆库，点选 `文件 > 导入`（或者直接按下Ctrl+I）。

![](/media/out5.png)![](/media/out7.png)

<u>导入时，因为我们只需要单词，所以对应文本的第二字段（这个字段在txt文档中是单词，前后都用Tab符隔开），即只需要选择“对应到单词”即可。</u>

注意，这里的笔记类型，我自定义个了一个叫做“欧路词典”的模板，地址：[点此下载](http://info.mazhangjing.com/blog/download/ouludic.apkg))，当然，你也可以自己设计一个模板。

> 附注：有时候导入后，可能有一些问题。在浏览器中选择”今天添加的“选项卡查看。比如有时候记忆库不对，那么在此选项卡下按下Ctrl+A全选笔记，然后选择改变记忆库（箭头所示）即可。

![](/media/liulanqi.png)

此外，还有一个问题，就是这个，记得删掉就好。

![](/media/out12.png)

## Chrome浏览器生词导入Anki

### 大致步骤

1、首先，你需要这个插件 **Anki划词助手**

[谷歌浏览器商店](https://chrome.google.com/webstore/detail/anki-%E5%88%92%E8%AF%8D%E5%88%B6%E5%8D%A1%E5%8A%A9%E6%89%8B/ajencmdaamfnkgilhpgkepfhfgjfplnn) | [本地镜像下载（暂无）](http://info.mazhangjing.com/blog/download) | [作者主页](http://ninja33.github.io)

2、其次，你还需要Anki端安装一个插件叫做 **ankiconnect** 

[原始下载](https://ninja33.github.io/20160817/anki-connect/ankiconnect.py) | [本地镜像下载](http://info.mazhangjing.com/blog/download/ankiconnect.py) 

3、安装方法：将`ankiconnect.py`文件复制到`/我的文档/Anki/addon`目录下，然后重启Anki。

4、最后，这是作者写的使用说明 → [Chrome插件《Anki 划词制卡助手》使用说明(含视频教程)](https://ninja33.github.io/20160817/anki-dict-helper-chrome-extension/)

**<u>5、必须强调的是，你必须使用保持Anki PC 端打开，才可以制作卡片。</u>**

### 一个简略的使用说明

使用此[模板（点击下载）](http://info.mazhangjing.com/blog/download/chromedic.apkg)，将Chrome的Anki划词助手这样设定：

![](/media/huaci1.png)![](/media/huaci2.png)

我的模板使用的区域名称如上图所示，对应Anki划词的右边标签取值即可。<u>打开一个网页，按下Shift，鼠标左键点选，即可出现如下弹框，点击箭头所指的＋即可添加到Anki，添加后会变成灰色，如图所示。</u>（**划词许开着Anki才可以制作卡片**）

![](/media/huaci3.png)

![](/media/huaci4.png)

如上图所示，一般来说，我比较喜欢选择有道的词典，当然，你也看到了，如果使用Anki划词提供的第一个词典有一些无伤大雅的格式问题，你可以调整CSS自行解决。模板下载地址：http://info.mazhangjing.com/blog/download/chromedic.apkg



## 从词典导入卡片的善后工作

### 一个好用的插件：Word Query

如果你按着上述步骤做下来，你可能发现，卡片只有一个单词，发音和释义都没有，这时，我们就需要借助一个叫做 `Word Query` 的插件来解决问题了，插件下载地址：

[官方下载地址](https://ankiweb.net/shared/info/775418273) | [本地镜像下载](http://info.mazhangjing.com/blog/download/wordquery.py)  

插件ID：775418273，在 `工具 > 插件 > 浏览&安装` 目录菜单下，输入ID即可自动下载。

![](/media/wq1.png)

如图，选择你需要补充的单词，点击WordQuery菜单的Option选项，打开选项窗口。如果你试用了和我一样的模板（欧路词典），那么按上图设置即可，**单选框选中的部分是需要查询单词所在的区域**，<u>选中后此区段就仅仅作为查词的用途，其“字典字段”不起作用</u>。音标我们选择有道的“音标”，释义、例句等下面的类似。如果你使用了我的模板，那么释义可以不选，例句选用有道的柯林斯英汉即可，这个柯林斯包含例句和释义。当然，你也可以用MDX Server选择一本自己的词典，具体使用方法见一下两篇文章：

- [Anki 制卡基础服务 MDX Server 使用说明](https://ninja33.github.io/20161113/mdx-server/)


- [AwesomeTTS (MDX Server版)使用说明(含视频教程)](https://ninja33.github.io/20161113/awesometts-mdx-server/)

点击确定即可，等待查词完成，即可开始愉快的复习了。为了大家阅读方便，我在下面一小节简单介绍一下MDX Server的使用方法，使用这个服务，可以用你喜欢的词典内容制作卡片。

### 使用一本自己喜爱的词典

首先，你需要有一台PC、Mac或者搭载 Linux 系统的电脑。最好是64位的Windows。

对于不是64位的非Windows系统，安装和使用方法见上面那个[使用说明的链接](https://ninja33.github.io/20161113/mdx-server/)，在下载的文件里，作者提供了百度云盘的词典地址，以及使用 Linux/Mac 所必要的软件说明，比如Python。 

![](/media/out10.png)

这个时候，我们的例句字段就要换成 MDX Server 的地址了，也就是 `http://localhost:8000` 。点击确定，但不要开始查词，我们还需要配置一下MDX Server。

如下图打开服务以及选择你的词典，我用的是牛津高阶第八版，点击“打开”。等一会，在浏览器输入 `http://localhost:8000/test` 来测试下是否服务正常，<u>这里的test字段即是你要查的词。</u>

![](/media/out1.png)

![](/media/out11.png)

测试成功后，返回Anki，选中你想要查的词，然后菜单 `WordQuery > Query Selected` 即可查词。

理论上来说，你可以用这个插件配合Anki来学习任何来源的单词，所以...好好学习吧。