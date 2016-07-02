# 将 RouterOS 用作 DNS 服务器

国内 DNS 解析充满着污染，而且对于墙外的某些不存在的网站，必须指定IP才能够访问，域名解析出来的IP肯定是不能访问的，因此需要实现自己域名解析。
对于单台机器来说，可以写hosts文件，但设备多了就麻烦，而且手机等设备也不好搞hosts，在路由器上面搭DNS服务器，将hosts文件转到路由器上，所有设备访问网络就比较方便了。

#### 1.首先设置DNS服务器

允许 DNS 服务被访问
```
[admin@MikroTik] ip dns
[admin@MikroTik] ip dns> set allow-remote-requests=yes
```
![DNS 设置 主界面](https://github.com/Ruikuan/blog/raw/master/Content/dns_setting_main.png?raw=true)  
界面里面的动态DNS服务器，是 pppoe 拨号带过来的电信的 DNS 服务器

#### 2.添加自己的域名解析列表

通过命令行添加
```
[admin@MikroTik] ip dns static> add name www.example.com address=10.0.0.1
[admin@MikroTik] ip dns static> add name www1.example.com address=10.0.0.2
```
自己用按照这个格式做个文本，一次性粘贴进去执行就行了。RouterOS也提供 API，可以编程自动添加。这个以后试了再写。

#### 3.在DHCP中将当前路由器作为DNS服务器分发

在IP - DNS Server - Networks 中将 DNS Servers 设置为路由器IP
![DHCP set DNS](https://github.com/Ruikuan/blog/raw/master/Content/routeros_dnsserver.png?raw=true)

命令行
```
[admin@MikroTik] ip dhcp-server network> set 0 dns-server 192.168.0.1
```

##### 参考资料
>[Manual:IP/DNS - MikroTik Wiki](http://wiki.mikrotik.com/wiki/Manual:IP/DNS)  
>[Manual:IP/DHCP Server - MikroTik Wiki](http://wiki.mikrotik.com/wiki/Manual:IP/DHCP_Server#Networks)

