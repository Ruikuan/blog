# 使用 ffmpeg 将 rtmp 流保存为 mp4 文件

在 Windows 下使用 ffmpeg， 将录制的视频按照日期命名，命令如下
```
set dt=%date:~0,4%%date:~5,2%%date:~8,2%
ffmpeg.exe -i "rtmp://host/videostream" -b:v 900k -vcodec libx264 -acodec aac -b:a 256k -strict -2 -t 2700 %dt%.mp4
```
参数我还没有完全弄清，-t 设定的是录制时间。上面命令使用的是软件编码的方式，cpu 利用率比较高。

机器上使用的是 Nvidia 的显卡，ffmpeg 可以使用 Nvidia 显卡硬编码，几乎不会占用 cpu
```
set dt=%date:~0,4%%date:~5,2%%date:~8,2%
ffmpeg.exe -i "rtmp://host/videostream" -b:v 900k -vcodec nvenc_h264 -acodec aac -b:a 256k -strict -2 -t 2700 %dt%.mp4
```

将这个写成 bat，让计划任务跑，就可以每天定时录制电视剧节目了。