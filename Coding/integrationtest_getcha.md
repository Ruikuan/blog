# 升级 .net 5.0 之后的 Integration Testing 小问题

项目升级到 .net 5.0 之后，原先 3.1 环境下运行得好好的 integration testing 运行失败了。感觉每次升级版本，它总会出问题，之前从 WebHost 升级到 Generic Host 就已经调整了一番，然后为了[解决验证问题也做了一番调整](./integration_testing_by_pass_auth.md)，升级到 5.0 之后，所有测试又全都失败了。这次主要失败在创建 HttpClient，因为我已经对 `WebApplicationFactory` 做了很多改动，在 Startup 的时候做了很多初始化操作，它失败的时候直接就跪了，没有提供什么有效信息。

看回官方文档 [Integration tests in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-5.0)，比之前又完善了很多。按照它上面的做法，把我原来凭自己摸索出来的处理方式做了改变，用 `dotnet test` 就能跑测试了。但 Test Explorer 上面的测试就一个都跑步了，尝试跑它就全都跳过，一个都没跑。

网上搜了下，[Stack Overflow 上面有人说要把测试项目以及待测试项目的 cpu 架构调整成一致](https://stackoverflow.com/a/42765018/7678578)，不然它就会这个样子。看了下我两个项目都一样的 AnyCPU，不过再仔细一想，为了方便发布成 Self-Contained 的 SingleFile，我的确修改了源项目的 `RuntimeIdentifier` 变成了 `win10-x64`， 测试项目想着也不会发布，就没有动它，会不会是这个不匹配造成的呢？

测试项目也对应添加了 `RuntimeIdentifier` 之后，Test Explorer 上面的测试终于又恢复了正常，可以直接在上面跑了。

