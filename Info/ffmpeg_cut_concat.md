# 使用 ffmpeg 切割和合并视频文件

试用了不少软件来进行视频文件的简单操作，例如切割和合并，效果都难以让人满意。今天试下直接使用 ffmpeg 来操作一下，效果相当不错，比各种自由软件好多了。

### 切割
切割命令如下：
```
ffmpeg -i input.mp4 -ss 00:21:25.0 -to 00:33:05.0 -c copy output.mp4
```
从视频从21分25秒 到 33分05秒的片段切割出来。其中 -to 参数可以换为 -t， -t表示的是切割的时间段，例如切割10秒钟的片段。

### 合并
只是简单的将多个参数完全一致的文件合并的话，可以采用 concat demuxer 的方式。
首先生成一个待合并视频列表文件 list.txt，内容如下
```
file 'input1.mp4'
file 'input2.mp4'
```
然后执行命令
```
ffmpeg -f concat -i list.txt -c copy output.mp4
```
轻松搞定。

## 改变分辨率

譬如将 1080p 的视频大小变为它的一半

```
ffmpeg -i video_1920.mp4 -vf scale=960:540 video_540.mp4 -hide_banner
```

## 处理音频

### 将 MP4 中的音频转录到 MP3

```
ffmpeg -i filename.mp4 filename.mp3
```
或
```
ffmpeg -i video.mp4 -b:a 192K -vn music.mp3
```

### 截取 MP3 中的一段

```
ffmpeg -i file.mp3 -ss 00:00:20 -to 00:00:40 -c copy file-2.mp3
```

### 合并 MP3

```
ffmpeg -i "concat:1.mp3|2.mp3" -acodec copy output.mp3
```

#### 参考
> [Using ffmpeg to cut up video](https://superuser.com/questions/138331/using-ffmpeg-to-cut-up-video)  
> [Concatenate two mp4 files using ffmpeg](https://stackoverflow.com/questions/7333232/concatenate-two-mp4-files-using-ffmpeg)
