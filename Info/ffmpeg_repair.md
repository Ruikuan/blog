# ffmpeg 修复视频文件

有时候下载视频文件到 99.9% 就无限卡住，就是没办法下载完。这时候一般把文件扩展名改为正常的视频文件就能看了，不过因为文件不完整，可能索引有点问题，拖动进度条等操作不正常，很耗时而且经常卡死。

可以通过下面的命令尝试修复。

```
ffmpeg -err_detect ignore_err -i video.mkv -c copy video_fixed.mkv
```

不用重新编解码，速度很快。