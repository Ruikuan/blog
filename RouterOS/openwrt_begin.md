# Hyper-V OpenWRT 折腾小记

现在的路由器用的是 RouterOS 的 RB450g ，虽然稳定可靠，但不开源不支持定制，有很多功能不能够很好支持，所以想虚拟出一个 OpenWRT 来，用它实现一些别的功能，取长补短。其实用普通的 linux 应该也可以，但既然都是折腾，OpenWrt 体积小，需求低，直接折腾 OpenWRT 也没什么不好。  

OpenWRT 对 Hyper-V 的支持不好，都是万恶的反微软浪潮搞的。而我对 linux 阵型一向不怎么懂，就在 [让OpenWRT完美适应Hyper-V ](https://soha.moe/post/make-openwrt-fits-hyperv.html) 求了一个 vhd 来开启了折腾之旅。建一个虚拟机，将 vhd 附加进去，就顺利跑起来了。  

### 网络配置

虽然虚拟机跑了起来，但它默认的网络配置不适合我现在的网络，需要将它的 dhcp 关掉，然后它自己也要从 lan 里面的 dhcp 获取 ip，修改如下：

/etc/config/network  
```
config interface 'loopback'
	option ifname 'lo'
	option proto 'static'
	option ipaddr '127.0.0.1'
	option netmask '255.0.0.0'

config interface 'lan'
	option type 'bridge'
	option proto 'dhcp'
	option ifname 'eth0'

config globals 'globals'
	option ula_prefix 'fdfa:2d07:0cfd::/48'
```  

删掉了 wan 端口的配置，因为用不上

/etc/config/dhcp
```
config dnsmasq
	option domainneeded '1'
	option boguspriv '1'
	option localise_queries '1'
	option rebind_protection '1'
	option rebind_localhost '1'
	#option local '/lan/'
	option domain 'lan'
	option expandhosts '1'
	option readethers '1'
	option leasefile '/tmp/dhcp.leases'
	option resolvfile '/tmp/resolv.conf.auto'
	option localservice '1'

config dhcp 'lan'
	option interface 'lan'
	option start '100'
	option limit '150'
	option leasetime '12h'
	option dhcpv6 'server'
	option ra 'server'
	option ignore '1' 

config dhcp 'wan'
	option interface 'wan'
	option ignore '1'

config odhcpd 'odhcpd'
	option maindhcp '0'
	option leasefile '/tmp/hosts/odhcpd'
	option leasetrigger '/usr/sbin/odhcpd-update'

'/usr/sbin/odhcpd-update'
```

主要是用 option ignore '1'  将 dhcp 禁用

修改完之后，重启 network 和 dnsmasq
```
/etc/init.d/network reload
/etc/init.d/dnsmasq restart
```

这样 OpenWRT 就能正确获取到 ip，能通过 ssh 访问了。不过访问前先要设置密码：

```
passwd
```

### 配置安装包的源

通常需要安装 luci web 管理界面，还有其他杂七杂八的东西，用 opkg 安装比较方便。但这个版本基本上什么东西都安装不了，因为没有配置源。配置源的方式如下：

/etc/opkg/customfeeds.conf
```
# add your custom package feeds here
#
# src/gz example_feed_name http://www.example.com/path/to/files
src/gz custom_feed http://downloads.openwrt.org/chaos_calmer/15.05/x86/generic/packages/luci
src/gz packag_feed http://downloads.openwrt.org/chaos_calmer/15.05/x86/generic/packages/packages
src/gz basepa_feed http://downloads.openwrt.org/chaos_calmer/15.05/x86/generic/packages/base
```
然后就可以执行命令安装东西了。
```
opkg update
opkg install xxx
```
由于编译的内核版本跟官方版本不是完全一致，有时候有些东西提示依赖不对，装不上，这个时候就可以强制忽略依赖问题安装
```
opkg install xxx --force-depends
```

我安装了vsftpd luci 等，安装 vsftpd 可以很方便地操作 OpenWRT 里的文件，配置文件也可以先拖出来，用 vscode 等编辑了，再拖回去，比 vim 等爽太多。

### 安装二进制文件

直接将文件用 ftp 传到上面，敲命令运行就行，要自动运行就看下面一节。

### 配置自动启动

貌似跟普通的 linux 不一样，自动启动的东西都放在 /etc/init.d/ 下面，里面的脚本拉 vsftpd 下来瞧瞧对照着写就行。写好之后，将脚本文件放到目录下，执行如下命令将它设置成可执行脚本并设置自动启动。
```
chmod +x xxxApp
xxxApp enable
```
然后开机之后，这个 xxxApp 就自动运行了。

### 配置 OpenVPN

很失败，没配置成功。安装是没问题的，但这个版本好像少了 TUN/TAP 模块，导致运行 tap 模式的 OpenVPN 没法使用。对我就没有什么意义了。可能需要重新编译内核，但我翻不了墙也没什么办法编译了。

### 配置 dns 

本来设想它作为 dns 服务器，然后它上层是 RouterOS 上的 dns 服务器的，但配置几次都失败，总是解析不到 RouterOS 上面设定的 dns，先放弃。