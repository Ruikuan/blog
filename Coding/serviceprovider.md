# 在 asp.net core 中通过非注入的方式获取 service

通常，在开发 asp.net core 应用的时候，会将需要依赖的 service 通过构造方法注入到对象中，这样在对象中就可以使用对应的服务了，类似如下的方式
```csharp
public class ApiController : Controller
{
    private CommitService _commitService = null;
    private WorkContext _workContext = null;

    public ApiController(CommitService commitService, WorkContext workContext)
    {
        _commitService = commitService;
        _workContext = workContext;
    }
}
```
但有时候我们并不是通过处理 http 请求的方式调用代码，比如我们需要在 asp.net core 中执行一个后台任务，这个后台任务是运行在后台线程中或者通过一个 timer 来触发，触发的代码需要访问系统里面的各个 service，但并没有一个方便的依赖注入点，这个时候我们应该怎么获得已经注册了的 service 呢？因为依赖注入实在是方便，我们是不会愿意自己手工用 new 构造 service的。

在 DotNetCore 中内置了一个 IServiceProvider 接口，asp.net core 就是使用它来获得注册服务的。IServiceProvider 接口如下：
```csharp
public interface IServiceProvider
{
    /// <summary>Gets the service object of the specified type.</summary>
    /// <returns>A service object of type <paramref name="serviceType" />.-or- null if there is no service object of type <paramref name="serviceType" />.</returns>
    /// <param name="serviceType">An object that specifies the type of service object to get. </param>
    /// <filterpriority>2</filterpriority>
    object GetService(Type serviceType);
}
```
引用了 Microsoft.Extensions.DependencyInjection.Abstractions 程序集之后，还可以使用泛型的 GetService<T>(this IServiceProvider provider) 方法。
```csharp
CommitService commitService = ServiceProvider.GetService<CommitService>();
```
所以问题就变成了，我们如何获得 ServiceProvider，然后就可以在适当的时候用它获得我们需要的服务。   


在 asp.net core 中获取 ServiceProvider 的方式大概有如下这些方式：

### 1. 通过 IServiceCollection.BuildServiceProvider 方法获得 ServiceProvider

这个扩展方法在Microsoft.Extensions.DependencyInjection 的 Assembly 中。  
serviceCollection 可以在 startup 组装 service 的时候获得。  
需要注意的是，BuildServiceProvider 生成的 provider 只能获取 build 之前注册的 service， 之后再注册到 serviceCollection 中的 service 是不能获取的。

### 2. 通过 IAppBuilder.ApplicationServices 获得 ServiceProvider

Appbuilder 可以在 startup Configure 的时候获得。

### 3. 通过 HttpContext.RequestServices 获得 ServiceProvider

这个没什么好说的，如果能拿到 HttpContext 对象，通常也能通过 Controller 的构造方法实现依赖注入了。

### 4. 通过 IWebHost.Services 获得 ServiceProvider

在 Program 类里面初始化完毕 webhost 之后，可以通过它获得。
  
  
  
获得想要的 ServiceProvider 之后，我们可以把这个 provider 放到随便某个后台任务能够访问的地方，需要的时候就能 happy 地用它获得各种 service 了。

