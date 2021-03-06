# VPN 客户端使用家里的透明代理

家里设置一个透明代理服务器网关（192.168.0.200）之后，DHCP 把所有设备的网关都指定为 192.168.0.200，当然代理网关自己的网关 IP 指定为 router 所在的 192.168.0.1，不然不能上网。设置了透明网关之后，家里的设备都能无感知地无障碍上网了。

根据之前的文章 [RouterOS 上设置 L2TP VPN](./l2tp_vpn.md) 的文章设定 VPN 服务器之后，使用 iPhone 连接上 VPN，发现它还是没办法访问某些站点，手机上测试下路由轨迹，发现它没有经过代理网关服务器 0.200，直接从 0.1 路由器就出去了。这就奇怪了。

翻查一下资料，MikroTik 论坛上面有人说 DHCP 并不会作用在 VPN 客户端上，L2TP 只是纯粹根据设定的 IP Pool 把网络地址设置上去而已，其他的设定基本没有的。如果要设置路由，得在客户端上面设定路由走向。

这就太麻烦了。想了一下，既然它的数据包已经到了 0.1，而它自己不会再走到 0.200，那么我们在 IP —— Firewall —— Mangle 里面设置一条 prerouting 的规则，把源地址是那个地址池的所有 IP，mark routing 成 toproxy；然后 IP —— Routes 里面加一条到所有目标地址，标记为 toproxy，gateway 是 192.168.0.200 的路由就行了。这样就能够强制把所有 VPN 客户端的所有流量全都跑到透明代理网关上去。

测试一下，果然成功了。现在在外面也可以用手机轻松无障碍了。