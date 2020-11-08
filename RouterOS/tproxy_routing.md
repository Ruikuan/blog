# 修改透明代理的路由方式

原先透明代理的处理方式是用 DHCP 将所有客户端的网关都设置成透明代理的IP 192.168.0.200。

那么家里所有的客户端的路由方式如下：
```
clientDevice(192.168.0.x) -------> proxyServer(192.168.0.200) -----> MainRouter(192.168.0.1)-----> pppoe
```
这样存在一个潜在的问题，如果 proxyServer 不可用，clientDevice 就没有办法用上网络，而且由于 DHCP 的生效时间限制，即时修改了 DHCP 的 gateway 配置，也要等 DHCP 租期到了才会更新（虽然可以通过插拔网线和断开再重连 wifi 解决 DHCP 的刷新问题）。那么我们为了能够尽快恢复网络，除了手动改变 IP 配置的 gateway 改回 192.168.0.1 之后，没有更好的办法。

因此把网络拓扑改成了下面的情况，内网互联不再通过 proxyServer，外网流量才路由到 proxyServer，同时为了避免 DNS 污染，将所有发送到主路由器的 DNS 请求也路由到 proxyServer。改进后的路由方式如下：

```
clientDevice(192.168.0.x) ---> MainRouter(192.168.0.1) ----> proxyServer -----> MainRouter ----->pppoe

```

这样处理后，万一 proxyServer 挂掉，或者维护，clientDevice 还是能进行内网访问，连上 mainRouter，取消到 proxyServer 的路由，直接从 pppoe 出去即可。
