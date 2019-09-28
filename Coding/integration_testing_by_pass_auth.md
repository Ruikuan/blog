# asp.net core 集成测试如何自动通过身份验证挑战

关于 asp.net core 的集成测试，官方文档 [Integration tests in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/test/integration-tests) 有比较详细的介绍。不过还有些文档里面没有说明清楚的地方，趟了一些坑，这里记录一下。

示例项目参见 https://github.com/Ruikuan/AuthWebAppIntegrationTestingDemo, 有 [core2.2](https://github.com/Ruikuan/AuthWebAppIntegrationTestingDemo/tree/core2.2) 和 [core3.0](https://github.com/Ruikuan/AuthWebAppIntegrationTestingDemo/tree/core3.0) 两个 branch，有些细节上面的不同，后面会说一下。

## 问题

我们的 `HomeController` 有两个 `Index` 和 `Privacy` 两个 Action，一个不需要验证，另一个需要身份验证。

```cs
public IActionResult Index()
{
    return View();
}

[Authorize]
public IActionResult Privacy()
{
    return View();
}
```

现在需要对这两个方法进行 Integration testing。示例文档里面只演示了遇到身份验证，进行重定向的测试。这里讲模拟通过了身份验证的测试。

## 解决方案

我们的解决办法是构建一个自动通过身份验证的中间件，将它插入到请求中间件管道的适当位置，这样所有经过这个中间件处理的 request 都具有了验证过的身份。

#### 批发验证用户中间件

创建中间件如下：（以下的代码全为 3.0 版本，若要看 2.2 版本的，参考上面的链接）

```cs
public class AuthenticatedTestRequestMiddleware
{
    private const string AuthenticationType = "TestAuthenticationType";

    public const string TestingHeader = "X-Integration-Testing";
    public const string TestingHeaderValue = "Pass-Auth";

    public const string FakeLoginIdHeaderKey = "X-Test-LoginId";

    private readonly RequestDelegate _next;


    public AuthenticatedTestRequestMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Headers.Keys.Contains(TestingHeader) &&
            context.Request.Headers[TestingHeader].First().Equals(TestingHeaderValue))
        {
            if (context.Request.Headers.Keys.Contains(FakeLoginIdHeaderKey))
            {
                var loginId = context.Request.Headers[FakeLoginIdHeaderKey].First();

                ClaimsIdentity identity = new ClaimsIdentity(AuthenticationType);
                identity.AddClaim(new Claim(ClaimTypes.Name, loginId));

                var claimsPrincipal = new ClaimsPrincipal(identity);
                context.User = claimsPrincipal;
            }
        }

        await _next(context);
    }
}
```

这个中间件验证每个请求，如果请求带有创建验证凭据的 header，就自动给它创建一个 `ClaimsPrincipal`。

#### 改变中间件管道

有了中间件，我们要把它插入测试请求管道的适当位置。众所周知，原 Web 应用的中间件管道是通过 `Startup.Configure` 来配置的，为了不影响原应用的处理路径，又要在测试中改变管道行为，我们创建一个继承 `Startup` 的 `TestServerStartup` 类。而且为了能对原管道进行改变，我们需要在原 `Startup.Configure` 中做一些微调。

原 `Startup` 变动：

```cs
// Startup.cs
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    else
    {
        app.UseExceptionHandler("/Home/Error");

        // --> 删除 app.UseHttpsRedirection(); 
        // --> 删除 app.UseHsts();
        ConfigureHttps(app); // <-- 将这里的 https 相关配置转移到 ConfigureHttps(app) 中
    }
    app.UseStaticFiles();
    app.UseRouting();
    app.UseAuthentication();

    ConfigureAdditionalMiddleware(app); // <-- 在管道组建代码中插入一个适当的 stub，我们后面将假造验证用户的中间件放到这个位置。
    
    app.UseAuthorization();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllerRoute(
            name: "default",
            pattern: "{controller=Home}/{action=Index}/{id?}");
    });
}
        //virtual 以供替换
protected virtual void ConfigureAdditionalMiddleware(IApplicationBuilder app)
{

}
        //virtual 以供替换
protected virtual void ConfigureHttps(IApplicationBuilder app)
{
    app.UseHsts();
    app.UseHttpsRedirection();
}

```

之所以需要有 `ConfigureHttps` 处理，是因为 integrating testing 是运行在 `Production` 运行环境的，如果不进行改变，所有的请求都会被重定向到对应的 https 路径，也就是所有的请求都会得到一个 redirect 响应，通常为 307。那么我们的所有请求测试都会失败。

创建 `Startup` 的子类 `TestServerStartup`：

```cs
public class TestServerStartup : AuthWebApp.Startup
{
    public TestServerStartup(IConfiguration configuration) : base(configuration)
    {
    }

    protected override void ConfigureAdditionalMiddleware(IApplicationBuilder app)
    {
        app.UseMiddleware<AuthenticatedTestRequestMiddleware>();
    }

    protected override void ConfigureHttps(IApplicationBuilder app)
    {
    }
}
```

