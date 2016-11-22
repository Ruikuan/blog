# 解决 RB450G 内网访问达不到千兆的问题

这几天架好家里的服务器，发现用笔记本通过路由器（rb450g）复制文件到服务器上，速度最多只有 50MB/s，而且速度不稳定，在 20MB/s ~ 40MB/s 之间来来回回，远远达不到千兆的性能。用笔记本接网线直连服务器复制文件，则轻松达到 110MB/s 的满速（两者都是千兆网卡）。通过路由器复制文件时看路由器的 cpu 使用率也不超过 50%，百思不得其解。google了下，也没发现什么有用资料，反而看到[它自家的测试](https://routerboard.com/rb450g)，速度确实达不到千兆。  

本来想算了，另外买一个千兆交换机吧，TP-Link 的五口千兆交换机也不过99块。不过多买个设备又要多接电，又要想办法塞到哪个角落里面，比较麻烦。随手到淘宝买 rb450g 店铺维护的那个网站上面翻了下，发现一个帖子讲怎么在 RouterOS 里面设置端口的交换功能，眼前一亮，进去一看，果然是我要找的功能。  

之前为了能让家里的设备都在一个网段，只能将所有内网端口都串在一个 bridge 上，然后将 DHCP 设在这个桥上，因此复制文件的时候，数据都在软桥上面走，路由器可能压力比较大。现在学习了交换功能，撤掉了桥，将内网端口设置在同一个交换（将 3/4/5 端口串在 2 上），并且将 DHCP 设置在端口 2 上，既保证了网段一致，又剩下了桥接的开销。设置完成之后，内网机器复制文件轻松达到了 110MB/s 而且非常稳定。

### 设置方式如下：
如下图，我们可以看到一个 RB750 的5 个以太网接口，我们需需要将ether3、ether4 与ether2 进行交换功能配置：  
![端口列表](https://github.com/Ruikuan/blog/raw/master/Content/etherlist.jpg)  

打开 ether3 和ether4 接口配置，并将Master Port 为ether2  
![设置](https://github.com/Ruikuan/blog/raw/master/Content/switchsetting.jpg)  

设置完成后，我们可以在 interface 列表中看到
![设置完成](https://github.com/Ruikuan/blog/raw/master/Content/finish.jpg)

  
##### 参考资料  
>[[基础] RouterBOARD 交换功能配置](http://bbs.router.com.cn/thread-45971-1-1.html)