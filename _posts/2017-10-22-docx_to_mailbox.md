---
layout: post
title: 使用Python实现一个提醒写日记并且自动存档日记的Qt GUI程序的简要说明
categories:
  - 效率
  - 软件编程实践
  - Python
  - Qt

---

> 大约是五年前，我开始了密集的日记记录。之前记日记是因为一个软件，叫做Day One，它只能在iOS平台使用，我买了iPhone 4、在之后的时间里陆续为此买过iPhone 5s、iPad mini2、iPad Air，iPhone 6s。这个软件遵循一些我认为可以称之为“禅”的理念，我在移动设备上写日记的历史一直到这家公司停止了经典版DayOne的支持（他们转向开发了一个功能更多的但是也更加平庸的文字记录软件DayOne2，它不是一个合格的日记软件），从那之后我就更多的使用电脑写日记。在大四的下学期，我写了一个程序，用于提醒我每天写日记，现在，借助于Python，我将其改成了一个更加全能的解决方案。


## 一、功能需求

1、我每天基本都会用Microsoft Office 365 Word程序在笔记本电脑上写日记，然后把它保存在一个叫做“工作文件夹”的目录下，这个目录通过坚果云同步。

2、每天晚上我都会打开另一台电脑，打打游戏或者读一些文章之类，我一般在睡觉前打印“工作文件夹”中从另一台电脑同步的日记存档。

3、我认为第三方同步工具（坚果云）是不可靠的，但是有时候希望在线阅读这些日记，所以一般将这些日记作为电子邮件发送到自己的邮箱中。因此我每天都将这些文件夹中的日记打开，手动复制到Outlook程序中，写好主题和文本，然后发送到自己专有的一个自有域名的邮箱账号下，这个邮件会自动通过转发到我的Outlook.com中，标记为重要，并且存档到一个在线文件夹中。这样就实现了坚果云、笔记本、主电脑、Outlook.com、自有域名(mazhangjing.com)邮件服务器(腾讯企业邮箱)的日记文件的存储，保证了其安全性。

<br>

## 二、提醒写日记的.NET程序代码及设计理念

我之前写过一个程序，大概是在17年5月份写的，这个程序用来专门提醒我每天写日记。这个东西很好，因为大部分时间我都会忘记去写日记，然后都是在这个软件的催促下完成这一工作的。这个程序使用VB.NET语言写的，非常简单，通过Windows计划任务，每天晚上22：00调用这个程序，如果当时没有打开电脑，则在打开电脑后立即运行程序。以下是其在VS Studio 2008中的样子。

![img](/media/pvo1.png)

![img](/media/pvo2.png)

这个程序包含两个窗口，第一个窗口的代码非常简单，如下：

![img](/media/form1.png)

大概意思就是，这个窗口（主窗口）打开后，会有两个选项，如果你点了“等下再说”按钮，这个窗口会自动最小化，这种最小化是那种在任务栏和状态栏你压根找不到那种，程序在这个时候会自动计时，40分钟后再次弹出这个窗口提醒，一直到你按下“我写好了”这个按钮为止。每次你按下“等下再说”后，程序的计数器会在后台工作，每当你按下这个按钮一次，系统就会在第二行显示“第n次提醒”字样，其中的n就是你按下这个按钮的次数+1。

如果你按下“我写好了”这个按钮，程序会显示窗口2，代码如下：

![img](/media/form2.png)

这个界面会随机显示一句鸡汤，在右下角有两个按钮，左边的不明显，并且很小，很难用鼠标点到，并且在心理上不会被认为是按钮；右边的很大，很容易点到。这两个按钮的文字含义基本类似，但是稍有差别，按下左边的按钮，系统会自动退出程序，不再提醒。按下右边的按钮，窗口2会自动关闭，窗口1还在，并且自动隐藏，在4分钟后继续提醒你写日记。

这个程序虽然简单，但是很有用。当我每每感到负罪感的按下“我写好了”，就要面对鸡汤的再次冲击，我在这个时候应该想要结束这种内疚的感觉，所以就会快速的选择一个按钮来结束面对这种情况，因此点击“好”的概率会更大，因为毕竟这是一种可以摆脱困境同时不用再之后自责的方法，因为我知道它只是隐藏了，可以在40分钟后再次选择。

虽然大多数情况下，我基本上都没有点击过“好”这个按钮，基本上在第一个窗口的提醒出来后就赶紧写日记，免得它再次弹出来。

总而言之，这种方法大概是有效的。

![](/media/pvo3.png)

![](/media/pvo4.png)

<br>

## 三、计划任务的妙用

每天定时打开不需要让程序驻留在后台占用内存，可以使用Windows 计划任务，新建一个计划任务即可。直接在开始菜单找到“任务计划程序”，打开后右侧“新建任务”即可。

![img](/media/pvo5.png)

需要注意的是，在箭头处的账户需要是你自己的账户，而非SYSTEM，虽然后者拥有更高的权限，但是对于这种自动运行不安全程序的计划来说，系统不会判定此任务有效。勾选“只在用户登录时运行”，可以解决不少问题。

![img](/media/pvo6.png)

在操作选项卡下，新建一个操作，程序文件框里写上程序所在目录，比如C:/XXX/XX/XXX.exe

需要注意，你需要对这个文件夹有权限，如果没有，更改文件夹读写权限或者换到非系统核心文件夹下。“起始于“输入框输入文件所在的绝对目录，对于bat文件需要更改CMD工作目录的操作，这一步非常重要。

