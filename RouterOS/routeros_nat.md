# RouterOS 中的 NAT

前几天琢磨 [**如何将外网请求重定向到内网**](https://github.com/Ruikuan/blog/blob/master/RouterOS/redirect_to_lan.md) 时碰到了不少困难，还是自己对 NAT 了解得太少导致的。这两天认真学习了 RouterOS 里面的 NAT 知识，将我对它的理解写在这里。  

### Destination NAT | DST-NAT

dst-nat 是重写 ip packet 的 dst-address 和 dst-port 的操作。  
进入路由器的包，使用这个操作，可以改写包的目标地址和端口，改变包的路由方向。  

假如有一个 ip 包如下：  
| |Address|Port|
| ------ | ------ | ------ |
|**Source**|192.168.0.23|6789|
|**Destination**|8.8.8.8|80|

对这个包应用 dst-nat，将它的目标地址改为 192.168.0.14，目标端口改为 8080
``` 
add chain=dstnat dst-address=8.8.8.8 protocol=tcp dst-port=80 \
  action=dst-nat to-address=192.168.0.14 to-port=8080
```
经过 dst-nat 之后，这个包就变成了  
| |    Address    | Port |
| ------ | ------ | ------ |
|   **Source**   | 192.168.0.23  | 6789 |
|**Destination** | 192.168.0.14  | 8080 |

拥有 IP 192.168.0.14 的设备将会收到这个包。  

为了能够应对将来到来的回复包，RouterOS 会记录这次 dst-nat 对应的转换映射如下：  
| |Original|Map|
| ------ | ------ | -------  |
|**Address**|8.8.8.8|192.168.0.14|
|**Port**|80|8080|

这样，当回复包经过路由器的时候，路由器会检查回复包的 Source Address 和 Port，有对应映射记录的话，会将 Source Address 和 Port 改变为映射前的地址和端口。  
（注意：这里暂不考虑回复包不经过路由器的情况，我们假设所有的包都经过路由器。）  

回复包  
|                |    Address    | Port |
| ------ | ------ | ------ |
|   **Source**   | 192.168.0.14  | 8080 |
|**Destination** | 192.168.0.23  | 6789 |
被根据映射端口记录修改源地址和端口，变为：
|                |    Address    | Port |
| ------ | ------ | ------ |
|   **Source**   |    8.8.8.8    |  80  |
|**Destination** | 192.168.0.23  | 6789 |
  
这样，发送端就能顺利收到回复包，两端能够进行顺利的信息交换了。

这个过程如图所示：  
![dst-nat](https://github.com/Ruikuan/blog/raw/master/Content/dst_nat.png)

### Source NAT | SRC-NAT 

src-nat 是对即将离开路由器的包，重写包的源地址和源端口的操作。  

通常在家庭网络中，路由器对外只有一个 IP，假设这个 IP 为 7.7.7.7，而路由器内网的设备有很多，有多个内网 IP，有个类似 192.168.0.0/24 这样的网段。内网的设备 A 要和外网的服务器进行信息交换，只能够借助路由器的外网 IP 来进行，因为外部网络并不知道如何将目标地址为 192.168.0.1 的包发送到何处，这个包自然没办法回到目标设备 A 了。  
因此，需要使用 src-nat 操作对内网准备出外网的包进行源地址重写，将源地址重写为 7.7.7.7，这样外部网络回复包的时候，会顺利将包发送到拥有公网 IP 7.7.7.7 的路由器上，路由器再根据之前记录的映射，将回复包的目标地址设置为内网的某 192.168.0.x IP，将它传递到目标设备上。  

假如路由器收到一个包如下：  
| |Address|Port|
| ------ | ------ | ------ |
|   **Source**   | 192.168.0.23  | 6789 |
|**Destination** |    8.8.8.8    |  80  |

为了能顺利收到服务器 8.8.8.8 的回复包，路由器(公网 IP 为 7.7.7.7)对它应用 src-nat：
```
add chain=srcnat src-address=192.168.0.0/24 \
  out-interface=wan action=src-nat to-address=7.7.7.7
```

即对所有经过 wan 端口出去互联网的包，都将它的源地址改为 7.7.7.7，端口不设置，让路由器随机选择一个端口，假设随机选择了 9876。经过转换，包变成了：  
|                |    Address    | Port |
| ------ | ------ | ------ |
|   **Source**   | 7.7.7.7  | 9876 |
|**Destination** |    8.8.8.8    |  80  |

路由器记录这次转换映射如下：  
| |Original|Map|
| ------ | ------ | -------  |
|**Address**|192.168.0.23|7.7.7.7|
|**Port**|6789|9876|

服务器 8.8.8.8 对该包进行回复，回复包的目标地址是 7.7.7.7，这样路由器就能顺利收到这个回复包了。路由器再根据原先的映射记录将包的目标地址转化为 192.168.0.23，目标端口转化为 6789，就不再赘述了。


除了从 WAN 出去的包，从 Lan 口出去的包也可以改写，其实从任何一个口出去的包都可以做 src-nat，不过为了保证包能够回到路由器，从 lan 口出去的包需要设置 to-address 为路由器的内网 IP，如 192.168.0.1 这样。

整个过程如图所示：  
![src-nat](https://github.com/Ruikuan/blog/raw/master/Content/src_nat.png)

### Masquerade

masquerade 属于 src-nat 的一种，是 src-nat 的特殊转化，它的不同在于，它会根据包出去的 interface，自动将包源地址转化为对应 interface 所属网络的路由器 IP，端口随机，也即 masquerade 的转换是智能化设置的，起到一个简化设置的作用。其他功能和处理方式跟 src-nat 是一样的。



#### 参考
> [Manual:Packet Flow](http://wiki.mikrotik.com/wiki/Manual:Packet_Flow)  
> [Manual:IP/Firewall/NAT](http://wiki.mikrotik.com/wiki/Manual:IP/Firewall/NAT)  
> [Hairpin NAT](http://wiki.mikrotik.com/wiki/Hairpin_NAT)