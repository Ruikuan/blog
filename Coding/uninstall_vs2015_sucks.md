# 解决因为卸载 vs2015 导致 LocalDB 没法使用的问题

这些天，看着 mSATA 128G 的 c 盘剩余空间所剩无几，想着已经安装了 vs2017，不如将 vs2015 卸载了，看能不能给 c 盘腾出多点空间。于是手贱将 vs2015 卸载掉了。控制面板卸载之后，还有很多相关的组件不会被卸载，看着碍眼，便更作死地找了微软的这个[全面清除 vs 的工具](https://github.com/Microsoft/VisualStudioUninstaller)，一下子将 vs 相关的东西全部干掉了。这下好了，不单 vs2015、还有早期残留的2013、2010、2008 都被干得干干净净。

但运行 vs2017 编译就报错了，找不到对应的组件。这个好办，运行下安装程序，修复一下，搞定了。ctrl + F5 编译通过，浏览器打开，跑起来了。

还是有问题，程序报异常：无法连接 sqlserver，找不到对应实例名称啥的。想用 SQL Server Management Studio 连一下，SSMS 也被 TotalUninstaller 一并干掉了，只好当场下载，安装。结果同样还是连不上，类似的错误。

估摸着应该是 TotalUninstaller 将 LocalDB 也干掉了，但看了下 visualstudio 2017 的安装程序，修复的时候应该也将 LocalDB 重新装上去了啊。不知道是不是装上去了，反正再下载一个 LocalDB 的安装程序，自己装下吧。于是下载了个 SQL Server 2016 LocalDB 的安装程序，装上去。测试下，仍然是有问题，未解决。

在 vs2017 的 server explorer 里面尝试添加一个 connection，说找不到实例。执行命令重新生成实例：
```
sqllocaldb delete MSSQLLocalDB
sqllocaldb create MSSQLLocalDB
```
重建了一个当前版本的实例，版本号为 13.*。直接将数据库文件 attach 上去，说不兼容的数据库。运行 cmd 命令
```
sqllocaldb v
```
结果在机器上报错，说访问注册表项返回 0 错误。

再想回来，我之前的 localdb 是基于 vs2015 自带的 SQL Server 2014 的 localdb，现在是 vs2017 的 SQL Server 2016 的 localdb，不兼容也正常。遂找到一个 sqlserver 2014 的安装程序，装了这个版本的 localdb。这下 sqllocadb v 列出了两列数据，一个是 12.* 的正常版本号，另一个本来应该是 13.* 版本号的，却显示同样的注册表项访问错误。既然显示错误，那么直接将 sqlserver 2016 localdb 卸载了。吸取经验，从控制面板卸载都是不干净的，直接运行它的安装程序，在安装程序里面选择卸载。再用 v 看了下，现在只剩下一个版本 12.* 了。

现在应该走上正道了。再用上面的指令删除 MSSQLLocalDB 再重建，得到了一个版本号为 12.* 的默认的 MSSQLLocalDB 实例。这下用 vs2017 的 server explorer 能顺利将数据库文件 attach 上去了，而且也能通过它的工具执行 sql 操作了。修改一下程序的 connection string，程序也顺利跑起来了。

但 SSMS 还是连不上，仔细看了下，擦，原来连的服务器是从 connection string 复制过来的：
```
(LocalDB)\\MSSQLLocalDB
```
而因为 connection string 是保存在 json 文件中的，将“\”写成了“\\”做转义。将它改过来，SSMS 也能顺利连上去了。因为这个乌龙，导致排查过程中花了不少时间。

至此，问题都解决了，避免了一次要重装系统的危机。总结一下主要问题在于 SSMS 连接服务器的乌龙，导致排查时间大大增加。另外一个是 localdb 不同版本的兼容性，导致问题。