在条件选项卡下，设置触发条件，在设置选项卡下勾选“如果错过计划时间，立即开始；如果任务运行超过以下时间，强制停止”

![img](/media/pvo7.png)

在触发器选项卡中设置程序的触发时间，这里选择每天22：00。

那么，需要注意的是，最好在操作选项卡中的程序输入框中输入的是一个bat批处理而不是exe程序，比如我的是：

`C:\Users\Administrator\计划任务\dailynotice.bat`

让系统调用这个bat批处理来运行bat内的命令。在bat文件内，写下exe程序的地址即可：

`C:\Users\Administrator\setup.exe`

注意这里的地址也需要计划执行者有运行的权限。

一切搞定后，试着运行下计划，看看哪里有问题，查查百度解决这些问题。

 <br>

## 四、使用Python把程序联系起来

这个程序还不是很好，比如每天如果我写过日记，当22：00的时候，这个程序还是会运行，我还要在这个程序内再点两下才行。为了解决这种情况，我使用Python来解决这个问题。并且，不论我是否写完了日记，我都需要打开Outlook，把Word中的文字分别复制到Outlook中并且发送出去，这实在是太繁琐了，因此也考虑使用Python来解决这个问题。

大体思路是，通过os模块来遍历我的工作文件夹，然后使用re模块来判断我的文件夹中哪些文件是日记文件，找到后把这些文件和使用shelve模块存储的包含日记文件文件名和扫描日期的字典中的数据进行比较，如果日记文件名没有出现在字典中则进行下一步，如果这些日记文件名都是在这个字典中出现的，则通过os模块函数调用打开日记提醒程序，提醒我写日记。如果在工作文件夹出现了新的日记文件，那么使用python-docx模块（第三方）从这个docx文档中提取元数据以及文本内容，将标题和内容分开，通过python的win32模块（第三方）打开outlook程序，将标题作为主题，将内容作为正文发送到指定收件箱中，如果这些步骤成功，则通过tk模块弹出对话框提示“发送完成”，否则，如果模块找不到，在stdout中显示相关错误，如果outlook发送出错，则弹出对话框提示相关错误。被发送的邮件会自动转发到我的outlook邮件中，然后设置规则判断邮件主题，自动存档到某一特定文件夹中，并且标记为已读和重要。对程序而言，如果发送成功，则自动更新我的那个包含文件名和更新日期的字典。

所需要的程序如下：

> Python 3.XX with pip 
>
> Python-docx module （use pipinstall python-docx安装）
>
> PypiWin32 module（use python-m pip install pypiwin32安装）
>
> Microsoft Office 365 Outlook（好像2007版之后的Outlook都可以）
>

Python程序如下：

```python
#!/usr/bin/env python3
import sys,os,io
try:
    from docx import *
except:
    print("You need install Pydocx by run 'pip install python-docx' in Bash/CMD if you have python3 and pip installed")
try:
    from tkinter import Tk
    from time import sleep
    from tkinter.messagebox import showwarning
    import win32com.client as win32
except Exception as err_:
    print('Error:',err_)
    print("Maybe you do not have PyPiWin32 components installed, please use 'python -m pip install pypiwin32' to install and try again.")

#读取文件列表，更新文件信息到词典，保存词典，从词典中读取已有信息，交叉对比文件列表更新，如果存在更新，更新词典并保存，并且触发发送到Outlook这一任务
import os,re,time,shelve
notelist=os.listdir('C:/工作文件夹')
reobj=re.compile(r"\b201\d年\d+月.*?日.*?.doc.*?",re.S)
notedict={}
db_old=shelve.open('C:/工作文件夹/daily_update_data/update_data')
for key in db_old:
    notedict[key]=db_old[key]
print('\n这是之前词典中保存的数据======================>','\n',notedict)


notefile_list=[]


for x in notelist[:]:
    if reobj.search(x):
        if x not in notedict:
            the_time=str(time.localtime()[0])+'/'+str(time.localtime()[1])+'/'+str(time.localtime()[2])+' '+str(time.localtime()[3])+":"+str(time.localtime()[4])
            notedict[x]='Update at '+ the_time
            notefile_list.append(str(x))
            # notefile_real=str(x)
        else:
            pass
    else:
        pass
print('\n这是准备写入词典中的数据======================>','\n',notedict)



if notefile_list:
    for notefile_real in notefile_list:
        print("[%s] 正在处理中====================================\n"%notefile_real)
        send_successful=False
        notefile=str("C:/工作文件夹/"+notefile_real)
        try:
            document=Document(notefile)
            text_article = [ paragraph.text for paragraph in document.paragraphs] # .encode('gb2312')
        except Exception as _err:
            print("检测到的文件读取出错，因此不进行更新，已跳过此文件。\n详细信息:%s"%_err)
            continue
        mail_text_body=''
        mail_subject=str(text_article[:1])[2:-2]
        print(mail_subject)
        for pargh in text_article[1:]:
            mail_text_body+=pargh+'\n'


        warn=lambda app:showwarning(app,notefile+'\n'+"邮件发送成功, 退出程序?")
        def outlook():
            app='Outlook'
            olook=win32.gencache.EnsureDispatch('%s.Application'%app)
            mail=olook.CreateItem(win32.constants.olMailItem)
            recip=mail.Recipients.Add("dayone@mazhangjing.com")
            subj=mail.Subject=mail_subject
            mail.Body=mail_text_body
            mail.Send()
            warn(app)
            olook.Quit()

        cpobject=document.core_properties
        def runprop(attr,cpobject):
            try:
                val_=getattr(cpobject,attr)
                if val_ !='':print(attr,'==>',val_)
            except Exception as err_:
                print("Error:",err_)
        for attr in ['author','language','revision','version','last_printed','modified','keywords','comments','category']:
            runprop(attr,cpobject) 

            
        Tk().withdraw()
        try:
            outlook()
            send_successful=True
            
            if send_successful==True: #如果发送成功再写入文件到数据库，否则不写入
                db=shelve.open('C:/工作文件夹/daily_update_data/update_data')
                for key in notedict.keys():
                    db[key]=notedict[key]
                db.close()
                print("\n数据库更新成功\n")
            else:
                pass
        except Exception as err_:
            if send_successful==True:
                warn3=showwarning('提示','发送成功，但Outlook程序配置错误。请允许程序调用Outlook发送邮件。\n错误详情：%s'%err_)
                warn3
            else:
                warn3=showwarning('提示','发送失败。Outlook程序配置错误，请允许程序调用Outlook发送邮件。\n错误详情：%s'%err_)
                print("\n由于Outlook配置错误，因此邮件没有发送，数据库没有更新\n")
                warn3

else:
    os.startfile(r"C:\Users\Administrator\Documents\Visual Studio 2008\Projects\EveryDayNotice\EveryDayNotice\bin\Release\EveryDayNotice.exe")
```

