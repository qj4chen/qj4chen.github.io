---
layout: post
typora-root-url: ..
title: 正则表达式：从投资者问里提取每经AI新闻
categories:
  - 正则表达式
  - 财经文本
keywords:
  - 问答

---


>本文在已有原始数据的基础上, 使用正则表达式`regular expression`, 提取投资者和上市公司高管之间的有趣问答. 
>
>据观察, 这种基础性的自然语言处理技巧被广泛地运用在部分财经媒体的``AI新闻``模块

# 获取数据

[东方财富网](https://www.eastmoney.com/)是一个庞大的财经门户网站, 里面每天都有大量的官方推送和用户生成的内容.

我们的教程也从东方财富网开始.

在[东方财富](https://so.eastmoney.com/web/s?keyword=)的网站上搜索某家股票, 点进结果, 拉到最下, 就能看到如下的`某家股票吧`. 下图展示了"宁德时代"的页面. 可以看到, 在最右边一栏有`问董秘`板块, 点进去之后就是投资者和上市公司之间的各种奇葩对话了. 

![image-20211103175135931](/img/eastmoney/image-20211103175135931.png)

假设我们还没有掌握爬虫技巧, 那么我们想做的事情就是多次点击页面中的`点击加载更多`, 让页面展示了足够多的问答数据之后, 再`右键`-`查看网页源代码`.

![image-20211103175614218](/img/eastmoney/image-20211103175614218.png)

在一个新弹出的页面里面, 我们可以看到一些`html`代码, 我们按`ctrl+A`将它们全部选中, 然后复制, 粘贴到一个记事本文件里, 保存成`data.txt`

有了数据, 接下来就能开始解析了. 

值得一提的是, 如果引入`爬虫`的技术, 其本质上就是把我们人工点击`点击加载更多`按钮的过程交给一个机器人来做, 显然, 这样更省时省力.

# 把文本导入进来

一般而言, 导入一个`.txt`文件, 我们直接使用`python`的`open(文件名).read()`方法就可以了, 也就是`html = open('./data.txt').read()`. 但有的时候, 如果需要读取的文本很大, 一个更安全的做法是使用如下的做法:
```python
with open('./data.txt') as file:
    html = file.read()
```
到这一步, 一个名为`html`的字符串类型的数据, 已经存在于我们的console里了
# 调试正则表达式
> 上帝: 没有人能一步写100%正确的正则表达式

什么是正则表达式呢? 

简单地说,  正则表达式就是想用一种**通用**的方法来表示一批具有某种相同特征的文本. 以我们刚刚获取的html代码为例:

![image-20211103191002465](/img/eastmoney/image-20211103191002465.png)

每一对**宁德时代的股民**与**宁德时代**的对话, 它们周围都有一些相似的html标记. 比如, 如果不算`换行符`和`空格`之类的干扰,  `<div class="qa_question_text">`之后, 接的一般都是**宁德时代的股民**的提问; 而`<div class="qa_answer_text">`之后, 并不直接跟着**宁德时代**的回答, 而是

```html
<div class="icon icon_answer_small"></div>
                                        <a href="/news,300750,1072951789.html" target="_blank">
                                            <div class="qa_answer_text">

                                                <div class="icon_arrow_left"></div>
                                                <p>
```

这样一些乱七八糟的html标记.

我们找到了**宁德时代的股民**与**宁德时代**的对话的"规律"!

对了, 上面那张截图是在这个[网站](https://regexr.com/)截取的. 上帝说"没有人能一步写好100%正确的正则表达式", 但是`https://regexr.com/`给我们提供了一个便捷地调试正则表达式的platform.

# 究竟什么是正则表达式

我们今天要写的正则表达式长这样: `<div class="qa_question_text">([\s\S]*?)<\/div>[\s\S]*?<div class="qa_answer_text">[\s\S]*?<p>([\s\S]*?)<\/p>`

而我们在`https://regexr.com/`上, 在`Expression`一栏里填入这个正则表达式, 把最开始复制的`网页源代码`粘贴进下面的大空白, 然后找到了15个匹配项. 下图里, 蓝色高亮的就是其中一个匹配项.

![image-20211103192311655](/img/eastmoney/image-20211103192311655.png)

我们来拆解这个长长的正则表达式.

`<div class="qa_question_text">`无需多说, 它是我们想要匹配的文本的`开头`, 我们希望能够**精确地找到**`<div class="qa_question_text">`.

同样地, `<\/div>`, `<div class="qa_answer_text">`, `<p>`, `<\/p>`都是我们想要精确匹配的. 我们可以把它们理解成扫描二维码时用来定位的那几个大黑块, 它们并不携带我们想要的信息, 但是我们想要的信息被包裹在其中.

* `\`是为了表示转义字符`/` (`\/`才能在正则表达式里表示`/`, 或许你不理解这一点, 但是, 看看`regexr.com`的侧边栏`cheatsheet`或许你就明白了)

<img src="/img/eastmoney/image-20211103194912573.png" width=400 />

可以看出, 现在, 我们的正则表达式还剩下`([\s\S]*?)`, `[\s\S]*?`, `[\s\S]*?`, `([\s\S]*?)`, 而这四个有两个还是相同的, 就只剩下了`([\s\S]*?)`, `[\s\S]*?`. 而它们俩的区别只是少了一对括号`()`.

那么, `[\s\S]*?`在做什么事呢? 一句话总结, 它能匹配两个精确字符串之间的全部字符. 比如我们截图里这个就在匹配`<div class="qa_question_text">`和`<\/div>`之间的全部字符. 

<img src="/img/eastmoney/image-20211103194218025.png" alt="image-20211103194218025" width=400 />

至于它具体是怎么实现的嘛, 在`https://regexr.com/`上, 把鼠标移动到某个正则表达式代码附近, 网站就会提示你, 用`小灰框`框出可以拆分的最小单元, 然后告诉你这个小单元是在做什么. 

然后, 结合着右边的`Cheatsheet`, 你可以弄懂任何和正则表达式有关的知识!

那么, 你也能发现用`()`把`[\s\S]*?`括起来, 是为了把`[\s\S]*?`匹配到的字符串当作一个group来对待, 方便后续使用. 

显然, 这里, 我们把**股民的问题**和**宁德时代的回答**分别括起来了.

# Python的正则表达式实现

我们首先需要导入一个叫`re`的包, 毫无疑问, 它是`regular expression`的缩写.

`re`为我们提供了一个名为`findall`的函数, suggested by its name, 它能找到所有的符合条件的子串.

一般来说, `re.findall`接受两个参数, 就能够实现功能. 

* `string`: where to look up, here the long raw html
* `pattern`: the string you try and try and find satisfying in https://regexr.com/

```python
import re
question_answer_pairs = re.findall(string=html,
                                   pattern=r'<div class="qa_question_text">([\s\S]*?)<\/div>[\s\S]*?<div class="qa_answer_text">[\s\S]*?<p>([\s\S]*?)<\/p>')
```

到这一步, 你已经基本上把我们想要的信息找到了. 但是用在`Python Console`里输出一下, 你发现, 还有小瑕疵:

![image-20211103200458414](/img/eastmoney/image-20211103200458414.png)

`\n`表示换行符, 也就是我们在word按下`enter`之后屏幕上出现的符号. 

我们想要干净的文本, 所以我们用`str`自带的`strip`方法, 就能很方便地去掉开头和结尾的换行符:

![image-20211103200730500](/img/eastmoney/image-20211103200730500.png)

那么, 我们用`list comprehension`对所有的文本都来一遍:

```python
question_answer_pairs = [(q.strip(), a.strip()) for q, a in question_answer_pairs]
```

到此为止, 我们已经实现了从一段原始的html文件中提取出一对对的股民与上市公司的问答.

# 完整代码
附上本文用到的全部代码：

```python
import re

with open('./data.txt') as file:
    html = file.read()

# method 1: not safe
questions = re.findall(string=html, pattern=r'<div class="qa_question_text">([\s\S]*?)<\/div>')
questions = [ques.strip() for ques in questions]
answers = re.findall(string=html, pattern=r'<div class="qa_answer_text">[\s\S]*?<p>([\s\S]*?)<\/p>')
answers = [ans.strip() for ans in answers]

# method 2: better
question_answer_pairs = re.findall(string=html,
                                   pattern=r'<div class="qa_question_text">([\s\S]*?)<\/div>[\s\S]*?<div class="qa_answer_text">[\s\S]*?<p>([\s\S]*?)<\/p>')
question_answer_pairs = [(q.strip(), a.strip()) for q, a in question_answer_pairs]
```

# 未完待续

那么, 如何用爬虫来自动地获取html文件呢, 怎么样才能不用人工点击那个`点击加载更多`按钮呢? 我们下期见.
