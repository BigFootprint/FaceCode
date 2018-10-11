---
title: Android开发One Piece
date: 2016-02-27 11:29:31
tags: [合集]
categories: Android
---

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/onepiece2.png" height="188" alt="One Piece"/></div>

## 工具相关
### TraceView
1. [Android自定义Lint实践](http://tech.meituan.com/android_custom_lint.html)。看上去很强大的样子，可以用于定制一些工具。
2. [Android Studio 小技巧合集](http://laobie.github.io/android/2016/02/14/android-studio-tips.html)。调试器技巧分类下很有用。
3. [Android分享：代码混淆那些事](https://segmentfault.com/a/1190000004461614)。跟Proguard相关的基本知识，也有一些不错的文章链接。
   <!--more-->
## 优化
1. [TRIM：提升磁盘性能，缓解Android卡顿](https://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=404234343&idx=1&sn=b297b01ee7c656b900417f14a2a0ccae&scene=1&srcid=0303KKfAyLlCyDJT1Sutn26i&key=710a5d99946419d9c72026bcf5c728d9f9fb967dc032ffe2a60669550d6677ace7cadcf6453d69de28540dcbdceefb49&ascene=0&uin=MTYzMjY2MTE1&devicetype=iMac+MacBookPro10%2C1+OSX+OSX+10.11.3+build(15D21)&version=11020201&pass_ticket=VbRCYWXa6oXQkbEbGqIqO1z%2BRQ8XRpmC8IJJVBb2EJ8%3D)。这篇文章提到了Android系统上长期读写文件造成的磁盘碎片化对Android系统流畅度的影响。

>Android手机大多采用NAND Flash架构的闪存卡。虽然 NAND Flash 的优点多多，但是为了延长驱动器的寿命，它的读写操作均是以 Page 为单位进行的，但擦除操作却是按 Block 为单位进行的。这会造成“写入放大”（Write Amplification）问题。这一点可以使用TRIM技术解决:
>
>>TRIM 是一条 ATA 指令，由操作系统发送给闪存主控制器，告诉它哪些数据占的地址是“无效”的。在 TRIM 的帮助下，闪存主控制器就可以提前知道哪些 Page 是“无效”的，便可以在适当的时机做出优化，从而改善性能。
>
>后面用实验证明了这种猜想，以及Android系统确实使用该技术对系统做出了优化，结论如下:
>
>1. 在 TRIM 无效的情况下，长期使用 SD 卡，磁盘写入速度会受到明显影响;
>2. TRIM 对因闲置数据块造成的 I/O 性能下降有一定的恢复作用;
>3. 大量的读写操作对 SD 卡造成了一定量的不可恢复的损耗;

2. [如何做到将apk大小减少6M](http://blog.csdn.net/UsherFor/article/details/46827587)。讲了一组与缩小APK大小相关的优化措施，挺全面。
3. [PNG图片压缩对比分析](https://mp.weixin.qq.com/s?__biz=MzI1NjEwMTM4OA==&mid=2651232233&idx=1&sn=03d9858ac451f2768b804d2604a8e12e)。调研了目前常用的PNG压缩工具和第三方库，在决定使用pngquant的基础上开发Gradle插件，完成PNG的自动打包压缩，非常具有实践性。

## 知识整理
1. [Android Bitmap面面观](https://mp.weixin.qq.com/s?__biz=MzA4MjU5NTY0NA==&mid=404530070&idx=1&sn=e2580b69d6ec73dabf8160216aa6702a&scene=1&srcid=0322dGvv5B4miLg3vt0q2rK2&key=710a5d99946419d9e5ab07eaa09e546952524b770e8a63c46c223b7588209d3a542fe4a0827571ac73b03cb78b404289&ascene=0&uin=MTYzMjY2MTE1&devicetype=iMac+MacBookPro10%2C1+OSX+OSX+10.11.4+build(15E65)&version=11020201&pass_ticket=Sqb%2BpOLTrZ0wixoScJ8flYxFV4tOFnBa7NFos1uIG8U%3D)

2. [Android MotionEvent详解](http://ztelur.github.io/2016/03/16/Android-MotionEvent%E8%AF%A6%E8%A7%A3/)。里面有着对Index、Pointer、Id的讨论。

3. [Android M 新的运行时权限开发者需要知道的一切](http://jijiaxin89.com/2015/08/30/Android-s-Runtime-Permission/)

4. [你应该知道的那些Android小经验](http://mp.weixin.qq.com/s?__biz=MzA4MjU5NTY0NA==&mid=404388098&idx=1&sn=8bbbba7692dca68cdda2212dec4d86c0&scene=21#wechat_redirect)

5. [Android 开发绕不过的坑：你的 Bitmap 究竟占多大内存？](http://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=403263974&idx=1&sn=b0315addbc47f3c38e65d9c633a12cd6&scene=4#wechat_redirect)。详细探究了关于图片放置在不同分辨率文件夹下面生成Bitmap后所占内存大小的计算方法，清楚明白。

   > 整理出来的公式分为两部分：
   >
   > 1）内存图片实际的宽高计算：内存宽/高 = (int)(文件实际宽/高 * 屏幕分辨率 / 文件夹分辨率 + 0.5)
   >
   > 2）内存占用计算：内存大小 = 内存宽 * 内存高 * 每个像素占用值。
   >
   > 每个像素占用值可以参见：
   >
   > ```c++
   > static int SkColorTypeBytesPerPixel(SkColorType ct) {
   >    static const uint8_t gSize[] = {
   >     0,  // Unknown
   >     1,  // Alpha_8
   >     2,  // RGB_565
   >     2,  // ARGB_4444
   >     4,  // RGBA_8888
   >     4,  // BGRA_8888
   >     1,  // kIndex_8
   >   };
   > ```
   >
   > 粗略的看，__图片占用内存大小与解析格式、存放目录以及手机屏幕密度相关__。

6. 待补充。

## 黑科技
1. [一种为 Apk 动态写入信息的方案](https://mp.weixin.qq.com/s?__biz=MzAxNjI3MDkzOQ==&mid=405919721&idx=1&sn=fdad21c0bc74d90e66443d488e8cdc8f&scene=0&key=710a5d99946419d977ba9c54e81b348f298b6edd533df2290bf866f6dc83684af4956d4748ed17fc4b79f57bd5c02431&ascene=0&uin=MTYzMjY2MTE1&devicetype=iMac+MacBookPro10%2C1+OSX+OSX+10.11.3+build(15D21)&version=11020201&pass_ticket=obnYCpEe4OrQ6yNew2Dy2ncIZX%2B8wffXKUYSyIB2KVY%3D)
   挺有意思的手法，这篇文章基于分享场景介绍了一种动态为APK写入信息的方法——从Zip文件格式入手，在文件尾部插入一小段信息，利用这段信息完成分享场景的完整性。赞！
2. [如何精确地测量java对象的大小-底层instrument API](http://blog.csdn.net/xieyuooo/article/details/7068216)和[java.lang.Instrument 代理Agent使用](http://my.oschina.net/xianggao/blog/362495)。主要介绍了`Agent`的存在以及Instrument测量Java对象大小的使用方式。