最后，将这个程序添加到Windows 计划任务中去。当然，由于Windows 10的BUG，对，微软官方说是BUG，所以需要新建一个bat文件，然后写入以下代码：

```bash
@echo off

cd C:\Users\Administrator\计划任务

start pythonw auto_send_docx_to_mailaddress.pyw

exit
```

这样调用C:\Users\Administrator\计划任务\ auto_send_docx_to_mailaddress.pyw 文件。然后在计划任务中，定时打开这个bat文件即可，当这个bat文件运行的时候，会自动调用这个pythonw程序。

以上的Python代码实践基本上连OOP面向对象编程的知识都不需要知道，以上程序只是Python强大胶水特性的一个表现，本质上这只是一个单文件的简单脚本，我们可以看到系统编程函数的一些简单调用和文件的持久化存储、tk这一官方GUI模块以及Outlook，DOCX文件包是如何被联系在一起的（通过两个非常有用的第三方包）。
<br>
## 五、一个月后的程序2.0版本

### （1） 修改缘起


起源是这样的，在某天我清理磁盘空间的时候，不小心删除了VS 2008，因此那些早期创建的程序都没有了——其实也就一个程序，就是那个提醒我写日记的程序，开始我觉得还行，没有就没有吧，后来，我发现这样很不好，因为我没有这软件的提醒，自己也逐渐不写日记了，这造成了一连串的后果，其中最严重的就是——我的精神状态开始变坏，更多的体验到焦虑和无聊。

因此，在一个月之后，我决定重写当时那个VB程序的代码。

这事是从昨天开始干的，本来十一点就要睡觉了，突然想要写，那么就立刻动工，关掉了《西部世界》的浏览器窗口，在这天晚上睡觉之前，我完成了整个VB.Net程序的复刻，总体来看，实现的还算不错。

大概是这个样子：

![img](/media/rm1.png)

根据Python语言和Qt的特点，我特意的省略了之前的那个“我写完了”的按钮，然后用一个上下文菜单代替，这样的话，程序看上去更加整齐。此外，我也提供了第二个选项，叫做：“明天再说”，因为有时候真的很难去在一天内完成日记，这样的话，这个按钮就很方便了。那么，程序如何知道昨天的我没有写日记呢？我使用了注册表，也就是通过QSettings进行调用，保存我从打开程序到关闭程序所用的时间、跳过的次数以及最终的状态：写了还是没有写。注册表大概看上去是这个样子：

![img](/media/rm2.png)

同样的，根据选择选项的不同，我最后返回的提示语也是不同的，如果跳过没写，那么关闭程序的提示语是“好好休息”，然后第二天程序会显示大大的“昨天没有写日记，今天呢？”的字在主窗口。如果写了，那么就返回一句我最喜欢的诗歌作为结束。那个红色的“1”使用了CSS技术，这是我最近才学习的知识。

总的来说，这个程序还算不错，实现了之前那个VB.NET程序的东西，并且利用了Qt的特性，加了一些新的东西：比如上下文菜单以及CSS。总而言之，写这个程序对我来说感觉很奇妙，我感到使用Python编程并不像魔法一样，该写的还是要写，需要实现什么功能，就要去做什么事情，并没有比VB.NET简单到哪里去，当然，相比较半年前我随便画画写的那个程序，这个Python文件在一个文件里包含了素有的代码，简单、纯净、优雅，不过，出乎我的意料，效率并没有我想象中那么高。看来，编程没有银弹这个预言确实很牢固，难于打破。

大概代码如下：

