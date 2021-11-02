---
layout: post
title: 纯音刺激 - 使用 Java 和 StackOverflow 实现 😂
categories:
  - Java
  - Stack Overflow
---

> 本文介绍了一种使用 Java Sound API 实现的纯音刺激，主要利用了三角函数，根据采样率和录音时长S确定数组长度，然后，对于数组中的每个位置填充 Math.sin( 2 * Pi * t / p ) * 127，其中 t 为数组下标，p 为周期，其计算方法为采样率/声音频率Hz。

使用方法为：AudioFormat 获取 AudioSystem 默认的 SourceDataLine，之后 write 上面生成的数组，之后 drain 和 close line 即可实现播放。

```java
import javax.sound.sampled.AudioFormat;
import javax.sound.sampled.AudioSystem;
import javax.sound.sampled.LineUnavailableException;
import javax.sound.sampled.SourceDataLine;
import java.util.Random;
import java.util.concurrent.TimeUnit;

/**
 * @implNote
 * 2019-04-02 WaveUtil 用来提供纯音，playForDuration 用来生成连续或者随机的指定声音频率数、声音时长、声音总时长、声音间隔的纯音
 */
public class WaveUtil {

    public static final int SAMPLE_RATE = 16 * 1024;

    private static byte[] createSinWaveBuffer(double freq, int ms) {
        int samples = (int)((ms * SAMPLE_RATE) / 1000);
        byte[] output = new byte[samples];
        double period = (double)SAMPLE_RATE / freq;
        for (int i = 0; i < output.length; i++) {
            double angle = 2.0 * Math.PI * i / period;
            output[i] = (byte)(Math.sin(angle) * 127f);  }
        return output;
    }

    public static void playForDuration(int soundFrequency, int soundDurationMs, int allDurationMS, int spaceMS)
            throws InterruptedException, LineUnavailableException {
        final AudioFormat af = new AudioFormat(SAMPLE_RATE, 8, 1, true, true);
        SourceDataLine line = AudioSystem.getSourceDataLine(af);
        line.open(af, SAMPLE_RATE);
        line.start();
        long startTime = System.nanoTime();
        double currentTime = 0;
        while (currentTime < allDurationMS) {
            byte [] toneBuffer = 
            createSinWaveBuffer(soundFrequency, soundDurationMs);
            line.write(toneBuffer, 0, toneBuffer.length);
            TimeUnit.MILLISECONDS.sleep(spaceMS);
            currentTime = (System.nanoTime() - startTime)/1000_000.0;
        }
        line.drain();
        line.close();
    }

    public static void playForDuration(int soundFrequency, int soundDurationMs,
                                         int allDurationMS, int randomSpaceFrom, int randomSpaceTo)
            throws LineUnavailableException, InterruptedException {
        final AudioFormat af = new AudioFormat(SAMPLE_RATE, 8, 1, true, true);
        SourceDataLine line = AudioSystem.getSourceDataLine(af);
        line.open(af, SAMPLE_RATE);
        line.start();
        long startTime = System.nanoTime();
        double currentTime = 0;
        Random random = new Random();
        while (currentTime < allDurationMS) {
            byte [] toneBuffer = 
            createSinWaveBuffer(soundFrequency, soundDurationMs);
            line.write(toneBuffer, 0, toneBuffer.length);
            int nextSpaceMs = 
            random.nextInt(randomSpaceTo - randomSpaceFrom) + randomSpaceFrom;
            TimeUnit.MILLISECONDS.sleep(nextSpaceMs);
            currentTime = (System.nanoTime() - startTime)/1000_000.0;
        }
        line.drain();
        line.close();
    }

}
```

Sound API 使用的时候，都是照着 Oracle 给的 Demo 摸索的，如果 Demo 没有，那么就百度... 想要解决这个问题的时候，Google 访问路径挂了，百度了半天，CSDN 上各路大神写的东西，实在是... 惨不忍睹，偶尔有几个是涉及 Android 的，并且写的也是语焉不详。

对于小众如 Scala with JavaFx 这种东西，在 Scala 官方论坛上都能找到很详细的解决方法和讨论，而 Java 作为快 20 年的语言，在中文社区竟然难以找到一个使用官方 API 实现的纯音刺激... 这实在是一种悲哀。

这篇文章是参照 StackOverflow 上的一个帖子，写的很棒，使用 Bing 搜的，很显然，不知道为什么，百度难以搜索到 StackOverflow 的问答，或者说，百度很难搜索到本领域几乎任何有用的回答。

参照的文章见此：https://stackoverflow.com/questions/8632104/sine-wave-sound-generator-in-java