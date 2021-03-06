# 使用 RemoteApp 来使用 QQ 旺旺之类鬼鬼祟祟的软件

QQ 之类的软件，因为一直以来的依赖，是刚需。但这些软件并不自律，QQ 安装的时候就给你装上一个 `QPCore Service` 的系统服务，这个服务还流氓到无法在服务管理控制台手动终止。一旦你通过杀进程或者禁止它启动，它就立刻耍流氓让 QQ 就不能使用。而且前段时间还有[通过这个服务给用户喂全家桶的行径](https://news.cnblogs.com/n/585753/)。同样的还有阿里旺旺，自带了个什么亮灯服务，还有另外的服务经人分析不断访问 IO 设备，大量耗电，造成笔记本续航尿崩。

这些软件安装的时候有些还植入了驱动级别的代码，监控了系统的几乎所有消息，不但隐私有问题，而且还很可能造成系统不稳定，说不定 Windows 10 升个级就蓝屏了也托了它们的福。

所以为了省心起见，这类软件必须跟主系统隔绝开来。一开始我是电脑不安装，然后 iPhone 里面装，不过手机输入实在太蛋疼。还是得寻找更好的使用方法。由于平时也要大量使用，就不能只是简单地将它们装到虚拟机里。这样有消息到来也看不到。

后来我想起之前搞过的 RemoteApp 就非常适合这种场景：将这些家伙都装到虚拟机里，然后发布成 RemoteApp，用 RemoteApp 访问它们。

现在使用 RemoteApp 需要虚拟机是 Windows Server，打开声音服务和用户体验桌面，跟 Windows 10 也没什么不一样。然后需要安装`远程桌面服务`。安装远程桌面服务需要虚拟机系统加入域，所以可以直接在虚拟机上也装上域控 `Active Directory 域服务`，这个服务直接在添加角色和功能里面选择，然后按照指引操作即可。安装远程桌面服务和发布 RemoteApp 的过程可以参考 [Step by Step How to Deploy RemoteApp in Windows Server 2016](https://newhelptech.wordpress.com/2017/07/23/step-by-step-how-to-deploy-remote-desktop-services-in-windows-server-2016/)。

发布并登录访问 RdWeb 之后（一般是访问 https://{vmip}/rdweb/），你就能看到上面的 app 列表，使用 chrome 点击 app 就会下载下来对应的 rdp 文件。将这些文件保存下来，更改它们的 `full address:s:` 后面的地址为你虚拟机的 IP 地址（原来是你装域控的时候搞出来的域名，一般解析不了）。再给它们创建快捷方式，快捷方式可以更改图表，找对应的 ico 文件把它们装扮成程序的样子。再把快捷方式复制到 `%ProgramData%\Microsoft\Windows\Start Menu\Programs` 目录，这样你就能在开始菜单发现了它们，右键点击，添加到开始屏幕即可。

第一次使用的时候，点击开始屏幕的图表，输入用户名密码并记住，以后就可以无缝打开并使用了，就好像这个程序是运行在本地一样，图表也会出现在右下角。需要注意的是两个系统最好使用同样的输入法和同样的标题栏颜色，这样就基本感觉不到自己用的是一个远程软件了。