```python
#!/usr/bin/env python3

import sys,time
import PyQt5
from PyQt5.QtGui import *
from PyQt5.QtCore import *
from PyQt5.QtWidgets import *
import UI_noticedlg

__VERSION__ = '0.0.1'

StyleSheet="""
QLabel#label_num{
    color:red;
}
"""
class Form(QDialog,UI_noticedlg.Ui_Dialog):
    def __init__(self,parent=None):
        super(Form,self).__init__(parent)
        self.setupUi(self)
        self.starttime=time.ctime()
        self.label_num.setText(" 1")
        self.setStyleSheet(StyleSheet)
        self.culc = 1
        self.timer = QTimer()
        self.finalstate = 0
        self.time = ''
        for ele in time.localtime()[0:3]:
            self.time += str(ele) 
        settings =QSettings()
        self.re_lastinfo = settings.value("MainWindow/LastDay")
        if self.re_lastinfo != None:
            if str(self.re_lastinfo.split(",")[2]) == '1':
                pass
            elif str(self.re_lastinfo.split(",")[2]) == '2':
                self.label_Info.setText("昨天没有写日记，那么今天呢？")

        self.infomation = "确认离开程序？"
        self.pushButton.clicked.connect(self.startNow)
        self.timer.timeout.connect(self.test)
    def startNow(self):
        self.hide()
        self.timer.start(2400000)
        self.culc += 1
        self.label_num.setText(str(self.culc) if len(str(self.culc)) > 1 else ' '+str(self.culc))
    
    def test(self):
        self.show()

    def finishLoop(self):
        self.finalstate = 1
        self.infomation = "咆哮吧，咆哮，怒斥那光的退缩"
        self.close()

    def unfinishLoop(self):
        self.finalstate = 2
        self.infomation = "好好休息"
        self.close()

    def contextMenuEvent(self,event):
        menu = QMenu()
        finishedAction = menu.addAction("写完了")
        finishedAction.triggered.connect(self.finishLoop)
        if str(self.re_lastinfo.split(",")[2]) != '2':
            unfinishedAction = menu.addAction("今天太累，明天补上")
            unfinishedAction.triggered.connect(self.unfinishLoop)   
        menu.exec_(event.globalPos())

    def closeEvent(self,event):
        if QMessageBox.information(self,"提示",self.infomation,QMessageBox.Ok|QMessageBox.Cancel) == QMessageBox.Cancel:
            event.ignore()

        settings = QSettings()
        settings.setValue("MainWindow/LastDay",QVariant(self.writeInfo()))
        settings.setValue('Data/'+self.time,QVariant(self.writeInfo()))

    def writeInfo(self):
        info = "%s,%s,%s,%s"%(str(time.ctime()),str(self.culc),str(self.finalstate),str(self.starttime))
        return str(info)


if __name__=="__main__":
    app = QApplication(sys.argv)
    app.setApplicationName("Daily Notice")
    app.setOrganizationName("Marvin Studio")
    app.setOrganizationDomain("http://www.marvinstudio.cn")
    form = Form()
    form.show()
    app.exec_()
```

### （2） 设计理念

不过，这当然不是结束，我在第二天早上起床后，觉得昨天那个程序不完美，因为在某些特殊情况下，我会更改文件夹的位置，或者变更日记的命名方法，这样的话，我的程序就缺乏灵活性。比如，最近一次，我的工作文件夹迁移了盘。而我的这个查找日记文件的程序依赖于一个特定的目录来进行查找，并且还需要一个固定的地方放置我的数据库信息，我非常痛苦的修改自己写的一团糟的代码，然后在一百多行中找出我设置文件夹和数据库的地方，并且逐个更改其位置到新的地方。这经历很不好，所以我决定做一些改变。

作为一个Python使用者，我所见到的大多数的Python脚本都不是程序员的作品，它们充斥着if、for、while的嵌套循环，难得出现一个函数，逻辑混乱，虽然有很多注释，但是依旧少见高级语句的使用，缺乏清晰的程序设计理念。我认为这种脚本就是为了一次性的完成一份工作，因此——有很多人认为——Python，其主要用途就是作为这样一种用来快速上手应付了事的东西——被低劣的称之为“脚本”，但这显然是一种错误的偏见。他们说，因为Python容易上手，所以任何人只要稍微努力一下，都可以写出来这种东西，用来充当重复工作的很好替代。至于那些需要长久使用的程序，他们会认为使用更加高效的语言去写比较好，这样的观点就给Python彻底划清了界限，把它归为和R、Matlab脚本一样的使用场景受限的工具中去了。实际上，我写的第一个程序，也就是本文之前的那个，本质上来说也是结构混乱缺乏内在逻辑的，但是不可否认的，这样做的原因是因为这样硬编码——把地址硬编码到文本中实在是一种“方便”的做法，对于初学者来言，或者那些有很多工作，学习Python只为了完成某项任务的人而言，这是完全正确并且唯一有效的方法。对这样的群体，代码优雅和复用性是不在考虑范围之内的。