就简单的讲中间件加入管道中，然后去除 https 重定向部分。

#### 将 `TestServerStartup` 用于测试

我们需要创建一个 `WebApplicationFactory<TEntryPoint>` 的子类 `TestWebApplicationFactory<TEntryPoint>` 将 `TestServerStartup` 用于构建测试服务器。

```cs
public class TestWebApplicationFactory<TEntryPoint> : WebApplicationFactory<TEntryPoint> where TEntryPoint : class
{
    protected override IHostBuilder CreateHostBuilder()
    {
        return base.CreateHostBuilder().ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<TestServerStartup>(); //<-- 
        });
    }
}
```

参照原 WebApp 的 `Program` 模板来构建 `HostBuilder`，将 `TestServerStartup` 用起来。这里 asp.net core 3.0 跟 2.2 区别很大，在 2.2 中，需要 `override IWebHostBuilder CreateWebHostBuilder()`，并参照 2.2 的 `program` 模板来构建 `WebHostbuilder`，详细参考上面链接。

这里还有一个注意点是 `TEntryPoint` 类型参数跟 `UseStartup` 使用的类型没有什么关系。`UseStartup<TestServerStartup>` 表示用 `TestServerStartup` 来构建服务器的依赖注入容器、中间件处理管道等，而 `TEntryPoint` 是 `WebApplicationFactory` 用来查找并匹配要测试应用（SUT）的一些属性的，譬如 `RootContentPath` 等。我们最后一般用测试目标项目的 `Startup` 来实例化 `TEntryPoint`，让它得到原应用的属性。

#### 开始测试

有了 `TestWebApplicationFactory`，我们就可以将它用到测试代码中了。测试代码如下：

```cs
                                                                 //使用原项目的 Startup
public class UnitTest1 : IClassFixture<TestWebApplicationFactory<AuthWebApp.Startup>> // <-- use SUT's Startup here, for it helps to set root path.
{
    private readonly TestWebApplicationFactory<AuthWebApp.Startup> _factory;

    public UnitTest1(TestWebApplicationFactory<AuthWebApp.Startup> factory) => _factory = factory;

    [Fact]
    public async Task Visit_Index_Success()
    {
        // Arrange
        var client = _factory.CreateClient(new WebApplicationFactoryClientOptions
        {
            AllowAutoRedirect = false
        });

        // Act
        var response = await client.GetAsync("/");

        // Assert
        response.EnsureSuccessStatusCode(); // Status Code 200-299
        Assert.Equal("text/html; charset=utf-8",
            response.Content.Headers.ContentType.ToString());

    }

    [Fact]
    public async Task Visit_Privacy_Success()
    {
        // Arrange
        var client = _factory.CreateClient(new WebApplicationFactoryClientOptions
        {
            AllowAutoRedirect = false
        });
        client.DefaultRequestHeaders.Add(AuthenticatedTestRequestMiddleware.TestingHeader, AuthenticatedTestRequestMiddleware.TestingHeaderValue);
        client.DefaultRequestHeaders.Add(AuthenticatedTestRequestMiddleware.FakeLoginIdHeaderKey, "fakeUser");

        // Act
        var response = await client.GetAsync("/Home/Privacy");

        // Assert
        response.EnsureSuccessStatusCode(); // Status Code 200-299
        Assert.Equal("text/html; charset=utf-8",
            response.Content.Headers.ContentType.ToString());
    }
}
```

在 `Visit_Privacy_Success` 中，我们将 header 注入 http 请求里，这样我们的中间件就能根据这些 header 来决定要不要生成验证票据，验证票据里面包含哪些内容。

运行 `dotnet test`，任务完成！


## 一些注意的地方

1. 在 3.0 项目中，我们的中间件要在 `app.UseAuthorization()` 前面，在 2.2 项目中，我们的中间件只要在 `app.UseMvc()` 前面就行了。
2. 在 2.2 项目中，如果我们的项目文件（测试项目跟被测试项目）的 `Microsoft.AspNetCore.App` 元引用包没有指定版本，有可能会导致测试失败，所有请求都返回 404 not found。很烦人，也很难找到原因。最简单的解决方法就是明确加入版本号 `2.2.0`，这里有相关的陈述 https://github.com/aspnet/AspNetCore/issues/8428。
3. 测试和被测试项目的 SDK 要设置为 `Microsoft.NET.Sdk.Web` 而不能是 `Microsoft.NET.Sdk`。

综合 2 和 3，就是 
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>netcoreapp2.2</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.App" Version="2.2.0"/>
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="2.2.0" />
    <PackageReference Include="xunit" Version="2.4.1" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.4.1" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\AuthWebApp\AuthWebApp.csproj" />
  </ItemGroup>

</Project>
```