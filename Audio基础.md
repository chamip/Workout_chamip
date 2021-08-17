### Audio基础

---

**Audio编码分类：**

无压缩：

pcm

音乐编码：

有损：mp3/aac/wma/ogg...

无损：wav/flac/alac/lpac

语音编码：

ARM-WB/AMR-NB/GSM/LPC/SPEEX/CELP/G.7xx/ADPCM...



音频：20-20kHz

采样频率

采样深度：按一定采样频率经数码脉冲取样，每一个离散脉冲信号被以一定的量化精度量化成一串二进制编码流，其位数就是采样深度，也叫量化精度。

Nyquist-Shannon采样定律：数模转换中，当采样频率fs.max大于信号中最高频率fmax2倍时，采样之后的数字信号完整保留了原始信号中的信息。

声道

---

#### Audio模块重点解决的问题：

Playback

Recording

Device control

Volume control

---

#### Audio框架

早期:OSS（Open Sound System）

新：ALSA（Advanced Linux Sound Architecture）


