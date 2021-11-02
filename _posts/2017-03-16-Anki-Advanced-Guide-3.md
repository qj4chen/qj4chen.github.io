---
type: post
title: 碎片化英语学习：用Anki把词典搬进你的脑袋
author: Corkine Ma
categories: [Anki操作指南, Project Muninn]
---

> 本文发表在 [简书首页推荐](http://www.jianshu.com/p/07f74ee0f1aa)，是我在之前那篇[高阶应用指南](https://blog.mazhangjing.com/2017/03/06/Anki-Advanced-Guide-2/)的基础上修改而作的**面向Anki生手**的文章。

曾经有一个笑话，是关于一个学习成绩很差的高中学生和他爸的一段对话。他爸问他，英语难学吗？这孩子说，难学；他爸问，你们一共才学3500个单词，每天学十个，一年不就3500了吗？每天十个单词很难吗？这孩子说，不难。那你为什么还是学不好呢？他爸反问。

众所周知，学习并不是单纯的把知识装在脑子里。学习面临着很多技术上的问题，有时候不仅你提取不出来，更有甚者你经常要面对无休无止的遗忘。著名心理学家艾宾浩斯曾经利用节省法，得到了关于无意义音节遗忘随时间变化的曲线，这个实验告诉我们，学习，要不断的复习。

学习并不是一件轻松的事，对于大部分人来说，保证学习的时间尚属困难，更何况还要抽出大量的时间用来复习。鲁迅先生说，时间就像海绵里的水，挤挤，总是会有的。其实我们每天都有大量零散、琐碎的时间，如果我们不能用霍格沃斯的魔法把闲散的时间粘在一起，但我们起码还有另外一条路——把整块的知识打散，一点点的学习。

碎片化学习有两个严重的问题，一是缺乏整体视角，二是我们经常学了忘，忘了学，熟悉的东西学了很多遍，不熟的东西还是不熟，这不仅会造成学习效率的低下，更严重的是它对我们学习热情造成了沉痛的打击。对于英语单词而言，第一个问题不是问题，第二个，在Anki的帮助下也不是问题。具体来说，<u>Anki就是这样一个神奇的软件，它以一种最为经济高效的方法帮助我们排列学习的次序，把你迫切需要学习的陌生的东西放在前面，如果你不熟悉这块知识，不要紧，Anki会一遍一遍的呈现给你，直到你记住为止。</u>Anki是一个开源的软件，它具备全平台同步的能力，也就是说，不管你用的是Android，iOS或者Windows，只要你能连上Web，你就可以随时随地的管理你的Anki笔记，随时随地的学习。

这篇文章，将向我们介绍Anki，以及我们可以用它来实现的一件件看似不可能的事。这叫做Anki的软件，它被设计以用来学习一切知识。Anki就像一把刀，它把知识切碎，然后按照一个设计的极其精妙的机制，让我们闲散时间的学习变成了可能，这种机制被用来对抗遗忘，这样的机制，对于语言的学习来说，效果最好。

> 本文的写作得到了很多朋友的支持，非常感谢他们的贡献和付出。

词典APP，不稀罕，几乎每个人手机里都装的有。但如何把词典的价值最大化，恐怕是很多学习者没有好好考虑过的事。如何把每天学习到的新的生词最大限度的利用，这就需要用到Anki，它将帮助我们碎片化的学习，一点点的把整块的知识打散，精细加工，然后深深地刻在脑子里。

这篇文章，教的，就是让你把每天查词的词典里学到的东西转移到Anki中去，然后让Anki检测你的复习效果，按时提醒你复习。

**具体来说，本文将告诉你：**

- 如何从 PC 浏览器添加生词到Anki
- 如何从有道**词典**导入生词本到Anki
- 如何从欧路**词典**导入生词本到Anki并且使用<u>你喜爱的字典内容</u>作为卡片来源

阅读本文需要一定的Anki操作基础，在此之前，你可能需要了解一下Anki的使用方法，参见这个链接：

**Anki Guide by Corkine**  [博客](https://blog.mazhangjing.com/2017/01/22/Anki-Guide/) | [简书](http://www.jianshu.com/p/fe95b3480e07) 

> Anki Guide by Corkine这篇文章的阅读需要您的PC安装Anki（[PC 版本下载地址](https://apps.ankiweb.net/downloads/current/anki-2.0.44.exe)），按照步骤阅读并亲身实践一遍大概需要**1个小时**，你可以找一个平常用来上网的时间段亲自试试，这对你的学习非常有帮助，如果你在一个星期内搭配Anki学习，那么你将可以用不到7天的时间学习别人半个月才能完成的知识，这样，你就可能有更多的时间用来做其他的事情。

我们知道，可能Anki在很多人眼里是简陋的，麻烦的，不易操作和使用的，看不懂和玩不转的，但是，请相信我们，你所学习的任何关于Anki的操作，都能让你更加自如的使用这个强大的工具。说句大话，只要人类还需要记忆，那么这个软件绝对是一把利器，它被设计用来将任何知识倒进你的脑袋，向来如此。

**关于词典软件的推荐：**

我推荐大家使用欧陆词典，因为这个软件支持`Mdict`词典扩展，所以<u>你可以用它来查各种你喜爱的词典</u>，比如牛津或者朗文（欧路词典内置了一些比较热门的词典，打开即用），欧路词典支持Mac、Windows、iOS和Android，设计明快，没有广告，不卡顿。此外，<u>将文件导入词典中即可实现点词查字典的功能</u>，和Kindle类似。

如果你比较喜欢社交化的词典APP，有道词典也不错；

如果你喜欢浏览网页，那么使用Chrome的一款插件叫做划词助手，它会自动查词，并且在你打开Anki的时候顺便添加生词到你的Anki中，简直没有更方便。

## 第一步：获取单词

### 欧路词典

欧路词典是一个很老的软件了，当年在Android 2.1的年代，就有这种说法：安卓词典用深蓝，苹果词典用欧路。到现在为止，欧路词典已经支持包括Android、iOS、Windows等各个平台，还是一如既往的好用。这个词典最大的优势是——它包含多部词典，或者，你可以说，这是一个<u>词典管理软件</u>。更为难得的是，它没有广告，很轻量，可以选择各种自定义的词典。它很自由，也很强大。

如果你之前一直为手机词典APP的臃肿、自动启动以及卡顿所苦恼，那么你会发现欧陆词典将横扫你对词典APP的恶劣印象，[下载](https://www.eudic.net/eudic/mac_dictionary.aspx)欧陆词典APP，在 `我的` 选项卡下注册或登录你的欧陆词典账户，然后查几个生词，把它们加入到生词本（单词释义的右边，那个小星星即生词本的标志）。

欧路词典生词本导入Anki需要你有一个欧路词典的同步账户，打开 [https://dict.eudic.net/studylist](https://dict.eudic.net/studylist) 这个网页，登录。

![](http://upload-images.jianshu.io/upload_images/4575222-d1c35dba63f7a729.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

选择你想要导出的生词本，在工具选项卡中点选“打印当前列表”。

<u>需要注意的是，**欧路词典的打印当前列表功能默认打印所有的单词**，你不能够选择某个生词本来指定范围。</u>

![](http://upload-images.jianshu.io/upload_images/4575222-2d8632f8fa37e88c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果你使用的是Chrome或者360安全/QQ/UC等浏览器，点击打印按钮旁边的“取消”，然后复制整个网页（按Crtl+A），如图：

![](http://upload-images.jianshu.io/upload_images/4575222-3b3d3d12dcee28c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

打开“记事本”程序，粘贴内容（或按下Ctrl+V），在菜单选择 `文件  > 保存` 命令（或按下Ctrl+S），<u>注意填写文件名以及选择编码为</u>**UTF-8**。

![](http://upload-images.jianshu.io/upload_images/4575222-f35b904a7e68c7cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

打开Anki，选择你一个目标记忆库，点选 `文件 > 导入`（或者按Ctrl+I）。

![](http://upload-images.jianshu.io/upload_images/4575222-1a844aae8b431c19.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)![](http://upload-images.jianshu.io/upload_images/4575222-f878d9647be74223.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<u>导入时，因为我们只需要单词，所以对应文本的第二字段（这个字段在txt文档中是单词，前后都用Tab符隔开），即只需要选择“对应到单词”即可。</u>

注意，这里的笔记类型，我自定义个了一个叫做“欧路词典”的模板，地址：[点此下载](http://info.mazhangjing.com/blog/download/ouludic.apkg)，当然，你也可以自己设计一个模板。

> 附注：有时候导入后，可能有一些问题。在浏览器中选择”今天添加的“选项卡查看。比如有时候记忆库不对，那么在此选项卡下按下Ctrl+A全选笔记，然后选择改变记忆库（箭头所示）即可。

![](http://upload-images.jianshu.io/upload_images/4575222-7626410f6146cc7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此外，还有一个问题，就是这个，记得删掉就好。

![](http://upload-images.jianshu.io/upload_images/4575222-25aae6a31e0c0fbf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 有道词典或金山词霸

我们推荐你注册一个欧路词典账户，按照上述步骤下载欧路词典APP，注册一个账户，然后用**电脑**访问[欧陆词典在线导入](https://dict.eudic.net/studylist/import)进行下一步操作，此网页的FAQ更加详细的教程，可以选择从金山词霸或者有道词典导入生词本。

![](http://upload-images.jianshu.io/upload_images/4575222-801d0e686fb5d6a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当然，我们也可以直接将有道的生词本在线转换成Anki可导入的格式，打开这个网址→[在线格式转换](http://corkine.github.io/yd2anki/) [这里是工具使用说明](http://list.mazhangjing.com/categories/anki/instruction/yd2anki)

### PC端浏览器

从PC端浏览器获取单词，你需要一个Chrome或者类Chrome浏览器，并且安装一个浏览器插件，然后在浏览网页的时候，直接按下Shift，鼠标指向生词，即可出现翻译，点击小星星即可一键添加到Anki中。下面简要的给出了步骤以及一个例子。

#### 概要

1、首先，你需要这个插件 **Anki划词助手**

[谷歌浏览器商店](https://chrome.google.com/webstore/detail/anki-%E5%88%92%E8%AF%8D%E5%88%B6%E5%8D%A1%E5%8A%A9%E6%89%8B/ajencmdaamfnkgilhpgkepfhfgjfplnn) | [本地镜像下载（暂无）](!http://info.mazhangjing.com/blog/download) | [作者主页](http://ninja33.github.io)

2、其次，你还需要Anki端安装一个插件叫做 **ankiconnect** 

[原始下载](https://ninja33.github.io/20160817/anki-connect/ankiconnect.py) | [本地镜像下载](http://info.mazhangjing.com/blog/download/ankiconnect.py) 

3、安装方法：将`ankiconnect.py`文件复制到`/我的文档/Anki/addon`目录下，然后重启Anki。

4、最后，这是作者写的使用说明 → [Chrome插件《Anki 划词制卡助手》使用说明(含视频教程)](https://ninja33.github.io/20160817/anki-dict-helper-chrome-extension/)

**<u>5、必须强调的是，你必须使用保持Anki PC 端打开，才可以制作卡片。</u>**

#### 举例

Anki导入此[模板（点击下载）](http://info.mazhangjing.com/blog/download/chromedic.apkg)，将Chrome的Anki划词助手这样设定：

![](http://upload-images.jianshu.io/upload_images/4575222-7d68e14985803458.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)![](http://upload-images.jianshu.io/upload_images/4575222-3c04294946ea6f94.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的模板使用的区域名称如上图所示，对应Anki划词的右边标签取值即可。<u>打开一个网页，按下Shift，鼠标左键点选，即可出现如下弹框，点击箭头所指的＋即可添加到Anki，添加后会变成灰色，如图所示。</u>（**划词需要开着Anki才可以制作卡片**）

![](http://upload-images.jianshu.io/upload_images/4575222-4acff4c17053beb4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4575222-9ed4b7a5c438dd7c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如上图所示，一般来说，我比较喜欢选择有道的词典，当然，你也看到了，如果使用Anki划词提供的第一个词典有一些无伤大雅的格式问题，你可以调整CSS自行解决。[模板下载地址](http://info.mazhangjing.com/blog/download/chromedic.apkg)

### 从文本文件导入

你也可以从Excel或者手动往Anki中添加单词， Anki导入文件需要用Tab或者空格分开单词，只要是满足这个条件的任何文本文档，都可以将其内容添加到Anki中。

## 第二步：查词典

如果说上一步是我们第一次和生词见面，那么这一步，我们将具体的认识这些单词，通过Anki内置的一个叫做Word Query的小插件，点几下鼠标，我们就可以为多达上千个单词配上释义（你甚至可以自己指定一本词典）、发音，这一步结束之后，我们的单词卡片就制作好了，余下的任务，就是你的了。

### 自动化查词工具：Word Query

如果你按着上述步骤做下来，你可能发现，卡片只有一个单词，发音和释义都没有，这时，我们就需要借助一个叫做 `Word Query` 的插件来解决问题了，插件下载地址：

[官方下载地址](https://ankiweb.net/shared/info/775418273) | [本地镜像下载](http://info.mazhangjing.com/blog/download/wordquery.py)  

插件ID：775418273，在 `工具 > 插件 > 浏览&安装` 目录菜单下，输入ID即可自动下载。

![](http://upload-images.jianshu.io/upload_images/4575222-c12c74c452b2913c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如图，选择你需要补充的单词，点击WordQuery菜单的 `Option` 选项，打开选项窗口。如果你试用了和我一样的模板（欧路词典），那么按上图设置即可，**单选框选中的部分是需要查询单词所在的区域**，<u>选中后此区段就仅仅作为查词的用途，其“字典字段”不起作用</u>。音标我们选择有道的“音标”，释义、例句等下面的类似。如果你使用了我的模板，那么释义可以不选，例句选用有道的柯林斯英汉即可，这个柯林斯包含例句和释义。当然，你也可以用 MDX Server 选择一本自己的词典，具体使用方法见一下两篇文章：

- [Anki 制卡基础服务 MDX Server 使用说明](https://ninja33.github.io/20161113/mdx-server/)


- [AwesomeTTS (MDX Server版)使用说明(含视频教程)](https://ninja33.github.io/20161113/awesometts-mdx-server/)

点击确定即可，等待查词完成，即可开始愉快的复习了。为了大家阅读方便，我在下面一小节简单介绍一下MDX Server的使用方法，使用这个服务，可以用你喜欢的词典内容制作卡片。

### 高级：换一本顺手的词典

首先，你需要有一台PC、Mac或者搭载 Linux 系统的电脑。最好是64位的Windows。

对于不是64位的非Windows系统，安装和使用方法见上面那个[使用说明的链接](https://ninja33.github.io/20161113/mdx-server/)，在下载的文件里，作者提供了百度云盘的词典地址，以及使用 Linux/Mac 所必要的软件说明，比如Python。 

![](http://upload-images.jianshu.io/upload_images/4575222-094cba3454053526.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个时候，我们的例句字段就要换成 MDX Server 的地址了，也就是 `http://localhost:8000` 。点击确定，但不要开始查词，我们还需要配置一下MDX Server。

如下图打开服务以及选择你的词典，我用的是牛津高阶第八版，点击“打开”。等一会，在浏览器输入 `http://localhost:8000/test` 来测试下是否服务正常，<u>这里的test字段即是你要查的词。</u>

![](http://upload-images.jianshu.io/upload_images/4575222-d7b5f8a77a78e80e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4575222-0a9470af82274f1c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

测试成功后，返回Anki，选中你想要查的词，然后菜单 `WordQuery > Query Selected` 即可查词。

理论上来说，你可以用这个插件配合Anki来学习任何来源的单词，所以...好好学习吧。

## 结语

这一篇教程看着似乎很长很麻烦，没错，它的确如此。如果你第一次接触Anki，你甚至必须首先了解Anki的使用方法，我另写了一篇很有用的[帮助教程](https://blog.mazhangjing.com/2017/02/19/Anki-Advanced-Guide/)，你也可以从[这篇教程](https://blog.mazhangjing.com/2017/02/19/Anki-Advanced-Guide/)里找到更多的帮助资源。

保守估计，你完成此篇教程大概需要**1个小时**，加上阅读Anki基础操作的**1个小时**，你或许充满疑惑，这接近1/12天的时间到底对大家来说值不值得，我一直在问自己这个问题。在这个年代，人人渴望着超越自己，有所成就。知乎上到处都是关于如何进步、如何学习、如何成长的问题。在心理学有关问题解决的理论里，有一个叫做**前景理论**或许可以很好的回答你的疑惑。

**前景理论**指出，问题的解决以及你的成就取决于我们看待问题的参照点以及解决问题的框架，一件事情值不值得，要看它在我们心中重不重要（权重），以及其到底有什么用，有没有用，有多大用（效用）。

我不会也没有能力去开一个Live，用两个小时的时间告诉大家致富成功的秘诀，但是，我可以说，**你用两个小时时间完全可以掌握这个叫做Anki的工具的使用方法。**

有人说，找一整块时间，拿着书学习才正经，但是，当你走出学校，被工资、生活步步紧逼的时候，学习需要的成本会高到很难让人接受。如果你不能够选择回到过去，那么起码这个工具给了你一个别的选择。

当你静下来甚至繁忙而枯燥的一天，你会发现，其实我们的闲暇时间，大多数都在等车、等人、吃饭、随便上上网、逛逛微博的时间里悄然而逝。如果你认为这些闲散的时间，我们完全可以做些更有意义的事——比如学习，如果你认为学习能够带来的能力以及思维上的提升，那么我们就说，这件事的效用很高，它在我们心里有很大的权重。

Anki并没有帮助我们变得更好，它只是一个工具，它像一把刀，把知识切割成块，分散到我们日常生活的角落里。我们可以轻松的用1分钟学习一张卡片，这可不是漫威，也不是星球大战，而是你完完全全能够做到的事情，根据前景理论，你应该做这件事。