# dotnetcore 无法获取 GBK(GB2312) Encoding 的问题

在使用自带的 zip 库的 `ZipFile.ExtractToDirectory` 时遇到个问题，解压缩出来的中文文件名/目录名变成了乱码。显然这里存在一个编码问题。查看文档，发现 `ZipFile.ExtractToDirectory` 有一个支持指定 `Encoding` 的重载。那么，应该采用哪个 Encoding 呢？

对于这种在中文 Windows 系统上文件名和文本文件的操作，在原来的 .net framework 中，一般是使用 `Encoding.Default` 就可以解决。`Encoding.Default` 会读取操作系统默认的 `ANSI` 编码设定，在中文 Windows 上也就是 `GBK` 编码。虽然语言不同的操作系统的默认设置不一样，比如英文的一般就是 `ASCII`，但对于之前只能运行在 Windows 上的 .net framework 来说，而且环境都是中文语言的前提下，这种设定通常都能解决问题。

而从 .net framework 4.6 起，内置的 encoding 就只支持 `ASCII`、`ISO-8859-1`、`UTF-7`、`UTF-8`、`UTF-16/UTF-16LE`、`UTF-16BE`、`UTF-32/UTF-32LE`、`UTF-32BE` 这几种编码了，dotnet core 自然也延续了这种做法。因此，我们再使用 `Encoding.Default`，通常就只能得到 `ASCII` 的 encoding 了。即使你使用 `Encoding.GetEncoding("GB2312")` 来直接获取对应的 encoding，也只能得到一个异常 `"'GB2312' is not a supported encoding name. For information on defining a custom encoding, see the documentation for the Encoding.RegisterProvider method."`，应该如何解决呢？

可以通过下面的步骤来解决：

1. 引用 `System.Text.Encoding.CodePages.dll`，可通过 Nuget 引用。
2. 加入以下代码，通常在应用启动的时候全局注册。

```cs
System.Text.EncodingProvider provider = System.Text.CodePagesEncodingProvider.Instance;
System.Text.Encoding.RegisterProvider(provider);
```

然后，就可以顺利地通过 `Encoding.GetEncoding("GB2312")` 或 `Encoding.GetEncoding("GBK")` 获得我们亲切的走地中文编码了。


##### 注
不建议使用 `Encoding.Default` 来进行编码操作，毕竟我们现在 dotnet core 应用可能运行在不同的操作系统上，不同的语言环境下，这种会随环境变化的方式实在太不可靠了。


> 参考  
> [ANSI是什么编码？](http://www.cnblogs.com/malecrab/p/5300486.html)  
> [Encoding.Default Property](https://msdn.microsoft.com/en-us/library/system.text.encoding.default%28v=vs.110%29.aspx?f=255&MSPPError=-2147217396)