我在更改之前代码的过程中，确实花费了不少的力气，这种力气——用于大量的调试——它带给我的内在兴趣其实不如从一开始新写一段凑合能用的代码，并且如果需要使用新的东西，比如带有参数以及参数默认值的函数，自己设计类、设计一个子类化的类，返回各种的值，对值的类型进行判断，迭代器、装饰器等等，还有那些往往被轻视的代码健壮性问题——try和raise Error的使用。这还只是一个py文件的困难，更不用说OOP在多个不同模块的调用和继承，如果需要扩展一下功能，势必要设计到stdout和stderr，要读写文件，读取和写入数据，编码就成了一个问题。而把这一切解决之后，我们面对的还是一个黑乎乎的命令行窗口，如果需要设计一个GUI程序，tkinter太丑，可用的资源也不多，wxPython上手缺乏资料，demo还不错，那么只剩下PyQt了，等你好不容易在Qt.io注册完账户，下载了4个GB的Qt安装程序并且花了1个小时进行安装以获得Qt帮助文档和设计器，你还需要面对对话框模态和非模态的问题，信号和槽的传递，了解主窗口的特性以及如何实例的去子类化一个部件，PyQt拥有上千个函数，幸好它的文档写的足够优美。但是，如何调用函数打开对话框，获取参数，如何使用注册表保存信息，如何使用traceback捕获异常，以及在程序实现的过程中漫长的调试，插入print语句，判断出错位置，而最终你会发现，自己有可能只是少写了一个冒号或者使用了中文的符号，甚至，四个空格和一个Tab混用之类。

因此，我们看到了丑陋的缺乏结构的代码，一个人写好后发了篇文章然后把程序给第二个人用，第二个人费了千辛万苦看懂代码，然后再继续循环下去。我觉得“简单”似乎成了Python的原罪，导致了低质量的代码成吨的出现。我在《西部世界》中，对于公司CEO非常的钦佩，我认为，细节，是非常重要的，缺乏细节的东西缺乏生命，一个优秀的创作者可以让自己的作品中藏有自己部分的人格，如果粗制滥造，这样的东西能不能称之为作品我都不清楚。诚然，从大部分人的思维里，合乎理性的选择是快速解决问题，然后走人拿钱就好。但是这种极端冷酷无情的做法我实在难以忍受，诚然，由于在这样的大环境下，我也不得不面临着一些外在的压力，因此，我的作品也不会十分完美，但是，正如某哲人说过，正是因为细节，它唤起了我们使用产品的情绪，我认为细节就像褶皱，它可以存储我们的情绪，在我们下一次拿起来相同的东西的的时候，我们会体会到这种因为细节而带来的回忆的重现，它能调动我们的情绪，带来一种人性化的感觉，而这种感觉，支撑着我去学习这些技术，哪怕再枯燥，正是这种对于现状的不满意，对于冷漠的恐惧，它吸引着我去做那些事情。

因此，虽然它带来更少的成就感，我依然实现了它，花了快一天的时间，将一个脚本结构化，然后使用两个GUI程序来接受用户的参数和显示提示信息。

### （3） 流程

絮絮叨叨的说了那么多...收回来，因此，我的第一个想法就是，把之前写的那个读取docx文件并且发送邮件的程序给剖开了，分成两个子函数，传递一些参数进行实现（这个在第一天的晚上我就完成了）。我那之前那个_auto_send_to_mailbox.py_的程序下手，花了一小段时间就改好了。然后是参数，参数应该来自于一个文件，让程序自己读取参数，然后去遍历文件并且发送邮件，这样的话就需要一个GUI程序来设置参数，我又花了一点时间设计了一个对话框，并且让其可以接受几个参数并且保存成文本文件，生成一个daily.setting文件作为之后读取数据、发送邮件的主程序的调用之用。

我的成果，也就是上午实现的程序如图所示：

![img](/media/rm3.png)

它包含三个exe程序，分别是：一个用来设置参数，把用户输入转换成文本保存在daily.setting文件中的向导程序；第二个是导入了checkandsend.py函数的可以检查文件并且和数据库中信息比较以及发送邮件的监测程序，其供系统定时调用，它会从daily.setting中读取参数，比如邮件，需要监察的文件夹和数据库位置之类信息，然后传递给已经结构化的参数，开始执行查找和邮件发送，并且将执行结果返回，使用GUI展示给用户。如果没有新的日记，那么它就唤醒第三个程序，也就是我第一天使用Pyhton重写的那个提醒程序。其实到这里已经结束了，如果我想要更改设置，那么只要点击向导程序设置参数，系统计划任务和最后一个程序绑定定时遍历文件夹并且调用结构化的函数发送邮件，如果没有日记，那么继续调用提醒程序提醒写日记。这个提醒程序和之前那个VB.NET实现的一样，使用了一些心理的小技巧，正是这个程序我十分需要，因此在第一天我就开发了它。第二天我做的工作，其实第一天已经可以完全正常运行了，但是，我依旧花了一天优化代码，提高复用性，这个过程中我收获了很多，当然，对于提高效率，它可能没什么帮助。

我对它唯一的不满意就是——它太松散了，有三个程序，这样很不好，我希望用户只用管理一个程序，因此，我决定只保留那个提醒的程序，但是，问题来了，如何在调用提醒程序前让这个程序先判断是否存在新文件，这个其实也简单，只要把那个监测程序写到if __name__=="__main" 这句话下面，在对话框实例化之后，show()函数调用之前进行判断就好。而对于那个向导程序，这个要实现就比较巧妙了，我们知道，我把参数保存在了daily.setting中，如果找不到这个文件，在提醒程序运行之后，在主窗口展示之前，我要先把向导弹出来，要求用户必须设置一个daily.setting，然后才能继续运行。而如果我们在运行程序中想要更改参数，我在上下文菜单中放了一个Action，点击后可以调出模态对话框，这个问题也就圆满解决了。最后如果我们在读取新文件或者调用outlook出错了要怎么办？我使用了两种做法，第一种是在判断文件，如果没有新文件后直接调用提醒，如果有新文件，那么不调用提醒，直接发送邮件，不论发送成功或者失败，都把结果显示出来，作为一个对话框，这样的话，当有新文件的时候，系统调用这个程序的时候，也算是和用户有一个交互。第二个做法就是拦截所有的print，然后把stdout导到一个log文件中，作为运行日志。

