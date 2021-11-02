---
layout: post
title: ThinkPad大升级!改装SSD混合HDD硬盘手记
date: 2014-06-19 04:58
author: user
comments: true
categories: [硬件折腾,过去的事]
---
电脑用久了，难免会折腾。笔者实在无法忍受自己的小黑卡顿如此，遂决定入手SSD，在贴吧潜水近一个月，搞清了安装步骤和部分黑幕，淘宝渠道购入商品，清单如下：

<strong>工具：</strong>

<strong>1,</strong><strong>十字口螺丝刀（大、小）</strong>

<strong>2,</strong><strong>外置光驱盒（附送挡板）</strong>

<strong>3,</strong><strong>笔记本光驱位支架（附送四个螺丝固定的，还有一块海绵来充实硬盘占位后空间利用不足的情况）</strong>

<strong>4,</strong><strong>固态硬盘（笔者此处选购的是浦科特</strong><strong>M6S</strong><strong>，</strong><strong>7mm</strong><strong>厚度）</strong>

<strong>5,</strong><strong>带有光驱的笔记本一台，</strong>

<strong>6,</strong><strong>装有</strong><strong>WinPE</strong><strong>的</strong><strong>U</strong><strong>盘一个，</strong>

<strong>7,Win8</strong><strong>安装光盘</strong>

<strong>环境：</strong>

<strong>UEFI,MBR</strong>

<strong>步骤</strong><strong>:</strong>

 1，&nbsp; 拆机，笔记本打开后面板，拆掉电池，拆下机械硬盘的支架上固定的螺丝，如图：

<br clear="all"  /> 注意把机械硬盘和托架，其实就是一个金属框框。拆下螺丝后按照图片安装SSD，并还原操作到笔记本主硬盘位。

&nbsp;

<p align="left"  > 开机设置AHCI启动（现在出厂预安装Win8系统的笔记本基本上都没有IDE模式了，即不需要操作。）</p>

<p align="left"  > 按F12进入启动项选择页面，选择开机启动，因为笔者电脑不能使用光驱启动，所以借用PE）之后，进入分区，格式化，选择对齐，512*8=4096以一个单位，对齐（64位推荐）选择Windows安装器，把开机引导文件等都放进这一个盘里，其实这就是MBR了。</p>

<p align="left"  > 至此，安装完毕。</p>

<p align="left"  >2，&nbsp; 把固定光驱的螺丝拧掉，抽出光驱盒。如图 然后如图 下一步完成光驱位硬盘的安装：</p>

<p align="left"  >3，&nbsp; 不要急着安装，先扣下原来光驱上带的挡板，这也是核心所在。</p>

<p align="left"  >4，&nbsp; 之后就不用多说了，把送的光驱挡板装到光驱上，安装到光驱盒里就可以了。</p>

<p align="left"  >5，&nbsp; ThinkVange APS硬盘保护系统测试</p>

<p align="left"  >&nbsp;</p>

<p align="left"  > 使用AS SSD软件测试,在开启APS保护情况下,不震动,SSD读写速率如图所示,最高476.25MB/S.</p>

<p align="left"  > 笔记本震动, 读写速率最高476MB/s,没有差别.</p>

<p align="left"  ><br clear="all"  /> 使用机械硬盘测试,不震动速率如下图: 最高80.3MB/s,开启震动后降到 因为中间硬盘驱动器停止读写,所以这个数据实际上是在系统忽略重复震动情况下读出来的,真正表明APS起作用的现象是后者测试用的时间是前者(不震动)的2倍多.</p>

<p align="left"  > 所以,APS可以完全识别光驱位的硬盘,据网络收集到的资料,APS原理是在机身内置了一个感应器,但有网友反映说如果更改驱动器,换掉原装的硬盘也无法工作,在笔者的测试过程中发现,如果仅仅是有SSD的话,APS系统同样不会检测到震动,所以我还是倾向于后者,即APS技术分别部署在SSD和TahinkPd机身内部.</p>

不管怎样,TP的APS技术都是必选软件,有效防护硬盘数据,并在混合硬盘的情况下可以识别SSD和HDD,实际上是不认识SSD,如此,在震动的条件下,使用SSD当主盘并不会有卡顿之感,以上.

**2014年6月19日**

