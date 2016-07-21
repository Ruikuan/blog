# 使用 ffmpeg 将 rtmp 流保存为 mp4 文件

在 Windows 下使用 ffmpeg， 将录制的视频按照日期命名，命令如下
```
set dt=%date:~0,4%%date:~5,2%%date:~8,2%
ffmpeg.exe -i "rtmp://live" -b:v 900k -vcodec libx264 -acodec aac -b:a 256k -strict -2 -t 2700 %dt%.mp4
```
ffmpeg 的参数太复杂，我还没有完全弄清，这里 -b:v 是视频的码率，-b:a 是音频的码率，-t 设定的是录制时间。上面命令使用的是软件编码 (libx264) 的方式，cpu 利用率比较高。

机器上使用的是 Nvidia 的显卡，ffmpeg 可以使用 Nvidia 显卡硬编码 (h264_nvenc)，几乎不会占用 cpu
```
ffmpeg.exe -i "rtmp://live" -b:v 900k -vcodec h264_nvenc -acodec aac -b:a 256k -strict -2 output.mp4
```
如果想要编码的视频质量更高，可以使用另外的编码参数，如下：
```
ffmpeg.exe -i "rtmp://live" -b:v 900k -vcodec h264_nvenc -profile:v high -preset default -acodec aac -b:a 256k -strict -2 output.mp4
```
也可以加大码率，或者干脆去掉 -b:v 码率，让它按照源的码率来工作。这主要是文件体积和视频质量的权衡。
```
ffmpeg.exe -i "rtmp://live" -c:v h264_nvenc -profile:v high -preset default -acodec aac -b:a 256k -strict -2 output.mp4
```


将这个写成 bat，放计划任务跑，就可以每天定时录制电视剧节目了。