最后，我把这个提醒事项程序打包之后，只需要调用这一个程序就好了，daily.setting在第一次调用的时候就可以生成，日志文件也很方便，而判断新日记的对话框也很好，直到这里，我才算真正的改造完了自己的应用程序，一个单一的二进制可执行文件，可访问注册表，带有一个模态对话框，可以生成参数并且自动根据参数进行搜索，执行发送邮件操作并且返回一个结果对话框，或者，在没有新日记的时候直接弹出提醒窗口提醒写日记，并且，如果你昨天没有写的话，今天必须写。

![img](/media/rm4.png)

下面是提醒事项主程序（不包含UI代码）

```python
#!/usr/bin/env python3

import sys,time
import PyQt5
from PyQt5.QtGui import *
from PyQt5.QtCore import *
from PyQt5.QtWidgets import *
import UI_noticedlg
import appsetting

__VERSION__ = '0.2.0'

StyleSheet="""
QLabel#label_num{
    color:red;
}
"""
class Form(QDialog,UI_noticedlg.Ui_Dialog):
    def __init__(self,parent=None):
        super(Form,self).__init__(parent)
        self.setupUi(self)
        self.starttime=time.ctime()
        self.label_num.setText(" 1")
        self.setStyleSheet(StyleSheet)
        self.culc = 1
        self.timer = QTimer()
        self.finalstate = 0
        self.time = ''
        for ele in time.localtime()[0:3]:
            self.time += str(ele)
        
        
        settings =QSettings()
        self.re_lastinfo = settings.value("MainWindow/LastDay")
        if self.re_lastinfo != None:
            if str(self.re_lastinfo.split(",")[2]) == '1':
                pass
            elif str(self.re_lastinfo.split(",")[2]) == '2':
                self.label_Info.setText("昨天没有写日记，那么今天呢？")

        self.infomation = "确认离开程序？"
        self.pushButton.clicked.connect(self.startNow)
        self.timer.timeout.connect(self.test)
    def startNow(self):
        self.hide()
        self.timer.start(2400000)
        self.culc += 1
        self.label_num.setText(str(self.culc) if len(str(self.culc)) > 1 else ' '+str(self.culc))
    
    def test(self):
        self.show()

    def finishLoop(self):
        self.finalstate = 1
        self.infomation = "咆哮吧，咆哮，怒斥那光的退缩"
        self.close()


    def unfinishLoop(self):
        self.finalstate = 2
        self.infomation = "好好休息"
        self.close()

    def contextMenuEvent(self,event):
        menu = QMenu()
        finishedAction = menu.addAction("写完了")
        finishedAction.triggered.connect(self.finishLoop)
        if str(self.re_lastinfo.split(",")[2]) != '2':
            unfinishedAction = menu.addAction("今天太累，明天补上")
            unfinishedAction.triggered.connect(self.unfinishLoop)
        settingAction = menu.addAction("配置程序")
        settingAction.triggered.connect(self.showSettingDlg)   
        menu.exec_(event.globalPos())

    def closeEvent(self,event):
        if QMessageBox.information(self,"提示",self.infomation,QMessageBox.Ok|QMessageBox.Cancel) == QMessageBox.Cancel:
            event.ignore()

        settings = QSettings()
        settings.setValue("MainWindow/LastDay",QVariant(self.writeInfo()))
        settings.setValue('Data/'+self.time,QVariant(self.writeInfo()))

    def writeInfo(self):
        info = "%s,%s,%s,%s"%(str(time.ctime()),str(self.culc),str(self.finalstate),str(self.starttime))
        return str(info)

    def showSettingDlg(self):
        settingdlg = appsetting.Form(self)
        if settingdlg.exec_():
            pass

if __name__=="__main__":
    app = QApplication(sys.argv)
    app.setApplicationName("Daily Notice")
    app.setOrganizationName("Marvin Studio")
    app.setOrganizationDomain("http://www.marvinstudio.cn")
    form = Form()
    from checkandsend import *
    import os,sys,time
    # os.chdir("C:/Users/Administrator/Desktop")
    from tkinter import Tk
    from tkinter.messagebox import showwarning

    try:
        tmp = sys.stdout
        sys.stdout = open('daily.log','a')
        print('\n\n','='*100)
        print('=============================================',time.ctime(),'======================================')
        print('='*100,'\n\n')
        loadfile = open('daily.setting','r')
        thefile = loadfile.read()
        address=str(thefile.split(",")[0])
        dbaddress=str(thefile.split(",")[1])
        # alertaddress = str(thefile.split(",")[4])
        emailaddress=str(thefile.split(",")[2])
        regular=str(thefile.split(",")[3])


        result,infomation,list,notedict= checkDaily(address=address+'/',
                regular=regular,dbaddress=dbaddress)
        if result == True:
            if list != []:
                print('需要写入的数据',list)
                result_2,result_num,result_txt,processinfo,errinfo= sendMail(list,address=address+'/',emailaddress=emailaddress,
                            dbaddress=dbaddress,preparenotedict=notedict)
                print(result_2,'\n',processinfo,'\n',errinfo,'\n',result_txt)
                Tk().withdraw()
                warn_info=showwarning('提示','%s\n%s\n%s'%(processinfo,errinfo,result_txt))
                warn_info
            else:
                print("成功检索数据，但未发现新数据")
                print(infomation,list)
                sys.stdout.close()
                try:
                    form.show()
                    app.exec_()
                except:
                    pass
    except:
        traceback.print_exc()
        print("读取设置出错,请打开程序后右键选择“配置程序”")
        
        settingdlg = appsetting.Form()
        if settingdlg.exec_():
            form.show()
        else:
            form.show()
        app.exec_()

    


```

