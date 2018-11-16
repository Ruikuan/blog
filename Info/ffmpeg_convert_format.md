# ffmpeg 转换视频格式

### 将 mkv 转成 mp4

```
ffmpeg -i LostInTranslation.mkv -codec copy LostInTranslation.mp4
```
由于这两个只是封装格式不同，编码可以不变。转换基本不需要花费什么 cpu 工作来转换编码，非常快。

### 转换成适合串流观看的 mp4

如 [优化 MP4 视频以便更快的网络串流](https://macplay.github.io/posts/you-hua-mp4-shi-pin-yi-bian-geng-kuai-de-wang-luo-chuan-liu/) 所述：  

> MP4 文件由名为 "`atoms`" 的数据块构成 。有存储字幕或章节的 `atoms` ，同样也有存储视频和音频数据的 `atoms` 。至于视频和音频 `atoms` 处于哪个位置，以及如何播放视频诸如分辨率和帧速等，这些元数据信息都存储于一个名为 `moov` 的特殊 `atom` 之中。当你播放视频时，程序搜寻整个 MP4 文件，定位 `moov` `atom`，然后使用它找到视频和音频数据的开头位置，并开始播放。然而， `atoms` 可能以任何顺序存储，所以程序无法提前得知 `moov` `atom` 在文件的哪个位置。如果你已经拥有整个视频文件，搜寻并找到 `moov` `atom` 问题并不大。然而，当你还没有拿到整个视频文件（比如说你串流播放 HTML5 视频时），恐怕就希望可以有另外一种方式。而这，就是串流播放视频的关键点！你无需事先下载整个视频文件，就可以立即开始观看。
> 
> 当串流播放时，你的浏览器会请求视频并接受文件头部，它会检查 `moov` `atom` 是否在文件开头。如果 `moov` `atom` 没有在文件开头，则它要么得下载整个文件并试图找到 `moov` ，要么下载视频文件的不同小块并寻找 `moov` `atom`，反复搜寻直到遍历整个视频文件。
> 
> 搜寻 `moov` `atom` 的整个过程需要耗费时间和带宽。很不幸的是，在 `moov` 被定位之前的这段时间里，视频都不能开始播放。

因此需要对 mp4 文件进行优化，重组 atoms，使 `moov` `atom` 位于文件开头。可以使用如下命令。

```
ffmpeg -i source_file -movflags faststart -codec copy output.mp4
```

`-movflags faststart` 参数告诉 ffmpeg 重组 MP4 文件 `atoms`，将 `moov` 放到文件开头。