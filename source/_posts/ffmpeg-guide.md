---
title: FFmpeg命令行转压视频
date: 2016-10-14 14:45:38
categories:
- 视频压缩
tags:
- 视频压缩
---
FFmpeg是一套可以用来记录、转换数字音频、视频，并能将其转化为流的开源计算机程序。采用LGPL或GPL许可证。它提供了录制、转换以及流化音视频的完整解决方案。它包含了非常先进的音频/视频编解码库libavcodec，为了保证高可移植性和编解码质量，libavcodec里很多code都是从头开发的。
![ffmpeg](http://wx2.sinaimg.cn/mw690/78d85414ly1fkhvok89i4j20m806wwif.jpg)
<!-- more -->
## 安装
直接到官网[http://ffmpeg.org/download.html](http://ffmpeg.org/download.html)根据系统下载对应的版本即可，建议直接下载static版，开箱即用，对于mac用户而言，也可以直接brew安装，命令行如下：

```bash
brew install ffmpeg --with-faac --with-fdk-aac --with-ffplay --with-fontconfig --with-freetype --with-libass --with-libbluray --with-libcaca --with-libsoxr --with-libquvi --with-frei0r --with-libvidstab --with-libvorbis --with-libvpx --with-opencore-amr --with-openjpeg --with-openssl --with-opus --with-rtmpdump --with-schroedinger --with-speex --with-theroa --with-tools --with-x265
```

## 使用
安装好之后就可以使用ffmpeg命令来压制你的视频文件了，下面为一个简单的命令行使用，对于压缩效果不满意的可以根据ffmpeg参数进行调整

```bash
ffmpeg -i your_video -vcodec libx264 -preset fast -crf 20 -y -vf "scale=1920:-1" -acodec libmp3lame -ab 128k your_output
```

对该命令的常用参数介绍如下

| 命令行参数 | 意义 | 默认值 |
|--------|---------|-------|
| -i | 输入文件 | |
| -vcodec | 编码格式，支持h264和h265 | xvid |
| -preset | 编码速率控制，编码加快，意味着信息丢失严重，输出视频质量差 | |
| -crt    | 控制输出质量的，范围0-51，0为无失真编码，建议18-28  | 23|
| -y    | 覆盖输出文件，即如果 output.wmv 文件已经存在的话，不经提示就覆盖掉  | |
| -vf    | 视频过滤器，样例中表示输出保持原始宽高比的1920视频  |  |
| -acodec | 音频编码方式 |  |
| -ab| 音频数据流量，一般选择32，64，96，128 | 推荐使用128 |

部分参数详细说明如下

<font color=red size=3>--crt:</font> 这个选项会直接影响到输出视频的码率，当设置了这个参数之后，再设置－b指定码率不会生效，本人对一个368M的avi文件进行压缩，结果如下(该视频总共450帧，时长15s)

| crf值 | 压缩后文件大小 | 
|--------|---------|
| 20 | 8.14M | 
| 20 | 8.14M | 
| 30 | 2.40M | 


<font color=red size=3>--preset:</font> 指定编码的配置,x264提供了一些预设值，而这些预设值可以通过preset指定。这些预设值有包括：ultrafast，superfast，veryfast，faster，fast，medium，slow，slower，veryslow和placebo。ultrafast编码速度最快，但压缩率低，生成的文件更大，placebo则正好相反。x264所取的默认值为medium。需要说明的是，preset主要是影响编码的速度，并不会很大的影响编码出来的结果的质量

## 常用参数

### 可选视频参数
ps:仅仅列出部分参数，部分高级选项请自行查阅[官方文档](http://ffmpeg.org/ffmpeg.html#Options)

| 命令行参数 | 意义 | 默认值 |
|--------|---------|-------|
| -bitexact | 使用标准比特率 | |
| -vb | 指定视频的比特率，也就是码率 | |
| -s size | 指定分辨率 |  |
| -r rate | 帧率 | 29.97 |


### 可选音频参数
| 命令行参数 | 意义 | 
|--------|---------|
| -ab| 设置比特率(单位：bit/s，也许老版是kb/s)前面，-ac设为立体声时要以一半比特率来设置，比如192kbps的就设成96，转换 默认比特率都较小，要听到较高品质声音的话建议设到160kbps（80） | 
| -ar | 设置音频采样率, 设置音频采样率 (单位：Hz)，PSP只认24000 | 
| -ac | 设置声道数，1就是单声道，2就是立体声，转换单声道的TVrip可以用1（节省一半容量），高品质的DVDrip就可以用2 |  
| -an | 取消音频 |  
| -vol | 设置录制音量大小, 在转换时可以用这个提高音量 | 


## 参考文献
1. [ffmpeg官方文档](http://ffmpeg.org/ffmpeg.html#Options)