下面是向导对话框主程序（不包含UI代码）：

```python
#!/usr/bin/env python3

import sys,os,io,shelve,traceback
import PyQt5
from PyQt5.QtCore import *
from PyQt5.QtGui import *
from PyQt5.QtWidgets import *
import ui_setting

class Form(QDialog,ui_setting.Ui_Dialog):
    def __init__(self,parent=None):
        super(Form,self).__init__(parent)
        self.setupUi(self)
        self.pushButton.clicked.connect(self.selectDb)
        self.pushButton_2.clicked.connect(self.selectCWD)
        self.address=''
        self.dbaddress =''
        self.buttonBox.accepted.connect(self.saveIt)
        
        try:
            loadfile = open('daily.setting','r')
            thefile = loadfile.read()
            self.address=str(thefile.split(",")[0])
            self.dbaddress=str(thefile.split(",")[1])
            self.label_3.setText(self.dbaddress)
            self.label_4.setText(self.address)
            self.lineEdit.setText(str(thefile.split(",")[2]))
            self.lineEdit_2.setText(str(thefile.split(",")[3]))

        except:
            QMessageBox.warning(self,"WARN",'从之前的文件中读取出错，如果你第一次使用此程序，请忽略此条消息')


    def selectCWD(self):
        address=QFileDialog.getExistingDirectory(self,"选择需要监视的文件夹",os.getcwd(),QFileDialog.ShowDirsOnly)
        if address != None:
            self.address = address
            self.label_4.setText(self.address)
        else:
            self.label_4.setText('未选择')

    def selectDb(self):
        choose = QMessageBox.information(self,'选项',"你是否需要新建一个数据库文件？如果没有，请点击'OK',否则点击'Cancel'选择你的数据库问卷",QMessageBox.Ok|QMessageBox.Cancel)
        if choose == QMessageBox.Ok:
            address=QFileDialog.getExistingDirectory(self,"选择需要监视的文件夹",os.getcwd(),QFileDialog.ShowDirsOnly)
            db=shelve.open(address+'/mydailydata')
            db['1999年1月1日.docx']='Update at NOTIME'
            self.dbaddress = address+'/mydailydata'
            self.label_3.setText(self.dbaddress)
        else:
            filename,type = QFileDialog.getOpenFileName(self,"选择你的数据库文件",'',"cmData files (*.dat)")

            if filename != None:
                if '.bak' in filename[-4:] or '.dat' in filename[-4:] or '.dir' in filename[-4:]:
                    filename = filename[:-4]
                    self.dbaddress = filename
                    self.label_3.setText(self.dbaddress)

                else:
                    self.label_3.setText('未选择')
                    QMessageBox.warning(self,"WARN",'无效文件，请重新选取')


            
    def saveIt(self):
        emailaddress = str(self.lineEdit.text())
        regularexp = str(self.lineEdit_2.text())
        if emailaddress == '' or regularexp == '' or self.dbaddress =='' or self.address == '' :#不对提醒程序判断
            QMessageBox.warning(self,"WARN",'输入数据无效，请检查后再试')
        else:
            try:

                savedata = open('daily.setting','w')
                savedata.write('%s,%s,%s,%s'%(self.address,self.dbaddress,emailaddress,regularexp))
                savedata.close()
                QMessageBox.information(self,"Info",'设置数据保存在daily.setting文件中')

            except Exception as _err:
                print(traceback.format_exc())
                QMessageBox.warning(self,"WARN",'数据保存失败')

        
    def runTest(self):
        pass        



if __name__=="__main__":
    app = QApplication(sys.argv)
    app.setApplicationName("Daily Notice")
    app.setOrganizationName("Marvin Studio")
    app.setOrganizationDomain("http://www.marvinstudio.cn")
    form = Form()
    form.show()
    app.exec_()
```

下面是两个在主程序中调用的两个函数的模块文件。

