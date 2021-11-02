---
layout: post
title: Java Sound 简明教程以及节律纯音实现
categories:
  - Java
  - Scala
  - Java Sound
---

> 这是一份翻译自 [developer.com](https://www.developer.com/java/other/article.php/2226701/Java-Sound-Creating-Playing-and-Saving-Synthetic-Sounds.htm) 的关于 Java Sound 的简明教程，以及我利用 Java Sound 实现的多种节律纯音实现方案。本教程主要介绍了声音的物理和编程含义，此外讲解了 Java Sound Sampled 包的基本用法：包括录音到流、文件、从流、文件中播放、事件等。本文并未涉及 Control API 和 MIDI 包。


其他资源:

- Oracle Java Sound Demo: https://www.oracle.com/technetwork/java/index-139508.html
- Oracle Java Sound Tutorials: https://docs.oracle.com/javase/tutorial/sound/

# Sound API

## 什么是声音？

从人类的视角看，声音是当压力波冲击，并且和耳膜接触时产生的一种感觉。通常来讲，这是压力波在空气中传播造成的，但传播介质也不仅仅是空气，如果你在水下游泳，听到的声音则是通过水这一介质以声压的方式传到到耳朵的。

## Java Sound API 概述

Java Sound（javax.sound 包）的目的在于编写特定的程序，这种程序可以在特定的时刻操纵计算机硬件（操作系统和麦克风）通过物理方式（震动）产生压力波以改变听者的听觉感受。

Sun 对于 Java Sound 的定义是，其是一个低层次的 API，用于控制音频媒体的输入和输出，并且可以为其添加各种效果。对于声音输入和输出，Java Sound API 提供了控制媒介所需要的大多数操作，其可扩展性以及灵活性都非常好。

Java Sound 提供对于音频功能的高度控制，但是并不包含复杂的声音控制器，或者相关的 GUI 控制工具（比如那些用来控制声音效果的酷炫的 DJ 软件）。相反，Java Sound 提供了用于构建这些用户层工具的底层支持，对于应用层用户，这可能不是他们想要的，但是，对于构建应用层程序的开发者来说，这些低级接口总是非常有用的。

Java Sound API 并非是一个玩具，其涉及很多复杂的问题。这一部分内容将主要做一个介绍，首先本教程会梳理 Java Sound API 中重要的包，比如 MIDI 和 sampled 包，之后，将会介绍音频采样、渲染的典型步骤。

## Java Sound 包组成

Java Sound 包括两个功能截然不同的包，其一用于音频数据采样，其二用于乐器数字接口数据处理（MIDI）。

对于音频数据采样而言，被采样的数据可以被认为是一系列的数字值，其代表了声压波的振幅/强度。换句话说，声压波的信息化表示就是通过在一系列时间点上采样的声音振幅/强度数据。如下所示：

![](http://static2.mazhangjing.com/20190427/2c1d1c9_java2004a.gif)

上图是通过白噪音生成器生成的声音片段，类似于机场的噪音。对于 CD 而言，CD 中存储的也是采样音频数据，举例而言，音乐可以以 44100 的采样率进行采样，即在一个声压图上，每个横轴的坐标的密度为 1/44100 s （每秒 44100 个点），纵轴的坐标则表示其声压，一般用一个 16bit 的带符号的整数表示。捕获音频的功能位于 `javax.sound.sampled` 和 `javax.sound.sampled.spi` 下。前者用于捕获、混合、回放音频。

采样音频数据可以来自实际采样的声压波，也可以由计算机生成，比如语音合成器，其产生了采样音频数据，这些数据被通过硬件转换为声压波，之后被人耳所接受，人就会体验到说话的感觉。

对于 MIDI 数据（通常是音乐声音或者是特殊的声音效果），其相关功能分布在 `javax.sound.midi` 和 `javax.sound.spi` 中。前者用于为 MIDI 的合成、排序、事件传输提供支持。

关于捕获音频和 MIDI 音频的区别，根据 Sun 的说法，这里的 sampled 不代表音频数据的来源，而代表音频数据的类型。即，采样音频可以看做声音本身，而 MIDI 音频则可以看做创建音乐的配方/原料。
说明：关于捕获音频和 MIDI音频，spi 包都用于服务提供者，而不是应用开发者创建可安装在系统上的自定义组件。关于 Java 捕获音频数据，其涉及很多主题，可以参考 [《Digital Signal Processing (DSP) in Java, Sampled Time Series》](http://www.dickbaldwin.com/dsp/Dsp00104.htm)。

## 捕获采样音频

典型的采样音频数据的采集分为两个步骤：使用麦克风将声压波转换成模拟声压波波形的电压，之后使用模数转换器测量特定时间点的电压，将其测量值转换为数字值。

## 呈现采样音频

采样音频的呈现也分为两个步骤：使用数模转换器将一系列的数字值转换成模拟电压，其振幅波形反映数字的值。之后将这个电压施加到扬声器、耳机或者类似输出设备上，这些设备将模拟电压转换成模拟电压模型的声压波。

## 示例代码

下面这个程序将会为你提供一些进一步了解 Java Sound API 的兴趣，其展示了使用 Java Swing GUI 和 Sound API 从麦克风捕获音频数据，然后通过计算机的扬声器回放的能力。

```java
public class AudioCapture01 extends JFrame{

    private boolean stopCapture = false;
    private ByteArrayOutputStream byteArrayOutputStream;
    private TargetDataLine targetDataLine;
    private AudioInputStream audioInputStream;
    private SourceDataLine sourceDataLine;
    
    public static void main(String[] args){ new AudioCapture01(); }

    public AudioCapture01(){
        final JButton captureBtn = new JButton("Capture");
        final JButton stopBtn = new JButton("Stop");
        final JButton playBtn = new JButton("Playback");

        captureBtn.setEnabled(true);
        stopBtn.setEnabled(false);
        playBtn.setEnabled(false);

        captureBtn.addActionListener(e -> {
            captureBtn.setEnabled(false);
            stopBtn.setEnabled(true);
            playBtn.setEnabled(false);
            captureAudio(); }
        );
        getContentPane().add(captureBtn);

        //end actionPerformed
        stopBtn.addActionListener(e -> {
            captureBtn.setEnabled(true);
            stopBtn.setEnabled(false);
            playBtn.setEnabled(true);
            stopCapture = true;
        }
        );
        getContentPane().add(stopBtn);

        //end actionPerformed
        playBtn.addActionListener(e -> playAudio());
        getContentPane().add(playBtn);

        getContentPane().setLayout(new FlowLayout());
        setTitle("Capture/Playback Demo");
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setSize(350,70);
        setVisible(true);
    }

    //This method captures audio input from a microphone and saves it in a ByteArrayOutputStream object.
    private void captureAudio(){
        try{
            //Get everything set up for capture
            AudioFormat audioFormat = getAudioFormat();
            DataLine.Info dataLineInfo = new DataLine.Info(TargetDataLine.class, audioFormat);
            targetDataLine = (TargetDataLine) AudioSystem.getLine(dataLineInfo);
            targetDataLine.open();
            targetDataLine.start();

            //Create a thread to capture the microphone data and start it running.  It will run until
            //the Stop button is clicked.
            Thread captureThread = new Thread(new CaptureThread());
            captureThread.start();
        } catch (Exception e) {
            e.printStackTrace(System.out);
            System.exit(0);
        }
    }

    //This method plays back the audio data that has been saved in the ByteArrayOutputStream
    private void playAudio() {
        try{
            //Get everything set up for playback.
            //Get the previously-saved data into a byte array object.
            byte[] audioData = byteArrayOutputStream.toByteArray();
            //Get an input stream on the byte array containing the data
            InputStream byteArrayInputStream = new ByteArrayInputStream(audioData);
            AudioFormat audioFormat = getAudioFormat();
            audioInputStream = new AudioInputStream(byteArrayInputStream, audioFormat,
                            audioData.length/audioFormat.getFrameSize());
            DataLine.Info dataLineInfo = new DataLine.Info(SourceDataLine.class, audioFormat);
            sourceDataLine = (SourceDataLine) AudioSystem.getLine(dataLineInfo);
            sourceDataLine.open(audioFormat);
            sourceDataLine.start();

            //Create a thread to play back the data and start it running.  It will run until
            //all the data has been played back.
            Thread playThread = new Thread(new PlayThread());
            playThread.start();
        } catch (Exception e) {
            e.printStackTrace(System.out);
            System.exit(0);
        }
    }

    //This method creates and returns an AudioFormat object for a given set of format parameters.  If these
    // parameters don't work well for you, try some of the other allowable parameter values, which
    // are shown in comments following the declarations.
    private AudioFormat getAudioFormat(){
        float sampleRate = 44100;
        //8000,11025,16000,22050,44100
        int sampleSizeInBits = 16;
        //8,16
        int channels = 1;
        //1,2
        boolean signed = true;
        //true,false
        boolean bigEndian = true;
        //true,false
        return new AudioFormat(sampleRate, sampleSizeInBits, channels, signed, bigEndian);
    }
    
    class CaptureThread extends Thread{
        //An arbitrary-size temporary holding buffer
        byte[] tempBuffer = new byte[10000];
        public void run(){
            byteArrayOutputStream = new ByteArrayOutputStream();
            stopCapture = false;
            try{
                while(!stopCapture){
                    //Read data from the internal buffer of the data line.
                    int cnt = targetDataLine.read(tempBuffer, 0, tempBuffer.length);
                    if(cnt > 0){
                        //Save data in output stream object.
                        byteArrayOutputStream.write(tempBuffer, 0, cnt);
                    }
                }
                byteArrayOutputStream.close();
            }catch (Exception e) {
                e.printStackTrace(System.out);
                System.exit(0);
            }
        }
    }

    class PlayThread extends Thread{
        byte[] tempBuffer = new byte[10000];

        public void run(){
            try{
                int cnt;
                //Keep looping until the input read method returns -1 for empty stream.
                while((cnt = audioInputStream.read(tempBuffer, 0, tempBuffer.length)) != -1){
                    if(cnt > 0){
                        //Write data to the internal buffer of the data line where it will be delivered to the speaker.
                        sourceDataLine.write(tempBuffer, 0, cnt);
                    }
                }
                //Block and wait for internal buffer of the data line to empty.
                sourceDataLine.drain();
                sourceDataLine.close();
            }catch (Exception e) {
                e.printStackTrace(System.out);
                System.exit(0);
            }
        }
    }
}
```


# Playback

> Java Sound API 基于 lines 和 mixers。下面将会介绍声音的物理和电特性，之后会介绍音频混合器 mixer 的用法，本文将会以一场摇滚音乐会为例，使用六个麦克风和两个立体声扬声器来进一步介绍混音器的进阶用途。此外，在这篇文章中，将会介绍 Java Sound 编程的核心概念，包括 mixers、lines、音频格式等等。几个重要的类对象是 SourceDataLine、Clip、Mixer、AudioFormat。

## 从演唱会说起

![](http://static2.mazhangjing.com/20190427/d2f7fa5_java2008a.gif)

如上所示，一个正在发表演讲的人，这个人通过扩音器进行演讲，扩音器系统通常由麦克风、放大器和扩音器组成。扩音器的作用是将人的声音方法，以便人群可以清楚的听到声音。当人说话时，其声带带动喉咙中的空气震动，进而导致其产生的声压冲击麦克风，麦克风的鼓膜震动被转换成模仿声压波的极低水平的电波，电波被输入一个放大器，被方法，从那里出来后，电波被输入一个扩音器，扩音器将放大的电波转换为高强度的声压波，模仿其声带发出的原始声压波。

下面考虑麦克风的内部构造：

![](http://static2.mazhangjing.com/20190427/a4e372d_java2008b.gif)

声压波撞击振膜，振膜带动一根附着的电线震动，当振膜时，线圈在强永磁体引起的磁场中来回运动。当线圈穿过磁场时，线圈中就产生了电压。

扬声器中的振膜震动使周围空气震动，产生高强度的声压波，这些声压波的波形模拟了声带产生的低强度的声压波。新的声压波强大到足以冲击到人群后面人的耳朵，因此，通过这样一种设施，由人产生的较弱的声音就可以被很远的距离的人听到。

考虑到如下的摇滚音乐会，其包含一组声带、一组麦克风、一个放大器、一个扩音器。当演唱会开始时，六个独立的舞台上的麦克风将会通过其所属的六个演奏者的音乐和人声所激发并且产生电信号，这些电信号必须被单独方法并且输入两个扩音器中。此外，特殊的声音效果，比如混响，和电信号一起，也被加入扩音器中。

![](http://static2.mazhangjing.com/20190427/c179cd4_java2008d.gif)

这两个扬声器旨在提供立体声音体验，比如，最右边的麦克风产生的所有电信号可能只适用于右边的扬声器，同样，左边麦克风产生的电信号则适用于左边的扬声器，其他麦克风则根据其位置比例不同而作用于两个不同的扬声器。这样的要求是通过音频混频器来完成的。一个典型的混频器 mixer 应该有能力接受许多独立的电信号，每个电信号代表一个独立的音频信号或线路 Line。

在任何情况下，一个典型的音频混频器能够对每条音频线进行独立的信号放大。此外，混频器还可以对每一个 Line 应用特殊的音频效果，比如混响，最后，Mixer 能够将单个放大的信号混合到输出行中，并且控制每个输入行对每个输出行的贡献（贡献通常被称作 pan）。

因此，在上图的场景中，调音师能够将 6 个麦克风产生的信号相加，从而产生两个输出信号，每个输出信号将被应用到其中的一个立体声扬声器上。实际上，六个麦克风的信号可以叠加产生的两个输出信号，其 pan 可以根据麦克风的位置不同来动态进行改变，这样，调音师通过根据六个麦克风的位置控制其各自 pan 值，就可以改变调音效果，从而为听众提供立体的音频体验。

说回 Java Sound API，Sun 描述如下：**Java Sound 不假定特定的音频硬件配置，其涉及允许在系统上安装不同种类的音频组件，并允许 API 编程访问这些组件。API 支持基本的声卡输入和输出控制，比如用于录制和回访声音文件，以及混合多个音频流。**

基本上，Java Sound API 是围绕 mixer 和 lines 构建的。

## Mixer

Sun 对于 mixer 的描述如下：混音器是一种包含一条或者多条线的音频设备，它并非一定要用于混合音频信号。实际上，混音器有多个输入源/行和至少一个输出源/行。前者通常是 SourceDataLine 实例，后者则是 TargetDataLine 实例。Port 对象可以是输入或者输出行。Mixer 接受预先录制的、可循环的声音作为输入，通过将 SourceDataLine 转型为 Clip 接口类型来实现。

## Line

Sun 对于 Line 的描述如下：Line 是数字音频的管道，比如音频输入或者输出端口 —— 用于进出混频器的路径。通过一条 Line 的音频数据可以是单声道的，也可以是多声道的，比如立体声。行可以添加控制，比如增益、平移和混响。

总的来说，其关系如下：

![](http://static2.mazhangjing.com/20190427/9c37cdb_java2008e.gif)


从编程的角度来看，图中显示了一个 Clip 对象和两个 SourceDataLine 对象作为输入构建的一个 Mixer 对象。这里的 Clip 指的是一个混合的输入对象，其内容不随时间变化，换句话说，在播放前将音频数据加载到 Clip 对象中，然后此 Clip 对象可以回放一次或者多次，你可以让它循环，重复播放。

而 SourceDataLine 和 Clip 不同，其实一个流式的混合输入对象，就像 XXXInputStream 一样，这种类型的对象可以接受音频数据流，并且实时的将音频数据推入混频器。这些流数据可以从很多地方产生，比如音频文件、网络连接、内存缓冲区等等。Clip 和 SourceDataLine 都可以被看做混频器的输入 Line，每个输入 Line 都可以有自己的混响、增益和 pan 控制。

如图所示，混频器从输入行读取数据，使用 Controls 混合输入信号，并且最终将信号发送到一个或者多个输出端口，比如扬声器、耳机插孔等。

## 示例代码讲解 - 播放

### 数据流准备

对于数据的播放而言，因为必须先有一个 AudioInputStream，而 AIS 需要一个 InputStream 来构造，因此，首先需要从 ByteArrayOutputStream 流读取 byte 数组数据，然后用数据构造一个新的 ByteArrayInputStream，传入 AIS 以准备好数据。

```java
byte audioData[] = byteArrayOutputStream.toByteArray();
InputStream byteArrayInputStream = new ByteArrayInputStream(audioData);
```

### 音频格式准备

音频数据在这里可以清楚的看到包含两个部分，其一是用于保存每个时刻声压的原始数据，其二是包含音频数据本身格式的元数据。Sun 认为，**AudioFormat 即格式，是特指音频流中字节的排列，其属性包括通道的数量、采样率、采样大小、编码技术。常用的编码技术包括线性脉冲编码调制 PCM，mu-law 以及 a-law 编码。byte 数组保存的是单纯的原始数据**，有很多可选的方式来解释这些字节序列，AudioFormat 显然是为此准备的。

```java
  private AudioFormat getAudioFormat(){
    float sampleRate = 8000.0F;
    int sampleSizeInBits = 16;
    int channels = 1;
    boolean signed = true;
    boolean bigEndian = false;
    return new AudioFormat(sampleRate,sampleSizeInBits,channels,signed,bigEndian);
  }
```

AudioFormat 包含可供选择的构造器，其中包括每秒抽样率/SampleRate，即每秒包含多少个声压的取样点。可选的有 8000， 11025， 16000， 22050 和 44100。此外还包括每个采样存储所用的比特数，可选的有 8 和 16。通道则包括单声道和双声道，此外，signed or nosigned 数据将会控制存储在 8/16 bit 的采样使用的是有符号还是无符号的数据。big-endian or little-endian 用于控制数据在内存中的排序方式。默认的编码方案是 PCM。

### 构建 AudioInputStream

```java
audioInputStream =new AudioInputStream(
          byteArrayInputStream, audioFormat, audioData.length/audioFormat.getFrameSize());
```

播放声音需要通过 AudioInputStream 流，此流可通过 ByteArrayInputStream 嵌套构造。而后者则可以通过传入一个 byte 数组来构造。因此，通过这种装饰器模式，byte 数组被包裹成为 ByteArrayInputStream 之后被包裹成为 AudioInputStream。其中AIS 需要耦合一个 AudioFormat 以解析 byte 数组中的数据，此外，还需要手动传入数据的帧数，这里使用 length/FrameSize（采样的总字节长度/每个帧的字节长度） 来获得。**帧是什么？对于 PCM 而言，帧指的是一个特定时间点上所有信道的一组样本组成。因此，一帧的长度对应样本的长度 * 通道数，一个样本的长度对应 1 或者 2 个 Byte（8 bit 或者 16bit）。**

> 注意，这里要区分采样 sample、帧 frame、数据 byte[] 的区别。采样占据比特数可选为 8bit 或者 16bit，如果是双通道，那么一个帧将包含两个采样（对应左右耳），也就是占据 16bit 或者 32bit。所有帧的长度总是对应 byte 数组的长度，而采样的长度仅在单通道的情况下才等于所有帧的长度，等于 byte 数组的长度。

### 获取 SourceDataLine

当我们终于通过 byte[] 和 AudioFormat 构建好 AudioInputStream，这时候就可以调用 SourceDataLine 来将 InputStream 实时输入到此 Line 上了吗？不是，首先，需要通过 DataLine.Info 对象来创建一个合适 AudioFormat 格式的 DataLine.Info 对象，然后拿着这个凭证向 AudioSystem 请求获取一个 Line，很显然，因为 DataLine.Info 包含了所需要 Line 的格式信息、输入属性（音频格式、内部缓冲区的最大和最小值），因此获取到的 Line 可以转型为 SourceDataLine。

```java
DataLine.Info dataLineInfo = new DataLine.Info(SourceDataLine.class, audioFormat);
sourceDataLine = (SourceDataLine)AudioSystem.getLine(dataLineInfo);
```

**SourceDataLine 被定义为可以写入数据的 Line，其作为混频器的来源。应用程序将音频字节写入源数据行，源数据行处理字节的缓冲并且将其发送到混频器。之后 Mixer 可能将其交付给一个默认的当前目标，比如某个输出端口。**

```java
sourceDataLine.open(audioFormat);
sourceDataLine.start();
```

当我们终于有了 SourceDataLine 之后，就可以使用其 API 来 open、start 执行流的释放了。这里的 open 用来以我们传入的特定格式打开这个 Line，然后使其获取所需的系统资源并且开始运行。

open 执行后影响到了系统的资源分配，如果打开成功，那么系统资源已经分配到了这个行。其会获取本机的硬件资源，比如声卡，并且初始化其所需的软件组件。资源总是有限的，即一个混频器的行数是有限的，在某个时候，多个应用程序或者一个应用程序可能会竞相使用混频器。当调用 start 方法后，即允许行进行数据的输入和输出了，如果在一个 started 的行上运行此方法，则不执行任何操作，除非缓冲区的数据已被刷新，否则该行将会从其停止时未处理的第一帧开始恢复 IO 操作（即调用停止后，start 在未刷新的情况下还会从最后一帧继续运行）。

```java
int cnt;
while((cnt = audioInputStream.read(tempBuffer, 0,tempBuffer.length)) != -1){
    if(cnt > 0){
    sourceDataLine.write(tempBuffer, 0, cnt);
}
```

通过对 AudioInputStream 和 SourceDataLine 的 write，SDL 逐渐缓冲了一些数据，调用 drain 将会阻塞线程并且等待 SourceDataLine 内部缓冲区清空，当其清空后，这意味着所有的音频数据都已经被发送到扬声器（并不需要每次都清空 SDL，因为当在 while 循环中 write 时，缓冲区在被覆盖前，会播放目前尚未被覆盖的声音，以腾空位置放置新数据，只有在最后 while 结束后的那部分才需要 drain）。

之后执行的 close 将会关闭 Line，其会释放系统资源。应用程序必须在退出前关闭所有打开的行，因为混频器被认为是共享的系统资源，可以反复打开和关闭。

# Capture

## Mixer

Mixer 是一种由一条或者多条 Line 的音频设备，其并非一定要混合音频信号。一个 Mixer 包含多个输入 Line 和至少一个输出 Line。前者是 SourceDataLine 实例，后者是 TargetDataLine 实例。Port 对象可以是输入或者输出行（Line 的实现），Mixer 接受提前录制的、可循环的声音片段，称之为 Clip，其也是 Line 的实现。

## Line

Line 是数字音频的管道，比如输入和输出，进出 Mixer 的路径。一条 Line 可以是单声道的，也可以是多声道的，其可以被控制，比如增益、平移和混响。

![](http://static2.mazhangjing.com/20190427/5f42c1f_java2012a.gif)

如上图所示，Mixer 包含了一个或者多个 Ports，一些 Controls 和一个 TargetDataLine 用于输出。TDL 用于从 Mixer 传送输出的音频数据，也经常用作程序其他部分的输入。TDL 的数据可以实时推送到其他程序中，比如音频文件、网络连接或者内存中的缓冲区。

不是所有的操作系统提供完全一致的混频器，使用 `javax.sound.sampled.AudioSystem.getMixerInfo` 获取此信息，在我的电脑上，如下 MixerInfo 可用：`Array(Default Audio Device, 外置耳机,  Mac mini 扬声器, Port 外置耳机, Port Mac mini 扬声器)`

Mixer.Info 类型是用来描述一个 Audio Mixer 的类，其包括了一个产品名、版本号和文本描述。Mixer.Info 也是最快获取一个特定 Mixer 的途径。

Line.Info 用于获取 Line 对象，在上文中的 SourceDataLine.Info 中已经介绍过了。使用 DataLine.Info 构造器可以通过传入一个 TargetDataLine.class 和一个指定的格式来构造此 Info，这样的一个 Line.Info 对象包含了对于一条 Line 的描述。其承载的信息根据其子类类型的不同而不同，比如 Clip、TargetDataLine、SourceDataLine 等。一般而言，一个 Info 总是知道其 Line 支持的音频格式、最小和最大的内部缓冲区大小。上文介绍过你可以通过 AudioSystem.getLine 通过 Line.Info 获取 Line 对象，这里使用的是默认的 Mixer，本质上是 DefalutMixer.getLine。Line.Info 的构造可以传入缓冲区大小、最大和最小大小，或者使用默认值。

> 区别 SourceDataLine 和 TargetDataLine，前者用于向 Mixer 发送数据，后者用于从 Mixer 获取数据，比如捕获声音等。在人的角度看，其实这两者恰好是反过来的，SourceDataLine 允许我们播放音频，TargetDataLine 则用于收集声音。注意，这里的命名是相对于 Mixer 而言的。

关于 TargetDataLine 需要注意的是，这是 Mixer 的一个输出 Port，因此，录制音频的应用程序应该足够快的从 TDL 读取数据，以防止缓冲区溢出（即上述要求我们定义缓冲区大小的那个缓冲区），如果溢出，那么新数据覆盖最老的旧数据，这可能导致捕获数据的不连续。

需要注意，一旦通过 Mixer.Info 获取 Mixer，通过 Mixer 和 Line.Info 获取 TargetDataLine，你就可以通过 open 和 start 来进行处理了。open 用于分配资源，而 start 则用于音频 IO 的开始处理。注意，Application 必须以一个合适的速度从 Line 的内部缓冲区获取数据，否则其内部缓冲区的数据会被自动覆盖。

一般通过 read 读取 Line 的缓冲区数据，注意，如果缓冲区尚未有足够的数据，那么其会阻塞并且等待读取，如果缓冲区有足够的数据，那么其会先传输足够的数据，保留未传输的数据，等待下一次传输。
在读取结束后，应该调用 TDL 的 stop 方法以停止数据向 TDL 的不断覆盖和输送。注意，得到的 byte[] 如果保存在内存中，可以保存在像 ByteArrayOutputStream 之类的地方，因为这是一个长度可变的容器（由其内部自动维护长度），唯一需要担心的就是 OutOfMemory 问题。

# File

音频如果一直保存在内存中，容易丢失，如果音频过大，还可能造成内存不足的问题。下面讲解了捕获声音到文件、从文件中播放的技术。

对于捕获到文件的关键在于 AudioSystem 的 write 方法，其可以直接将 TargetDataLine 和文件耦合，自动建立流，当 TDL stop 的时候自动完成 File 的释放。这避免了手动的先保存到 Stream，然后从 Stream 自己写到 File 中的问题。

对于从文件播放，需要定义 File 的 AudioFileFormat.Type，其内含了可用的 AudioFormat，直接输送到 AudioInputStream 在 SourceDataLine 播放即可。

## Capture to File

下面的代码用来捕获样本数据到文件中：

```java
import javax.swing.*;
import java.awt.*;
import java.io.*;
import javax.sound.sampled.*;

public class AudioRecorder02 extends JFrame{

    private AudioFormat audioFormat;
    private TargetDataLine targetDataLine;

    private final JButton captureBtn = new JButton("Capture");
    private final JButton stopBtn = new JButton("Stop");

    private final JRadioButton aifcBtn = new JRadioButton("AIFC");
    private final JRadioButton aiffBtn = new JRadioButton("AIFF");
    private final JRadioButton auBtn = new JRadioButton("AU",true); //selected at startup
    private final JRadioButton sndBtn = new JRadioButton("SND");
    private final JRadioButton waveBtn = new JRadioButton("WAVE");

    public static void main(String[] args){
        new AudioRecorder02();
    }

    public AudioRecorder02(){
        captureBtn.setEnabled(true);
        stopBtn.setEnabled(false);

        //Register anonymous listeners
        //end actionPerformed
        captureBtn.addActionListener(e -> {
            captureBtn.setEnabled(false);
            stopBtn.setEnabled(true);
            captureAudio();
        });

        //end actionPerformed
        stopBtn.addActionListener(e -> {
            captureBtn.setEnabled(true);
            stopBtn.setEnabled(false);
            targetDataLine.stop();
            targetDataLine.close();
        });

        //Put the buttons in the JFrame
        getContentPane().add(captureBtn);
        getContentPane().add(stopBtn);

        //Include the radio buttons in a group
        ButtonGroup btnGroup = new ButtonGroup();
        btnGroup.add(aifcBtn);
        btnGroup.add(aiffBtn);
        btnGroup.add(auBtn);
        btnGroup.add(sndBtn);
        btnGroup.add(waveBtn);

        //Add the radio buttons to the JPanel
        JPanel btnPanel = new JPanel();
        btnPanel.add(aifcBtn);
        btnPanel.add(aiffBtn);
        btnPanel.add(auBtn);
        btnPanel.add(sndBtn);
        btnPanel.add(waveBtn);

        //Put the JPanel in the JFrame
        getContentPane().add(btnPanel);

        //Finish the GUI and make visible
        getContentPane().setLayout(new FlowLayout());
        setTitle("Copyright 2003, R.G.Baldwin");
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setSize(300,120);
        setVisible(true);
    }

    //This method captures audio input from a
    // microphone and saves it in an audio file.
    private void captureAudio(){
        try{
            //Get things set up for capture
            audioFormat = getAudioFormat();
            DataLine.Info dataLineInfo = new DataLine.Info(TargetDataLine.class, audioFormat);
            targetDataLine = (TargetDataLine) AudioSystem.getLine(dataLineInfo);

            new CaptureThread().start();
        }catch (Exception e) {
            e.printStackTrace();
            System.exit(0);
        }
    }

    //This method creates and returns an
    // AudioFormat object for a given set of format
    // parameters.  If these parameters don't work
    // well for you, try some of the other
    // allowable parameter values, which are shown
    // in comments following the declarations.
    private AudioFormat getAudioFormat(){
        float sampleRate = 8000.0F;
        //8000,11025,16000,22050,44100
        int sampleSizeInBits = 16;
        //8,16
        int channels = 1;
        //1,2
        boolean signed = true;
        //true,false
        boolean bigEndian = false;
        //true,false
        return new AudioFormat(sampleRate, sampleSizeInBits, channels, signed, bigEndian);
    }

    class CaptureThread extends Thread{
        public void run(){
            AudioFileFormat.Type fileType = null;
            File audioFile = null;

            //Set the file type and the file extension
            // based on the selected radio button.
            if(aifcBtn.isSelected()){
                fileType = AudioFileFormat.Type.AIFC;
                audioFile = new File("junk.aifc");
            }else if(aiffBtn.isSelected()){
                fileType = AudioFileFormat.Type.AIFF;
                audioFile = new File("junk.aif");
            }else if(auBtn.isSelected()){
                fileType = AudioFileFormat.Type.AU;
                audioFile = new File("junk.au");
            }else if(sndBtn.isSelected()){
                fileType = AudioFileFormat.Type.SND;
                audioFile = new File("junk.snd");
            }else if(waveBtn.isSelected()){
                fileType = AudioFileFormat.Type.WAVE;
                audioFile = new File("junk.wav");
            }

            try{
                targetDataLine.open(audioFormat);
                targetDataLine.start();
                AudioSystem.write(new AudioInputStream(targetDataLine), fileType, audioFile);
            }catch (Exception e){
                e.printStackTrace();
            }
        }
    }
}
```

可以看到，大致和之前很类似。这里着重强调一下 AudioSystem 和 AudioFileFormat 两个对象。**AudioSystem 包含了一系列的方法，这些方法用于在不同格式之间转换音频数据、在流和文件之间传输数据。AS 也提供了一些静态方法，用于直接从默认的 Mixer 获取 Line，而不需要先构建 Mixer，然后再获取 Line。**

**AudioFileFormat.Type 类提供了几种内置的保存到文件中的音频格式类，AIFC，AIFF，AU，SND，WAVE 等。各个平台对其支持可能有差别。调用 AudioSystem#write 方法可以快速的将一个包装了 TargetDataLine 的 AudioInputStream 按照 AudioFileFormat.Type 的格式输出到一个 File 文件中。**

_注意，这里包裹 TDL 的 AIS 不需要执行关闭，当 TDL 停止的时候，即 stop 方法调用的时候偶，AIS 会自动关闭其内部依赖，包括流和文件输出。_ 但是至于 AIS 依赖的 ByteArrayInputStream 需不需要手动关闭呢？这是个问题。


## Playing Back Audio Files

执行如下所述的代码，以从文件中播放声音。

```java
import javax.sound.sampled.*;
import javax.swing.*;
import java.io.File;

public class AudioPlayer02 extends JFrame{

    private AudioFormat audioFormat;
    private AudioInputStream audioInputStream;
    private SourceDataLine sourceDataLine;
    private boolean stopPlayback = false;
    private final JButton stopBtn = new JButton("Stop");
    private final JButton playBtn = new JButton("Play");
    private final JTextField textField = new JTextField("junk.au");

    public static void main(String[] args){
        new AudioPlayer02();
    }

    public AudioPlayer02(){
        stopBtn.setEnabled(false);
        playBtn.setEnabled(true);

        //end actionPerformed
        playBtn.addActionListener(e -> {
            stopBtn.setEnabled(true);
            playBtn.setEnabled(false);
            playAudio();
        });

        //end actionPerformed
        stopBtn.addActionListener(e -> stopPlayback = true);

        getContentPane().add(playBtn,"West");
        getContentPane().add(stopBtn,"East");
        getContentPane().add(textField,"North");

        setTitle("Copyright 2003, R.G.Baldwin");
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setSize(250,70);
        setVisible(true);
    }

    private void playAudio() {
        try{
            File soundFile = new File(textField.getText());
            audioInputStream = AudioSystem.getAudioInputStream(soundFile);
            audioFormat = audioInputStream.getFormat();
            System.out.println(audioFormat);

            DataLine.Info dataLineInfo = new DataLine.Info(SourceDataLine.class, audioFormat);

            sourceDataLine = (SourceDataLine)AudioSystem.getLine(dataLineInfo);

            new PlayThread().start();
        }catch (Exception e) {
            e.printStackTrace();
            System.exit(0);
        }
    }

    class PlayThread extends Thread{
        byte[] tempBuffer = new byte[10000];

        public void run(){
            try{
                sourceDataLine.open(audioFormat);
                sourceDataLine.start();

                int cnt;
                while((cnt = audioInputStream.read(tempBuffer,0,tempBuffer.length)) != -1 && !stopPlayback){
                    if(cnt > 0) sourceDataLine.write(tempBuffer, 0, cnt);
                }
                sourceDataLine.drain();
                sourceDataLine.close();

                //Prepare to playback another file
                stopBtn.setEnabled(false);
                playBtn.setEnabled(true);
                stopPlayback = false;
            }catch (Exception e) {
                e.printStackTrace();
                System.exit(0);
            }
        }
    }
}
```

播放声音使用的是 AudioSystem 的 getAudioInputStream 方法，此方法接受一个文件，从中获取流。AudioFormat 可以从此 AIS 中从文件直接读取。可以认为，byte[] 数组保存了基本的数据，AudioFormat 保存了基本的元信息，而 AudioFileFormat 以及各种格式的采样音频则提供了不同的数据压缩实现，其内涵了 byte[] 和 AudioFormat 的内容，即可以看做后者是前者的实现。

这样的话，从文件中读取数据，直接通过一个 buffer 在 AIS 和 SourceDataLine 之间传递数据即可。

# Audio Event

可以为 DataLine 添加监听器：

```java
import javax.sound.sampled.*;
import javax.swing.*;
import java.awt.*;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.InputStream;

public class AudioEvents01 extends JFrame{

    private boolean stopCapture = false;
    private ByteArrayOutputStream byteArrayOutputStream;
    private AudioFormat audioFormat;
    private TargetDataLine targetDataLine;
    private AudioInputStream audioInputStream;
    private SourceDataLine sourceDataLine;

    public static void main(String args[]){
        new AudioEvents01();
    }

    public AudioEvents01(){
        final JButton captureBtn = new JButton("Capture");
        final JButton stopBtn = new JButton("Stop");
        final JButton playBtn = new JButton("Playback");

        captureBtn.setEnabled(true);
        stopBtn.setEnabled(false);
        playBtn.setEnabled(false);

        //Register anonymous listeners
        captureBtn.addActionListener(e -> {
            captureBtn.setEnabled(false);
            stopBtn.setEnabled(true);
            playBtn.setEnabled(false);
            captureAudio();
        });
        getContentPane().add(captureBtn);

        //end actionPerformed
        stopBtn.addActionListener(e -> {
            captureBtn.setEnabled(true);
            stopBtn.setEnabled(false);
            playBtn.setEnabled(true);
            stopCapture = true;
        });
        getContentPane().add(stopBtn);

        playBtn.addActionListener(e -> playAudio());
        getContentPane().add(playBtn);

        getContentPane().setLayout(new FlowLayout());
        setTitle("Copyright 2003, R.G.Baldwin");
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setSize(250,70);
        setVisible(true);
    }

    private void captureAudio(){
        try{
            //Get everything set up for capture
            audioFormat = getAudioFormat();
            DataLine.Info dataLineInfo = new DataLine.Info(TargetDataLine.class, audioFormat);
            targetDataLine = (TargetDataLine)AudioSystem.getLine(dataLineInfo);

            targetDataLine.addLineListener(e -> {
                        System.out.println("Event handler for TargetDataLine");
                        System.out.println("Event type: " + e.getType());
                        System.out.println("Line info: " + e.getLine().getLineInfo());
                        System.out.println();
                    }
            );
            new CaptureThread().start();
        }catch (Exception e) {
            e.printStackTrace(System.out);
            System.exit(0);
        }
    }

    private void playAudio() {
        try{
            byte[] audioData = byteArrayOutputStream.toByteArray();
            InputStream byteArrayInputStream = new ByteArrayInputStream(audioData);
            AudioFormat audioFormat = getAudioFormat();
            audioInputStream = new AudioInputStream(byteArrayInputStream, audioFormat, audioData.length/audioFormat.getFrameSize());
            DataLine.Info dataLineInfo = new DataLine.Info(SourceDataLine.class, audioFormat);
            sourceDataLine = (SourceDataLine)AudioSystem.getLine(dataLineInfo);
            sourceDataLine.addLineListener(e -> {
                        System.out.println("Event handler for SourceDataLine");
                        System.out.println("Event type: " + e.getType());
                        System.out.println("Line info: " + e.getLine().getLineInfo());
                        System.out.println();//blank line
                    }
            );
            new PlayThread().start();
        }catch (Exception e) {
            e.printStackTrace(System.out);
            System.exit(0);
        }
    }

    private AudioFormat getAudioFormat(){
        float sampleRate = 8000.0F;
        //8000,11025,16000,22050,44100
        int sampleSizeInBits = 16;
        //8,16
        int channels = 1;
        //1,2
        boolean signed = true;
        //true,false
        boolean bigEndian = false;
        //true,false
        return new AudioFormat(sampleRate, sampleSizeInBits, channels, signed, bigEndian);
    }

    class CaptureThread extends Thread{
        byte[] tempBuffer = new byte[10000];
        public void run(){
            byteArrayOutputStream = new ByteArrayOutputStream();
            stopCapture = false;
            try{
                targetDataLine.open(audioFormat);
                targetDataLine.start();
                while(!stopCapture){
                    int cnt = targetDataLine.read(tempBuffer, 0, tempBuffer.length);
                    if(cnt > 0){
                        byteArrayOutputStream.write(tempBuffer, 0, cnt);
                    }
                }
                byteArrayOutputStream.close();
                targetDataLine.stop();
                targetDataLine.close();
            }catch (Exception e) {
                e.printStackTrace(System.out);
                System.exit(0);
            }
        }
    }

    class PlayThread extends Thread{
        byte[] tempBuffer = new byte[10000];

        public void run(){
            try{
                int cnt;
                sourceDataLine.open(audioFormat);
                sourceDataLine.start();
                while((cnt = audioInputStream.read(tempBuffer, 0, tempBuffer.length)) != -1){
                    if(cnt > 0){
                        sourceDataLine.write(tempBuffer, 0, cnt);
                    }
                }
                sourceDataLine.drain();
                sourceDataLine.close();
            }catch (Exception e) {
                e.printStackTrace(System.err);
                System.exit(0);
            }
        }
    }
}
```

LineEvent 的 Type 可选有 Open、Close、Start、Stop。可以通过 LineEvent.getLine.getLineInfo 获取 Line 和其 Info。在 LineListener 中，对于简单动作，update 直接执行，对于复杂动作，通过创建新线程执行即可。

# Synthetic Sound

如下代码可以生成单、双声道的各种声音，并且播放或者发送到文件中。

```java
public class AudioSynth01 extends JFrame{

    private AudioFormat audioFormat;
    private AudioInputStream audioInputStream;
    private SourceDataLine sourceDataLine;

    private static float sampleRate = 16000.0F;
    private static int channels = 1;
    //Allowable true,false

    //A buffer to hold two seconds monaural and one
    // second stereo data at 16000 samp/sec for
    // 16-bit samples
    private byte[] audioData = new byte[16000 * 4];

    private final JButton generateBtn = new JButton("Generate");
    private final JButton playOrFileBtn = new JButton("Play/File");
    private final JLabel elapsedTimeMeter = new JLabel("0000");

    private final JRadioButton tones = new JRadioButton("Tones",true);
    private final JRadioButton stereoPanning = new JRadioButton("Stereo Panning");
    private final JRadioButton stereoPingpong = new JRadioButton("Stereo Pingpong");
    private final JRadioButton fmSweep = new JRadioButton("FM Sweep");
    private final JRadioButton decayPulse = new JRadioButton("Decay Pulse");
    private final JRadioButton echoPulse = new JRadioButton("Echo Pulse");
    private final JRadioButton waWaPulse = new JRadioButton("WaWa Pulse");

    private final JRadioButton listen = new JRadioButton("Listen",true);
    private final JTextField fileName = new JTextField("junk",10);

    public static void main(String[] args){
        new AudioSynth01();
    }

    public AudioSynth01(){
        final JPanel controlButtonPanel = new JPanel();
        controlButtonPanel.setBorder(BorderFactory.createEtchedBorder());

        final JPanel synButtonPanel = new JPanel();
        final ButtonGroup synButtonGroup = new ButtonGroup();
        final JPanel centerPanel = new JPanel();

        final JPanel outputButtonPanel = new JPanel();
        outputButtonPanel.setBorder(BorderFactory.createEtchedBorder());
        final ButtonGroup outputButtonGroup = new ButtonGroup();

        playOrFileBtn.setEnabled(false);

        generateBtn.addActionListener(e -> {
            playOrFileBtn.setEnabled(false);
            new SynGen().getSyntheticData(audioData);
            playOrFileBtn.setEnabled(true);
        });

        playOrFileBtn.addActionListener(e -> playOrFileData());

        controlButtonPanel.add(generateBtn);
        controlButtonPanel.add(playOrFileBtn);
        controlButtonPanel.add(elapsedTimeMeter);

        synButtonGroup.add(tones);
        synButtonGroup.add(stereoPanning);
        synButtonGroup.add(stereoPingpong);
        synButtonGroup.add(fmSweep);
        synButtonGroup.add(decayPulse);
        synButtonGroup.add(echoPulse);
        synButtonGroup.add(waWaPulse);

        synButtonPanel.setLayout(new GridLayout(0,1));
        synButtonPanel.add(tones);
        synButtonPanel.add(stereoPanning);
        synButtonPanel.add(stereoPingpong);
        synButtonPanel.add(fmSweep);
        synButtonPanel.add(decayPulse);
        synButtonPanel.add(echoPulse);
        synButtonPanel.add(waWaPulse);

        centerPanel.add(synButtonPanel);

        outputButtonGroup.add(listen);
        JRadioButton file = new JRadioButton("File");
        outputButtonGroup.add(file);

        outputButtonPanel.add(listen);
        outputButtonPanel.add(file);
        outputButtonPanel.add(fileName);

        getContentPane().add(controlButtonPanel,BorderLayout.NORTH);
        getContentPane().add(centerPanel, BorderLayout.CENTER);
        getContentPane().add(outputButtonPanel, BorderLayout.SOUTH);

        setTitle("Copyright 2003, R.G.Baldwin");
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setSize(250,275);
        setVisible(true);
    }

    private void playOrFileData() {
        try{
            InputStream byteArrayInputStream = new ByteArrayInputStream(audioData);

            boolean bigEndian = true;
            boolean signed = true;
            //Allowable 8000,11025,16000,22050,44100
            int sampleSizeInBits = 16;
            audioFormat = new AudioFormat(sampleRate, sampleSizeInBits, channels, signed, bigEndian);
            audioInputStream = new AudioInputStream(byteArrayInputStream, audioFormat, audioData.length/audioFormat.getFrameSize());
            DataLine.Info dataLineInfo = new DataLine.Info(SourceDataLine.class, audioFormat);

            sourceDataLine = (SourceDataLine) AudioSystem.getLine(dataLineInfo);

            if(listen.isSelected()){
                new ListenThread().start();
            } else{
                generateBtn.setEnabled(false);
                playOrFileBtn.setEnabled(false);
                try{
                    AudioSystem.write(audioInputStream, AudioFileFormat.Type.AU, new File(fileName.getText() + ".au"));
                }catch (Exception e) {
                    e.printStackTrace();
                    System.exit(0);
                }
                generateBtn.setEnabled(true);
                playOrFileBtn.setEnabled(true);
            }
        }catch (Exception e) {
            e.printStackTrace();
            System.exit(0);
        }
    }

    class ListenThread extends Thread{
        byte[] playBuffer = new byte[16384];

        public void run(){
            try{
                generateBtn.setEnabled(false);
                playOrFileBtn.setEnabled(false);

                sourceDataLine.open(audioFormat);
                sourceDataLine.start();

                int cnt;
                long startTime = new Date().getTime();

                while((cnt = audioInputStream.read(playBuffer, 0, playBuffer.length)) != -1){
                    if(cnt > 0){
                        sourceDataLine.write(playBuffer, 0, cnt);
                    }
                }

                sourceDataLine.drain();
                int elapsedTime = (int)(new Date().getTime() - startTime);
                elapsedTimeMeter.setText("" + elapsedTime);

                sourceDataLine.stop();
                sourceDataLine.close();

                generateBtn.setEnabled(true);
                playOrFileBtn.setEnabled(true);
            }catch (Exception e) {
                e.printStackTrace();
                System.exit(0);
            }
        }
    }

    class SynGen{
        ByteBuffer byteBuffer;
        ShortBuffer shortBuffer;
        int byteLength;

        void getSyntheticData(byte[] synDataBuffer){
            byteBuffer = ByteBuffer.wrap(synDataBuffer);
            shortBuffer = byteBuffer.asShortBuffer();

            byteLength = synDataBuffer.length;

            if(tones.isSelected()) tones();
            if(stereoPanning.isSelected())
                stereoPanning();
            if(stereoPingpong.isSelected())
                stereoPingpong();
            if(fmSweep.isSelected()) fmSweep();
            if(decayPulse.isSelected()) decayPulse();
            if(echoPulse.isSelected()) echoPulse();
            if(waWaPulse.isSelected()) waWaPulse();

        }

        void tones(){
            channels = 1;//Java allows 1 or 2
            //Each channel requires two 8-bit bytes per
            // 16-bit sample.
            int bytesPerSamp = 2;
            sampleRate = 16000.0F;
            // Allowable 8000,11025,16000,22050,44100
            int sampLength = byteLength/bytesPerSamp;
            for(int cnt = 0; cnt < sampLength; cnt++){
                double time = cnt/sampleRate;
                double freq = 950.0;//arbitrary frequency
                double sinValue =
                        (Math.sin(2*Math.PI*freq*time));
                shortBuffer.put((short)(16000*sinValue));
            }//end for loop
        }//end method tones
        //-------------------------------------------//

        //This method generates a stereo speaker sweep,
        // starting with a relatively high frequency
        // tone on the left speaker and moving across
        // to a lower frequency tone on the right
        // speaker.
        void stereoPanning(){
            channels = 2;//Java allows 1 or 2
            int bytesPerSamp = 4;//Based on channels
            sampleRate = 16000.0F;
            // Allowable 8000,11025,16000,22050,44100
            int sampLength = byteLength/bytesPerSamp;
            for(int cnt = 0; cnt < sampLength; cnt++){
                //Calculate time-varying gain for each
                // speaker
                double rightGain = 16000.0*cnt/sampLength;
                double leftGain = 16000.0 - rightGain;

                double time = cnt/sampleRate;
                double freq = 600;//An arbitrary frequency
                //Generate data for left speaker
                double sinValue =
                        Math.sin(2*Math.PI*(freq)*time);
                shortBuffer.put(
                        (short)(leftGain*sinValue));
                //Generate data for right speaker
                sinValue =
                        Math.sin(2*Math.PI*(freq*0.8)*time);
                shortBuffer.put(
                        (short)(rightGain*sinValue));
            }//end for loop
        }//end method stereoPanning
        //-------------------------------------------//

        //This method uses stereo to switch a sound
        // back and forth between the left and right
        // speakers at a rate of about eight switches
        // per second.  On my system, this is a much
        // better demonstration of the sound separation
        // between the two speakers than is the
        // demonstration produced by the stereoPanning
        // method.  Note also that because the sounds
        // are at different frequencies, the sound
        // produced is similar to that of U.S.
        // emergency vehicles.

        void stereoPingpong(){
            channels = 2;//Java allows 1 or 2
            int bytesPerSamp = 4;//Based on channels
            sampleRate = 16000.0F;
            // Allowable 8000,11025,16000,22050,44100
            int sampLength = byteLength/bytesPerSamp;
            double leftGain = 0.0;
            double rightGain = 16000.0;
            for(int cnt = 0; cnt < sampLength; cnt++){
                //Calculate time-varying gain for each
                // speaker
                if(cnt % (sampLength/8) == 0){
                    //swap gain values
                    double temp = leftGain;
                    leftGain = rightGain;
                    rightGain = temp;
                }//end if

                double time = cnt/sampleRate;
                double freq = 600;//An arbitrary frequency
                //Generate data for left speaker
                double sinValue =
                        Math.sin(2*Math.PI*(freq)*time);
                shortBuffer.put(
                        (short)(leftGain*sinValue));
                //Generate data for right speaker
                sinValue =
                        Math.sin(2*Math.PI*(freq*0.8)*time);
                shortBuffer.put(
                        (short)(rightGain*sinValue));
            }//end for loop
        }//end stereoPingpong method
        //-------------------------------------------//

        //This method generates a monaural linear
        // frequency sweep from 100 Hz to 1000Hz.
        void fmSweep(){
            channels = 1;//Java allows 1 or 2
            int bytesPerSamp = 2;//Based on channels
            sampleRate = 16000.0F;
            // Allowable 8000,11025,16000,22050,44100
            int sampLength = byteLength/bytesPerSamp;
            double lowFreq = 100.0;
            double highFreq = 1000.0;

            for(int cnt = 0; cnt < sampLength; cnt++){
                double time = cnt/sampleRate;

                double freq = lowFreq +
                        cnt*(highFreq-lowFreq)/sampLength;
                double sinValue =
                        Math.sin(2*Math.PI*freq*time);
                shortBuffer.put((short)(16000*sinValue));
            }//end for loop
        }//end method fmSweep
        //-------------------------------------------//

        //This method generates a monaural triple-
        // frequency pulse that decays in a linear
        // fashion with time.
        void decayPulse(){
            channels = 1;//Java allows 1 or 2
            int bytesPerSamp = 2;//Based on channels
            sampleRate = 16000.0F;
            // Allowable 8000,11025,16000,22050,44100
            int sampLength = byteLength/bytesPerSamp;
            for(int cnt = 0; cnt < sampLength; cnt++){
                //The value of scale controls the rate of
                // decay - large scale, fast decay.
                double scale = 2*cnt;
                if(scale > sampLength) scale = sampLength;
                double gain =
                        16000*(sampLength-scale)/sampLength;
                double time = cnt/sampleRate;
                double freq = 499.0;//an arbitrary freq
                double sinValue =
                        (Math.sin(2*Math.PI*freq*time) +
                                Math.sin(2*Math.PI*(freq/1.8)*time) +
                                Math.sin(2*Math.PI*(freq/1.5)*time))/3.0;
                shortBuffer.put((short)(gain*sinValue));
            }//end for loop
        }//end method decayPulse
        //-------------------------------------------//

        //This method generates a monaural triple-
        // frequency pulse that decays in a linear
        // fashion with time.  However, three echoes
        // can be heard over time with the amplitude
        // of the echoes also decreasing with time.
        void echoPulse(){
            channels = 1;//Java allows 1 or 2
            int bytesPerSamp = 2;//Based on channels
            sampleRate = 16000.0F;
            // Allowable 8000,11025,16000,22050,44100
            int sampLength = byteLength/bytesPerSamp;
            int cnt2 = -8000;
            int cnt3 = -16000;
            int cnt4 = -24000;
            for(int cnt1 = 0; cnt1 < sampLength;
                cnt1++,cnt2++,cnt3++,cnt4++){
                double val = echoPulseHelper(
                        cnt1,sampLength);
                if(cnt2 > 0){
                    val += 0.7 * echoPulseHelper(
                            cnt2,sampLength);
                }//end if
                if(cnt3 > 0){
                    val += 0.49 * echoPulseHelper(
                            cnt3,sampLength);
                }//end if
                if(cnt4 > 0){
                    val += 0.34 * echoPulseHelper(
                            cnt4,sampLength);
                }//end if

                shortBuffer.put((short)val);
            }//end for loop
        }//end method echoPulse
        //-------------------------------------------//

        double echoPulseHelper(int cnt,int sampLength){
            //The value of scale controls the rate of
            // decay - large scale, fast decay.
            double scale = 2*cnt;
            if(scale > sampLength) scale = sampLength;
            double gain =
                    16000*(sampLength-scale)/sampLength;
            double time = cnt/sampleRate;
            double freq = 499.0;//an arbitrary freq
            double sinValue =
                    (Math.sin(2*Math.PI*freq*time) +
                            Math.sin(2*Math.PI*(freq/1.8)*time) +
                            Math.sin(2*Math.PI*(freq/1.5)*time))/3.0;
            return(short)(gain*sinValue);
        }//end echoPulseHelper

        //-------------------------------------------//

        //This method generates a monaural triple-
        // frequency pulse that decays in a linear
        // fashion with time.  However, three echoes
        // can be heard over time with the amplitude
        // of the echoes also decreasing with time.
        //Note that this method is identical to the
        // method named echoPulse, except that the
        // algebraic sign was switched on the amplitude
        // of two of the echoes before adding them to
        // the composite synthetic signal.  This
        // resulted in a difference in the
        // sound.
        void waWaPulse(){
            channels = 1;//Java allows 1 or 2
            int bytesPerSamp = 2;//Based on channels
            sampleRate = 16000.0F;
            // Allowable 8000,11025,16000,22050,44100
            int sampLength = byteLength/bytesPerSamp;
            int cnt2 = -8000;
            int cnt3 = -16000;
            int cnt4 = -24000;
            for(int cnt1 = 0; cnt1 < sampLength;
                cnt1++,cnt2++,cnt3++,cnt4++){
                double val = waWaPulseHelper(
                        cnt1,sampLength);
                if(cnt2 > 0){
                    val += -0.7 * waWaPulseHelper(
                            cnt2,sampLength);
                }//end if
                if(cnt3 > 0){
                    val += 0.49 * waWaPulseHelper(
                            cnt3,sampLength);
                }//end if
                if(cnt4 > 0){
                    val += -0.34 * waWaPulseHelper(
                            cnt4,sampLength);
                }//end if

                shortBuffer.put((short)val);
            }//end for loop
        }//end method waWaPulse
        //-------------------------------------------//

        double waWaPulseHelper(int cnt,int sampLength){
            //The value of scale controls the rate of
            // decay - large scale, fast decay.
            double scale = 2*cnt;
            if(scale > sampLength) scale = sampLength;
            double gain =
                    16000*(sampLength-scale)/sampLength;
            double time = cnt/sampleRate;
            double freq = 499.0;//an arbitrary freq
            double sinValue =
                    (Math.sin(2*Math.PI*freq*time) +
                            Math.sin(2*Math.PI*(freq/1.8)*time) +
                            Math.sin(2*Math.PI*(freq/1.5)*time))/3.0;
            return(short)(gain*sinValue);
        }
    }
}
```

上文介绍了合成声音的制作。合成声音区别于 MIDI 在于，前者更接近于我们日常所说的音效，比如电脑开机、弹出硬盘、错误时的嘟嘟声，而 MIDI，虽然被定义为音乐片段，尽管可以并不代表乐器的声音，但其功能更多的是为制作音乐服务的。

上文介绍的声音有一些特点，比如:

- Tones 的算法使用了单声道，包含三种不同声音频率的混合，两秒单声道。

- Stereo Panning 则包含了一秒的双声道数据，声音从左扬声器逐步传递到右扬声器，并且逐步降低音量。

- Stereo Pingpong 也是一秒双声道数据，声音在两个扬声器之间迅速的切换。

- FM Sweep 包含双耳的单声道声音，开始是 100Hz，线性转变到 1000Hz。

- Decay Pulse 包含两秒的单声道声音，在前一秒从最大值开始衰减到零，在后一秒没有声音。

- Echo Pulse 包含两秒单声道声音，基于 Decay Pluse 算法的脉冲，随着时间的推移，其声音重复出现，不过每次出现，声音都逐渐降低。

- WaWa Pulse 包含两秒的单声道声音，和 Echo Pluse 类似，除了在后半段进行了翻转。

这个程序中涉及的核心在于 byte[] 长度、Frame 长度、Sample 长度的计算，这决定了声音的时间。此外，声音呈现时的函数、声音的频率、响度也是关键所在，其影响声音呈现的特色。

- 采样率：采样率可选  8000， 16000 等，其代表了每秒采样的点的个数。最高可选 44100。

- 样本大小：可选 8 bit 或者 16 bit，其代表了一个样本所使用的数据量。8 bit 即 1 byte，可以包含 1-127 个可选值，这意味着最响的声音是最弱的声音的最高 127 倍，显然这并不理想。还可以使用 16bit，其可选 32767 个值，这一般是理想的选择。注意，这里的样本大小指的是单声道，数据量的大小取决于样本大小 * 样本量 * 通道数，所有的数据均写在 byte[] 数组中，声音的读取取决于解释数据的元信息，即通道数。一般而言，Java 的 short 类型可以正好用来表示 16bit signed PCM 编码的样本大小。

- 通道数：通道的个数，一条 Line 可以包含多条通道，通道数据堆积在一个 byte[] 数组中，按照顺序逐个取出解释。多通道 byte[] 长度不等于 样本长度 * 样本所占 byte 数。注意还有一个叫做帧 Frame 的概念，一个 Frame 定义一个时间片段，其包含多个在同一时间的不同通道的所有数据，这些共享时间的不同通道的样本共同组成了一帧

- 有无符号：Java 默认是 Signed 数据，这也是默认值。

- 内存排序：声音序列化的数据涉及其在内存中排序的方式，Java 默认使用 Big-endian，这区别于 C 在 PC 上的 little-endian。

在程序中，我们创建了一个用于存放数据的 byte[] 数组，通过算法生成数据并且填充到此数组中。这需要一定的工作，首先，byte[] 数组被用来构造 ByteBuffer，之后，ByteBuffer 被转换成 ShortBuffer，Short 类型用于 16bit 的 PCM 正好合适。之后，根据采样率、通道数可以计算播放若干秒需要的数组长度，比如双声道的 16000 采样率，播放 4 s 需要数组长度为：16000 * 2 * 2 * 4，16000 是帧数，双声道需要的样本量为 16000 * 2，因为每个样本占据 2 个 byte，因此需要 16000 * 2 * 2 的数组播放一秒的数据。

现在确定了 byte[] 数组长度后，在算法中，根据需要的通道数（每个样本对应的帧数）和样本采样率、每个帧占据的大小计算样本长度。之后为每个节点通过正弦函数计算结果，乘以响度（这里乘以 16000 的原因是因为 Short 数据长度的一半是 16000）。需要注意，这里需要将时间/周期转换成弧度，因此，使用 2 * Pi * Time 即可。之后通过正弦计算得到结果，存入数组即可。注意这里还有一个纯音频率的概念，频率指的是一个周期的时间，这里需要乘以 Freq 才能得到一个时刻的周期弧度值。

# Java Sound In Scala

## 播放、录制、纯音和节律刺激接口

因为本来是要用 Java Sound 生成纯音来着，因此，在 Scala 中，我定义了两个接口，然后提供了一个简单的用于播放、录制、生成纯音和间歇纯音的实现：

AudioFunction 接口用于提供录制、播放声音的同步阻塞 API 定义。

```scala
import java.io.{ByteArrayInputStream, ByteArrayOutputStream, File}
import javax.sound.sampled.{AudioFileFormat, AudioFormat, AudioInputStream, AudioSystem}

trait AudioFunction {
  var stopPlayMarker: Boolean
  var stopRecordMarker: Boolean
  def playSound(audioInputStream: AudioInputStream): Unit
  def playSound(file: File): Unit = {
    val audioInputStream = AudioSystem.getAudioInputStream(file)
    playSound(audioInputStream)
  }
  def playSound(implicit audioMaker: AudioMaker, audioFormat: AudioFormat, millSeconds: Int, freq: Double, volume: Double): Unit = {
    val bytes = audioMaker.makeTone(audioFormat, millSeconds, freq, volume)
    val stream = new AudioInputStream(new ByteArrayInputStream(bytes), audioFormat, bytes.length / audioFormat.getFrameSize)
    playSound(stream)
  }
  def recordToFile(audioFormat: AudioFormat, audioFileFormatType: AudioFileFormat.Type, file: File): Unit = {
    val byteArrayOutputStream = new ByteArrayOutputStream()
    recordToStream(audioFormat, byteArrayOutputStream)
    val data = byteArrayOutputStream.toByteArray
    val inpStream = new ByteArrayInputStream(data)
    AudioSystem.write(
      new AudioInputStream(inpStream, audioFormat, data.length/audioFormat.getFrameSize),
      audioFileFormatType , file)
    inpStream.close()
    byteArrayOutputStream.close()
  }
  def recordToStream(audioFormat: AudioFormat, byteArrayOutputStream: ByteArrayOutputStream): Unit
}
```

AudioMaker 接口用于提供制造纯音的 API 定义：

```scala
import javax.sound.sampled.AudioFormat

trait AudioMaker {
  def makeTone(audioFormat: AudioFormat, millSeconds: Int, freq: Double, volume: Double): Array[Byte]
}
```

ToneUtils 接口提供了用于播放间隔节律、无节律纯音的 API 定义：

```java
import javax.sound.sampled.LineUnavailableException;

public interface ToneUtils {
    void playForDuration(double soundFrequency, int soundDurationMs, int allDurationMS, int spaceMS) throws InterruptedException, LineUnavailableException;
    void playForDuration(double soundFrequency, int soundDurationMs,
                         int allDurationMS, int randomSpaceFrom, int randomSpaceTo) throws LineUnavailableException, InterruptedException;
}
```

## 基于线程休眠的节律声音实现

下面使用 Java Sound 提供了 ToneUtils 的基于线程休眠的实现：

```java
public class SimpleLongToneUtilsImpl implements ToneUtils {

    private float sampleRate = 16000.0F;
    //Allowable 8000,11025,16000,22050,44100
    private int sampleSizeInBits = 16;
    //Allowable 8,16
    private int channels = 1;
    //Allowable 1,2
    private boolean signed = true;
    private boolean bigEndian = true;

    private AudioFormat audioFormat =  //Get the required audio format
            new AudioFormat(sampleRate, sampleSizeInBits, channels, signed, bigEndian);

    //Get info on the required data line
    private DataLine.Info dataLineInfo = new DataLine.Info(SourceDataLine.class, audioFormat);

    private SourceDataLine sourceDataLine = (SourceDataLine) AudioSystem.getLine(dataLineInfo);

    public SimpleLongToneUtilsImpl() throws LineUnavailableException {
    }

    //一种缓冲器，用于以每秒16000采样量存储16位样本的两秒单耳和一秒立体声数据
    //控制时间
    private byte[] getBuffer(double soundMs) {
        return new byte[Math.toIntExact(Math.round(32 * soundMs))];
    }

    private void putSyntheticData(byte[] synDataBuffer, double freq){
        ByteBuffer byteBuffer;
        ShortBuffer shortBuffer;
        int byteLength;
        //Prepare the ByteBuffer and the shortBuffer
        // for use
        byteBuffer = ByteBuffer.wrap(synDataBuffer);
        shortBuffer = byteBuffer.asShortBuffer();

        byteLength = synDataBuffer.length;

        channels = 1;//Java allows 1 or 2
        //Each channel requires two 8-bit bytes per
        // 16-bit sample.
        int bytesPerSamp = 2;
        sampleRate = 16000.0F;
        // Allowable 8000,11025,16000,22050,44100
        int sampLength = byteLength/bytesPerSamp;
        for(int cnt = 0; cnt < sampLength; cnt++){
            double time = cnt/sampleRate;
            double sinValue = (Math.sin(2*Math.PI*freq*time));
            shortBuffer.put((short)(16000*sinValue));
        }
    }

    private synchronized void justPlay(byte[] audioData, int allDurationMS, int spaceMS,
                                       int randomSpaceFrom, int randomSpaceTo)
            throws LineUnavailableException, InterruptedException {
        //Open and start the SourceDataLine
        sourceDataLine.open(audioFormat);
        sourceDataLine.start();
        Random random = new Random();
        long start = System.nanoTime()/1000000;
        while((System.nanoTime()/1000000 - start) < allDurationMS) {
            sourceDataLine.write(audioData, 0, audioData.length);
            sourceDataLine.drain();
            if (spaceMS == -1) {
                int nextSpaceMs = random.nextInt(randomSpaceTo - randomSpaceFrom) + randomSpaceFrom;
                TimeUnit.MILLISECONDS.sleep(nextSpaceMs);
            } else {
                TimeUnit.MILLISECONDS.sleep(spaceMS);
            }
        }
        //Block and wait for internal buffer of the
        // SourceDataLine to become empty.
        sourceDataLine.drain();
        sourceDataLine.stop();
        sourceDataLine.close();
    }

    @Override
    public void playForDuration(double soundFrequency, int soundDurationMs, int allDurationMS, int spaceMS) {
        byte[] buffer = this.getBuffer(soundDurationMs);
        this.putSyntheticData(buffer, soundFrequency);
        //byte[] sinWaveBuffer = createSinWaveBuffer(soundFrequency, soundDurationMs);
        try {
            this.justPlay(buffer, allDurationMS, spaceMS, 0,0);
        } catch (LineUnavailableException | InterruptedException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void playForDuration(double soundFrequency, int soundDurationMs, int allDurationMS,
                                int randomSpaceFrom, int randomSpaceTo) {
        byte[] buffer = this.getBuffer(soundDurationMs);
        this.putSyntheticData(buffer, soundFrequency);
        try {
            this.justPlay(buffer, allDurationMS, -1, randomSpaceFrom, randomSpaceTo);
        } catch (LineUnavailableException | InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

## 基于纯音数据生成的节律声音实现

而下面的实现则使用了另外的解决方案，这里的实现并不完美（使用了取模），最好的方式是使用 while 循环拼接声音 + 无声音刺激，直到数组长度等于我们预期的满足某种 AudioFormat 的 Array[Byte] 的长度（通道 * 每样本占据的 byte 数 * 每秒的采样数 * 采样的秒数），对于非规律声音而言，使用的方法是先生成所需声音，然后创建数据，进行拼接，怎么着也都相同的麻烦。

```scala
import java.io.{ByteArrayInputStream, ByteArrayOutputStream}
import java.nio.ByteBuffer
import javax.sound.sampled.{AudioFormat, AudioInputStream, AudioSystem}
import scala.collection.mutable
import scala.util.Random

//函数接受的值都是值引用，如果需要在一个线程中的函数根据某个 Marker 进行变化，那么需要使用一个类
//并且将其 Marker 设置为一个类变量，然后这个函数在类空间中访问此类变量以进行实时处理。
class SimpleAudioFunctionMakerToneUtilsImpl extends AudioFunction with AudioMaker with ToneUtils {

  override var stopPlayMarker: Boolean = false

  override var stopRecordMarker: Boolean = false

  override def playSound(audioInputStream: AudioInputStream): Unit = {
    val line = AudioSystem.getSourceDataLine(audioInputStream.getFormat)
    line.open()
    line.start()
    val buffer = new Array[Byte](10000)
    var cnt = 0
    while (cnt != -1 && !stopPlayMarker) {
      cnt = audioInputStream.read(buffer)
      if (cnt > 0) line.write(buffer, 0, cnt)
    }
    line.drain() //当 Line 被覆盖前，其会先试图播放，因此，只用在最后 drain 即可
    line.stop()
    line.close() //Line close 会自动处理 AudioInputStream 的关闭吗？
    audioInputStream.close()
    stopPlayMarker = false
  }

  override def recordToStream(audioFormat: AudioFormat, byteArrayOutputStream: ByteArrayOutputStream): Unit = {
    val line = AudioSystem.getTargetDataLine(audioFormat)
    line.open()
    line.start()
    val buffer = new Array[Byte](10000)
    var cnt = 0
    while (!stopRecordMarker) {
      cnt = line.read(buffer, 0, buffer.length)
      if (cnt > 0) byteArrayOutputStream.write(buffer, 0, cnt)
    }
    val array = byteArrayOutputStream.toByteArray
    val byteArrayInputStream = new ByteArrayInputStream(array)
    byteArrayInputStream.close()
    //byteArrayOutputStream.close()
    line.close()
    stopRecordMarker = false
  }

  override def makeTone(audioFormat: AudioFormat, millSeconds: Int, freq: Double, volume: Double = 16000.0): Array[Byte] = {
    val channels = audioFormat.getChannels
    val rate = audioFormat.getSampleRate.toInt
    val bytesPerSamp = audioFormat.getSampleSizeInBits match {
      case 16 => 2
      case 8 => 1
      case _ => throw new RuntimeException("AudioFormat SampleSizeInBits 参数不对")
    }
    val arraySize = channels match {
      case i => (i * rate * (millSeconds * 1.0 / 1000) * bytesPerSamp).toInt
    }
    val array = new Array[Byte](arraySize)
    val byteBuffer = ByteBuffer.wrap(array)
    val shortBuffer = byteBuffer.asShortBuffer()
    for (i <- 0 until arraySize/bytesPerSamp) {
      val time = i * 1.0/rate
      val sinValue = Math.sin(2 * Math.PI * freq * time)
      shortBuffer.put((volume * sinValue).asInstanceOf[Short])
    }
    array
  }

  var toneSampleRate: Float = 16 * 1000

  override def playForDuration(soundFrequency: Double, soundDurationMs: Int, allDurationMS: Int, spaceMS: Int): Unit = {
    val format = new AudioFormat(toneSampleRate,16,1,true,true)
    val toneData = makeTone(format, soundDurationMs, soundFrequency)
    val emptyData = makeTone(format, spaceMS, soundFrequency, 0)
    val periodMs = soundDurationMs + spaceMS
    val periodData = toneData ++ emptyData
    val period = allDurationMS * 1.0 / periodMs
    val totalLength = (periodData.length * period).toInt
    val array = new Array[Byte](totalLength)
    for (index <- 0 until totalLength) {
      array(index) = periodData(index % periodData.length)
    }
    val byteInStream = new ByteArrayInputStream(array)
    val inpStream = new AudioInputStream(byteInStream, format, array.length/format.getFrameSize)
    playSound(inpStream)
    inpStream.close()
    byteInStream.close()
  }

  override def playForDuration(soundFrequency: Double, soundDurationMs: Int, allDurationMS: Int, randomSpaceFrom: Int, randomSpaceTo: Int): Unit = {
    val format = new AudioFormat(toneSampleRate,16,1,true,true)
    val times = getTimeArray(soundDurationMs, randomSpaceFrom, randomSpaceTo, allDurationMS)
    val allData = mutable.Buffer[Byte]()
    val isSound: Int => Boolean = _ == soundDurationMs
    times.foreach(i => {
      if (isSound(i)) allData ++= makeTone(format, i, soundFrequency)
      else allData ++= makeTone(format, i, soundFrequency, 0)
    })
    val byteInStream = new ByteArrayInputStream(allData.toArray)
    val inpStream = new AudioInputStream(byteInStream, format, allData.length/format.getFrameSize)
    playSound(inpStream)
    inpStream.close()
    byteInStream.close()
  }

  def getTimeArray(sound: Int, randomFrom: Int, randomTo: Int, total: Int): Array[Int] = {
    def getIt: mutable.Buffer[Int] = {
      val random = new Random()
      var currentLength = 0
      val array = mutable.Buffer[Int]()
      var lastRand = -1
      while (currentLength < total) {
        val rand = random.nextInt(randomTo - randomFrom) + randomFrom
        array.append(sound)
        array.append(rand)
        currentLength += sound
        currentLength += rand
        lastRand = rand
      }
      val diff = lastRand - (currentLength - total)
      if (diff > randomFrom) {//如果一个可以处理
      val rev = array.reverse
        rev(0) = diff
        if (rev.sum != total) throw new RuntimeException("总数不对")
        rev.reverse
      } else {//否则使用最后两个
        throw new RuntimeException("总数不对2")
      }
    }
    var res = mutable.Buffer[Int]()
    var notOk = true
    while (notOk) {
      try {
        res = getIt
        notOk = false
      } catch {
        case _: Throwable =>
      }
    }
    res.toArray
  }
}
```

虽然如此，这种方式的优点在于，可以完全控制声音的表现方式，可以将声音和某些关键时间点对齐，对声音的操纵更强。
