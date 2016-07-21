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