```python
#!/usr/bin/env python3
import sys,os,traceback,shelve
try:
    from docx import *
except:
    raise ImportError("You need install Pydocx by run 'pip install python-docx' \
                        in Bash/CMD if you have python3 and pip installed")
try:
    from tkinter import Tk
    from time import sleep
    from tkinter.messagebox import showwarning
    import win32com.client as win32
except Exception as err_:
    raise ImportError("Maybe you do not have PyPiWin32 components installed, please \
                    use 'python -m pip install pypiwin32' to install and try again. %s"%err_)


def checkDaily(address='',regular='',dbaddress=''):
    '''address地址使用Unix风格斜杠，包含最后斜杠，为日记文件夹地址，regular为正则语法，dbaddress为数据库地址
        精确到文件
    '''
    try:
        import os,re,time,shelve
        notelist=os.listdir(address)
        reobj=re.compile(regular,re.S)
        notedict={}
        db_old=shelve.open(dbaddress)
        for key in db_old:
            notedict[key]=db_old[key]

        showbefore = '\n这是之前词典中保存的数据======================>\n%s'%notedict


        notefile_list=[]


        for x in notelist[:]:
            if reobj.search(x):
                if x not in notedict:
                    the_time=str(time.localtime()[0])+'/'+str(time.localtime()[1])+'/'+str(time.localtime()[2])+' '+str(time.localtime()[3])+":"+str(time.localtime()[4])
                    notedict[x]='Update at '+ the_time
                    notefile_list.append(str(x))
                else:
                    pass
            else:
                pass
        showafter = '\n这是准备写入词典中的数据======================>\n%s'%notedict

        return True,'%s\n\n%s'%(showbefore,showafter),notefile_list,notedict
    except:
        return False,traceback.format_exc(),notefile_list,notedict


def sendMail(notefile_list=[],address='',emailaddress='',dbaddress='',preparenotedict={}):
    '''存在两种错误，文件读不出来，邮件发不出去，这两种错误不触发异常，只需要记录并且给用户弹窗就好，但至于其他错误，比如数据库无法读写，则强制报错
        程序接受一个文件夹地址参数，此参数包含最后的斜杠，使用Unix风格斜杠；emailaddress为邮件地址，dbaddress为数据库文件地址；notefile_list为需要写入的目录
    '''
    processinfo = ''
    errinfo = ''
    try:
        if notefile_list:
            for notefile_real in notefile_list:
                processinfo += "\n=====================================[%s] PROCESSINFO====================================\n"%notefile_real
                errinfo += "\n=====================================[%s] ERRORINFO====================================\n"%notefile_real
                send_successful=False
                notefile=str(address+notefile_real)
                try:
                    document=Document(notefile)
                    text_article = [ paragraph.text for paragraph in document.paragraphs]
                except Exception as _err:
                    processinfo += "\n读取错误，详情请查看ERRORINFO。\n"
                    errinfo += "\n检测到的文件读取出错，因此不进行更新，已跳过此文件。\n详细信息:%s\n"%_err
                    continue

                mail_text_body=''
                mail_subject=str(text_article[:1])[2:-2]
                print('正在处理===>',mail_subject) #调试打开
                for pargh in text_article[1:]:
                    mail_text_body+=pargh+'\n'


                def outlook():
                    app='Outlook'
                    olook=win32.gencache.EnsureDispatch('%s.Application'%app)
                    mail=olook.CreateItem(win32.constants.olMailItem)
                    recip=mail.Recipients.Add(emailaddress)
                    subj=mail.Subject=mail_subject
                    mail.Body=mail_text_body
                    mail.Send()
                    olook.Quit()

                cpobject=document.core_properties
                def runprop(attr,cpobject,processinfo):
                    try:
                        val_=getattr(cpobject,attr)
                        if val_ !='':processinfo += (attr,'==>',val_)
                    except Exception as err_:
                        processinfo += "\nError:%s\n"%err_
                for attr in ['author','language','revision','version','last_printed','modified','keywords','comments','category']:
                    runprop(attr,cpobject,processinfo=processinfo) 

                    

                try:
                    outlook()
                    send_successful=True
                    processinfo += "邮件发送成功。"
                except Exception as err_:
                    if send_successful==True:
                        processinfo += '提示[%s]: 发送成功。但Outlook程序配置错误，请允许程序调用Outlook发送邮件。\n错误详情：%s'%(notefile_real,err_)
                    else:
                        processinfo += "发送失败，详情请查看ERRORINFO"
                        errinfo += '提示[%s]: 发送失败。Outlook程序配置错误，请允许程序调用Outlook发送邮件。\n错误详情：%s'%(notefile_real,err_)

                if send_successful==True: #如果发送成功再写入文件到数据库，否则不写入
                    try:
                        db=shelve.open(dbaddress)
                        for key in preparenotedict.keys():
                            db[key]=preparenotedict[key]
                        db.close()
                        processinfo += "\n数据库更新成功\n"
                    except:
                        processinfo += "\n数据库更新失败，详情请查看ERRORINFO\n"
                        errinfo += "\n邮件发送成功但数据库更新失败。\n"
                        raise ValueError("数据库更新失败")
                else:
                    pass
                
            return True,2,'完成数据处理',processinfo,errinfo
        else:
            return True,1,"未检测到新增的日记文件",processinfo,errinfo
    except:
        return False,0,traceback.format_exc(),processinfo,errinfo

if __name__=="__main__":
    #程序测试开始
    result,infomation,list,notedict= checkDaily(address='E:/Windows_WorkFolder/工作文件夹/',
            regular=r"\b201\d年\d+月.*?日.*?.doc.*?",dbaddress='E:/Windows_WorkFolder/工作文件夹/daily_update_data/update_data')
    if result == True:
        if list != []:
            print('需要写入的数据',list)
            result_2,result_num,result_txt,processinfo,errinfo= sendMail(list,address='E:/Windows_WorkFolder/工作文件夹/',emailaddress='dayone@mazhangjing.com',
                        dbaddress='E:/Windows_WorkFolder/工作文件夹/daily_update_data/update_data',preparenotedict=notedict)
            print(result_2,'\n',processinfo,'\n',errinfo,'\n',result_txt)
        else:
            print("成功检索数据，但未发现新数据")
            print(infomation,list)

```

所有文件可参考我的GitHub仓库：http://pybook.mazhangjing.com/Project_EveryDayNotice/

以上。





