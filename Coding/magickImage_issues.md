# 使用 MagickImage.NET 的一些坑

MagickImage.NET 是一个图片处理库，可以用于转换图片格式、改变图片大小等。在 .net framework 和 dotnet core 上都可以跑。我主要用它来生成缩略图。生成缩略图的代码很简单，如下：

```cs
using (MagickImage image = new MagickImage(sourceFile))
{
    image.Thumbnail(800, 800); // 按照比例生成宽高都不会超过 800 的缩略图
    image.Write(targetFile);
}
```

不过还是遇到了一些坑。

### 光跑 CPU 不干活

第一次运行代码 demo 时，cpu 飙到 100%，风扇狂转，然而代码就是跑不过 `image.Thumbnail(800, 800)` 那行，感觉是挂在了那里，但它又没有抛异常。我担心哪里出了问题，赶紧把进程 kill 掉了。重复几次，都是同样的表现。心里想，这玩意真不靠谱！  

后来又硬着头皮跑了一两分钟没 kill 进程，看它会玩出什么幺蛾子。嘿！结果烤完两分钟之后，它就正常了，缩略图也顺利生成了。之后再跑，也不会再烧 cpu 了，每次都是立刻就生成缩略图。  

最后在它的 github 里得到了回应，说它是能用 OpenGL 来处理图片的，每台机器第一次跑的时候，会先进行基准测试，看到底是 GPU 快还是 CPU 快，然后选择 GPU 还是 CPU 来处理图片。如果嫌这样烦，直接通过 `OpenCL.IsEnabled = false;` 将 OpenGL 处理禁用就可以了。

### 奇怪的全黑缩略图

然后在使用过程中，又出现了诡异的问题。

生成一批图片的缩略图时，第一张生成的缩略图永远都是全黑的，而且只有 4KB 多点的大小，而之外的其他图片都正常。而且如果将缩略图的大小改成 900，它又没有问题了，但 700/750 也会随机出问题。真是百思不得其解。

而且这个问题只会出现在上面这种代码里，如果使用下面的代码，也不会有问题。但这样的代码跑起来慢得离谱。

```cs
using (MagickImage image = new MagickImage(sourceFile))
{
    MagickGeometry size = new MagickGeometry(800, 800);
    size.IgnoreAspectRatio = false;
    image.Resize(size);
    image.Write(targetFile);
}
```

在这个诡异的问题上消磨了半天，忽然灵机一动，会不会是因为使用 GPU 来处理图片的问题？在我印象中，之前 GPU 某些时候并不是太稳定，偶尔出现花屏，黑屏之类的问题。嗯，Microsoft Edge 就经常崩……

然后按照上面的方法禁用了 OpenGL 之后，问题就不再出现了，代码跑得很欢快，腰不酸了，腿不疼了，一口气搞了好多张。正好这些代码是要部署到 linux server 上面的，上面根本没有什么鬼的 GPU。


## 结论

结论就是要想用得宽心点，还是不要用它的 OpenGL 了……