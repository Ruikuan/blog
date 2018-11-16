# ffmpeg 转换视频格式

将 mkv 转成 mp4

```
ffmpeg -i LostInTranslation.mkv -codec copy LostInTranslation.mp4
```
由于这两个只是封装格式不同，编码可以不变。转换基本不需要花费什么 cpu 工作来转换编码，非常快。