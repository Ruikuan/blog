# Chorme 和 asp.net core ajax with Credentials 跨域访问实现

一般来说，asp.net core 只要这样简单配置代码，就能和Edge配合的很好了，Edge默认就会使用对应目标站点的 Credentials
```csharp
app.UseCors(options =>{options.AllowAnyOrigin());
```

但Chorme要求比较严格，首先需要在客户端配置 withCredentials
```javascript
//vue resource 的配置，其他的同理
Vue.http.options.xhr = {withCredentials: true}
```
而且对服务器端的返回也有严格限制，当要带上Credentials时，不能允许 Origins 设置为 *，只能指定特定的Origins，
另外，服务器端也必须指定允许Credentials，因此需要将服务器端代码改为：
```csharp
app.UseCors(options =>
{
    options.WithOrigins("http://localhost:8080"); //特定的源
    options.AllowCredentials();  //显式指定
});
```