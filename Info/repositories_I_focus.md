# 我关注的 Repositories

记在这里免得到时候找起来麻烦
* [dotnet/home](https://github.com/Microsoft/dotnet) 开源 dotnet 的主页
* [dotnetcore/home](https://github.com/dotnet/core)  dotnetcore 主页
    * [coreclr - .NET Core Runtime](https://github.com/dotnet/coreclr) 
    * [corefx - .NET Core Libraries](https://github.com/dotnet/corefx)  
    * [.NET Compiler Platform ("Roslyn")](https://github.com/dotnet/roslyn)
    * [.NET Core Lab](https://github.com/dotnet/corefxlab) This repo is for experimentation and exploring new ideas that may or may not make it into the main corefx repo.
        * [Span<T>](https://github.com/dotnet/corefxlab/blob/master/docs/specs/span.md) Span<T> is a new type we are adding to the platform to represent contiguous regions of arbitrary memory, with performance characteristics on par with T[]. 
    * [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet) BenchmarkDotNet is a powerful .NET library for benchmarking.
* [asp.net core home](https://github.com/aspnet/home) 
    * [asp.net core mvc](https://github.com/aspnet/Mvc)
    * [Entity Framework Core](https://github.com/aspnet/EntityFramework) 
    * [Razor](https://github.com/aspnet/Razor)
    * [DependencyInjection](https://github.com/aspnet/DependencyInjection)
* [Elasticsearch](https://github.com/elastic/elasticsearch) A Distributed RESTful Search Engine
* [nopCommerce](https://github.com/nopSolutions/nopCommerce) 基于 asp.net(no core) 的电子商务网站，有点重，不支持 async
* [SimplCommerce](https://github.com/simplcommerce/SimplCommerce) A super simple, cross platform, modularized ecommerce system built on .NET Core
* [NMemory](https://github.com/tamasflamich/nmemory) 一个用 c# 开发的内存数据库，不一定好用，就看看实现方式
* [CSharpEWAH](https://github.com/lemire/csharpewah) 用 c# 写的位图索引压缩库，很久很久没更新了
* [Orchard](https://github.com/OrchardCMS/Orchard) 一个基于 asp.net mvc 的 CMS
* [Umbraco CMS](https://github.com/umbraco/Umbraco-CMS) 一个基于 asp.net 的 CMS
* [CommonMark.NET](https://github.com/Knagis/CommonMark.NET) markdown -> html 
* [Markdig](https://github.com/lunet-io/markdig) Markdig is a fast, powerful, CommonMark compliant, extensible Markdown processor for .NET.
* [Stackoverflow](https://github.com/StackExchange)
    * [StackExchange.Redis](https://github.com/StackExchange/StackExchange.Redis) 高性能的 .net redis client
    * [Dapper](https://github.com/StackExchange/dapper-dot-net) 简单的 orm，高性能
    * [Jil](https://github.com/kevin-montrose/Jil) 一个据说很快的 Json 库，使用生成 IL 的方式工作，但好像不支持 dotnet core
* [The Accord.NET Framework](https://github.com/accord-net/framework/) 基于 .NET 的机器学习库（认知等），从 AForge.NET 扩展而来 
* [Orleans - Distributed Virtual Actor Model](https://github.com/dotnet/orleans) 微软分布式云计算框架
* [机器学习资料](https://github.com/ty4z2008/Qix) 里面有两篇文章对于机器学习的资料收集得比较全
* [Universal Windows app samples](https://github.com/Microsoft/Windows-universal-samples)
* [ILSpy](https://github.com/icsharpcode/ILSpy)
* [KCP - A Fast and Reliable ARQ Protocol](https://github.com/skywind3000/kcp) KCP是一个快速可靠协议，能以比 TCP浪费10%-20%的带宽的代价，换取平均延迟降低 30%-40%，且最大延迟降低三倍的传输效果，底层用通常用 udp，一些说明参见 [这里](https://zhihu.com/question/48777542/answer/112575371)
    * [TCP端口加速器](https://github.com/xtaci/kcptun) TCP端口加速器，用于kcp-go协议测试，可以用来加速 ss
* [Microsoft.IO.RecyclableMemoryStream](https://github.com/Microsoft/Microsoft.IO.RecyclableMemoryStream) 一个 pooled stream，使用它可以重用 memorystream，避免频繁的 GC 甚至 Gen2 GC，极大提高系统的性能和可伸缩性。
* [NAudio](https://github.com/naudio/NAudio) 一个音频处理的项目，近期可能用到。 
* [FFmpegInterop library for Windows](https://github.com/Microsoft/FFmpegInterop) 在 UWP 中使用 FFmpeg
* [ZeroFormatter](https://github.com/neuecc/ZeroFormatter) Fastest C# Serializer and Infinitely Fast Deserializer for .NET, .NET Core and Unity.
* [ClosedXML](https://github.com/closedxml/closedxml) ClosedXML makes it easier for developers to create Excel 2007/2010/2013 files.
* [.NET Standard based Windows Service support for .NET](https://github.com/dasMulli/dotnet-win32-service) Helper classes to set up and run as windows services directly on .net core.
* [RazorEngine](https://github.com/Antaris/RazorEngine) Open source templating engine based on Microsoft's Razor parsing engine
* [MessagePack for CLI](https://github.com/msgpack/msgpack-cli) This is MessagePack serialization/deserialization for CLI (Common Language Infrastructure) implementations  
* [Inferno](https://github.com/sdrapkin/SecurityDriven.Inferno) 一个基于 .NET 的加密库
* [Ulterius](https://github.com/Ulterius/server) 基于 .NET 的远程管理解决方案，管理服务器的文件、监控硬件资源等。貌似不是太高端，但它能传界面到浏览器。
* [Wyam](https://github.com/Wyamio/Wyam) Wyam is a simple to use, highly modular, and extremely configurable static content generator that can be used to generate web sites, produce documentation, create ebooks, and much more. 
* [Jint](https://github.com/sebastienros/jint) Jint is a Javascript interpreter for .NET which provides full ECMA 5.1 compliance and can run on any .NET platform. Because it doesn't generate any .NET bytecode nor use the DLR it runs relatively small scripts faster.
* [Zepto.js](https://github.com/madrobby/zepto) Zepto is a minimalist JavaScript library for modern browsers with a largely jQuery-compatible API. If you use jQuery, you already know how to use Zepto.
* 图形处理库
    * [ImageSharp](https://github.com/JimBobSquarePants/ImageSharp) 完全托管代码写成，可移植。完成度高，但不是非常完善，性能尚可
    * [Magick.NET](https://magick.codeplex.com/) 完善，支持 .NET Core 但只支持 Windows，后面可能会改。追求图像质量。
    * [CoreCompat.System.Drawing](https://github.com/CoreCompat/CoreCompat) 完全支持 System.Drawing 的 API，支持 Linux，但照样用的 GDI+，不适合多线程使用
    * [SkiaSharp](https://github.com/mono/SkiaSharp/) The .NET wrapper for [Google’s Skia cross-platform 2D graphics library](https://skia.org/). 非常快，但目前还不支持 .NET core，可以在 mono 上跑。
* [WeChatircd](https://github.com/MaskRay/wechatircd) IRC控制微信 [让IRC客户端控制微信网页版](https://maskray.me/blog/2017-02-19-how-i-use-wechat-recent-updates-of-wechatircd)
* [FluentEmail](https://github.com/lukencode/FluentEmail) All in one email sender for .NET and .NET Core. Send email from .NET or .NET Core. A bunch of useful extension packages make this dead simple and very powerful. 可用 Razor 做模板。
* [Senparc.Weixin —— 微信 .NET SDK](https://github.com/JeffreySu/WeiXinMPSDK) 包含微信支付