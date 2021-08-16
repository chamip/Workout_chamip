### MediaCodec/音视频编解码
---

Java定义：

```java
public final class MediaCodec extends Object
  Java.lang.Object
  ->android.media.MediaCodec
```

MediaCodec类可用于访问Android底层的媒体编解码器，也就是，编码器/解码器组件。它是Android底层多媒体支持基本架构的一部分。一个编解码器通过处理输入数据来产生输出数据。MediaCode采用**异步方式**处理数据，并且使用了一组输入输出缓存（buffer）。当请求或接收到一个空的输入缓存（buffer），向其中填充满数据并将它传递给编解码器处理。编解码器处理完这些数据并将处理结果输出至一个空的输出缓存（buffer）中。最终，你请求或接收到一个填充了数据的输出缓存（buffer）,使用完其中的数据，并将其释放回编解码器。

---

编解码器处理三种类型的数据：压缩数据、原始音频数据、原始视频数据。

**压缩缓存/Compressed Buffers**

输入缓存-buffers（用于解码器）和输出缓存-buffers（用于编码器）中包含由媒体格式类型决定的压缩数据。对于视频类型是单个压缩的视频帧。对于音频数据通常是单个可访问单元(一个编码的音频片段，通常包含几毫秒的遵循特定格式类型的音频数据)，但这种要求也不是十分严格，一个缓存-buffer可能包含多个可访问的音频单元。在这两种情况下，缓存不会在任意的字节边界上开始或结束，而是在帧或可访问单元的边界上开始或结束。

**原始音频缓存/Raw Audio Buffers**

原始的音频数据缓存（buffers）包含整个PCM（脉冲编码调制）音频数据帧，这是每一个通道按照通道顺序的一个样本。每一个样本是一个按照本机字节顺序的16位带符号整数（16-bit signed integer in native byte order）。

```java
short[] getSamplesForChannel(MediaCodec codec, int bufferId, int channelIx) {
 　　ByteBuffer outputBuffer = codec.getOutputBuffer(bufferId);
 　　MediaFormat format = codec.getOutputFormat(bufferId);
 　　ShortBuffer samples = outputBuffer.order(ByteOrder.nativeOrder()).asShortBuffer();
 　　int numChannels = formet.getInteger(MediaFormat.KEY_CHANNEL_COUNT);
 　　if (channelIx < 0 || channelIx >= numChannels) {
 　　　　return null;
 　　}
 　　short[] res = new short[samples.remaining() / numChannels];
 　　for (int i = 0; i < res.length; ++i) {
 　　　　res[i] = samples.get(i * numChannels + channelIx);
 　　}
 　　return res;
 }
```

**原始视频缓存/Raw Video Buffers**

视频缓存-buffers根据它们的颜色格式（color format）进行展现。视频编解码器支持3种颜色格式：本地原始视频格式/灵活的YUV格式/其他格式。从Android5.1开始所有的视频编解码器都支持灵活的YUV格式。

---

数据处理方式在不同的API版本下有三种：使用缓存的异步处理方式/使用缓存的同步处理方式/使用缓存数组的同步处理方式。

从Android 5.0开始，首选的方法是调用configure之前通过设置回调异步地处理数据。异步模式稍微改变了状态转换方式，因为你必须在调用flush()方法后再调用start()方法才能使编解码器的状态转换为Running子状态并开始接收输入buffers。同样，初始调用start方法将编解码器的状态直接变化为Running子状态并通过回调方法开始传递可用的输入buufers。

![](https://imagestypora.oss-cn-hangzhou.aliyuncs.com/imagestypora.oss-cn-hangzhou.aliyuncs.com缓存.png)

**使用缓存的同步处理方式**

从Android5.0开始，即使在同步模式下使用编解码器你应该通过getInput/OutputBuffer(int) 和/或 getInput/OutputImage(int) 方法检索输入和输出buffers。这允许通过框架进行某些优化，例如，在处理动态内容过程中。如果你调用getInput/OutputBuffers()方法这种优化是不可用的。不要同时混淆使用缓存和缓存数组的方法。而且仅仅在调用start()方法后或取出一个值为INFO_OUTPUT_FORMAT_CHANGED的输出buffer ID后你才可以直接调用getInput/OutputBuffers方法。

**使用缓存数组的同步处理方式**

在Android 4.4之前，一组输入或输出buffers使用ByteBuffer[]数组表示。在成功调用了start()方法后，通过调用getInput/OutputBuffers()方法检索buffer数组。在这些数组中使用buffer的ID-s（非负数）作为索引，如下面的演示示例中，注意数组大小和系统使用的输入和输出buffers的数量之间并没有固定的关系，尽管这个数组提供了上限边界。