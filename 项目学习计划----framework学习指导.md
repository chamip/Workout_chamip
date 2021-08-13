### 项目学习计划----framework学习指导

---

##### 2021.08.10 按照（共通->video->audio->graphics）顺序学习相关知识，每个领域需要在学习之后输出***总结文档***，包括整个大模块的***类结构图***、主要接口的***时序图***。

---

##### AMS四大组件：Activity、Service、Provider、BroadcastReceiver

### Service

---

##### 2021.08.12

多媒体播放组件：

1. MediaPlayer：播放控制
2. MediaCodec：音视频编解码
3. OMX：多媒体部分采用的编解码标准
4. StageFright：MediaPlayerService层框架，提供API给Java/JNI调用
5. AudioTrack：音频播放

---

##### 2021.08.13

阅读Mediaplayer源码。

整理文档。

---

---

### 智能指针

智能指针不是指针，其实质就是引用计数。它是一种数据结构，是安全回收内存的一种机制。

#### 原理

实现方式就是通过智能指针对new出来的实际对象的引用进行计数，自动对new出来的对象进行生命周期的管理和自动销毁。

![](https://imagestypora.oss-cn-hangzhou.aliyuncs.com/imagestypora.oss-cn-hangzhou.aliyuncs.com智能指针时序图.png)

#### Android智能指针原理

- 所有对象从RefBase派生
- mRefs管理引用次数
- sp是强引用
- wp是弱引用
- ![智能指针](https://imagestypora.oss-cn-hangzhou.aliyuncs.com/imagestypora.oss-cn-hangzhou.aliyuncs.com智能指针.png)

#### 实际对象的引用状态

![](https://imagestypora.oss-cn-hangzhou.aliyuncs.com/imagestypora.oss-cn-hangzhou.aliyuncs.com实际对象的引用状态.png)

