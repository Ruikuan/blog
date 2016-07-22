# 部署 asp.net core 应用到 windows server 2008 r2

我正在用的 dotnet core 版本为 1.0。
预先在 windows 2008 r2 上安装 iis 之类的就先跳过了。一般的部署步骤可以参考微软的官方教程 [部署到 asp.net core 到 iis ](https://docs.asp.net/en/latest/publishing/iis.html)
但依照教程做之后，还有问题，访问网站还是会出错。其实还是有些依赖组件要安装的，但教程里面没有提。
#### 1. 安装 .NET Core Windows Server Hosting 
这个教程里面有说明
#### 2. 安装 KB2533623 补丁
这是 dotnet core 的一个前置依赖，到 https://support.microsoft.com/en-us/kb/2533623 下载对应版本安装
#### 3. 安装 KB2999226 补丁
另一个依赖，到 https://support.microsoft.com/en-us/kb/2999226 下载安装

都安装完之后，如果正确按照教程的指引来正确配置了 iis 和 publish 的话，网站就能顺利跑起来了。


#### 关于 Data Protection Key
按照上面步骤操作下来，网站是能跑起来了，但还有个问题，就是 Data Protection 的 key 会漂移。  
在没有使用持久化 key 的时候，每次 dotnet 跑起来，都会生成并使用新的 key，这样就导致每次进程起来 key 都不一样，而这个 key 不单影响显式用 data protect api 的操作，还影响用户验证等，造成的现象就是即使使用持久化 cookie，每次服务程序重启之后，验证信息还是会丢失，又需要用户重新登录。  
教程 [部署到 asp.net core 到 iis ](https://docs.asp.net/en/latest/publishing/iis.html) 里有提到使用 powershell 脚本生成并保存 key 到注册表的方法，但这个脚本在 windows 2008 r2 下直接运行是有问题的，会提示 [Microsoft.Win32.RegistryView] 类型找不到。解决方法是安装个新的 powershell，下载这个补丁 [ Windows Management Framework 5.0 ](https://www.microsoft.com/en-us/download/details.aspx?id=50395) 安装之后，再运行脚本即可。