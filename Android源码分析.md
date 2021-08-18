### Android源码分析

---

![](https://imagestypora.oss-cn-hangzhou.aliyuncs.com/imagestypora.oss-cn-hangzhou.aliyuncs.comandroid架构.jpg)

Application层就是一个应用程序；Framework提供一个Java的运行环境以及对功能实现的封装；Runtime/ART是一个java虚拟机，因为Android上层是Java，需要再编译一次成为更低级的语言；Libraries层是功能实现区，有很多函数库，包括多媒体编解码/浏览器渲染/数据库实现等；Kernel层负责和硬件接触，通过它来操作硬件。

#### 阅读源码的思路

1. **Android源码分为功能实现上的纵向，以及功能拓展上的横向。**比如研究音频系统的实现原理，*纵向*：需要从一个音乐的开始播放追踪，然后发现解码库的调用，共享内存的创建和使用，路由的切换，音频输入设备的开启，音频流的开始。比如研究音频系统包括哪些内容，*横向*：通过Framework的接口，发现音频系统主要包括放音，录音，路由切换，音效处理等。
2. **Android的功能模块绝大部分是C/S架构**。Server是需要攻破的地方，**Libraries是精髓**，理解它之后才能向HAL和Kernel学习。
3. **Android的底层是Linux Kernel**。需要熟悉Kernel的基础协议，HAL是对kernel层的封装，方便各硬件接口统一，换硬件不用到Libraries里面改代码，在HAL里改就行。