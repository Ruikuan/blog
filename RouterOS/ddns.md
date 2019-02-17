# RouterOS 使用 3322 动态域名

为了将内网的服务提供出去，需要申请一个动态域名。我是在 pubyun（即以前的 3322.org） 申请的免费动态域名。    

申请之后，可以通过 [pubyun 提供的 api](http://www.pubyun.com/wiki/%E5%B8%AE%E5%8A%A9:api) ，在 pppoe 获得的 ip 改变之后，动态更新域名的 ip。    

更新 IP 的脚本如下：
```
:local ednsuser "{username}"
:local ednspass "{password}"
:local ednshost "{example.f3322.net}"
:local ednsinterface "{pppoe-out1}"
:local members "http://members.3322.org/dyndns/update?system=dyndns"
:local status [/interface get [/interface find name=$ednsinterface] running]
:if ($status!=false) do={
:local ednslastip [:resolve $ednshost]
:if ([ :typeof $ednslastip ] = nil ) do={ :local ednslastip "0" }
:local ednsiph [ /ip address get [/ip address find interface=$ednsinterface ] address ]
:local ednsip [:pick $ednsiph 0 [:find $ednsiph "/"]]
:local ednsstr "&hostname=$ednshost&myip=$ednsip"
:if ($ednslastip != $ednsip) do={
:local result [/tool fetch url=($members . $ednsstr) mode=http user=$ednsuser password=$ednspass as-value output=user]
:log info ($ednshost . " " .$result->"data")
}
}
```
主要的逻辑就是检查 pppoe-out1 的 ip 跟当前域名 example.f3322.net 解析出的ip是不是一致，不一致的话，就用 api 更新域名的 ip。  

脚本放在 system - scripts 里面，命名为 DDNS ，这个脚背通过计划任务隔一段时间执行一次，一般还是不少于10分钟吧。计划任务在 system - scheduler 里面增加， OnEvent 里面填
```
DDNS
```
即可，DDNS 是上面定义的脚本的名称。  

经过测试，能够顺利更新动态域名的 IP。

#### 备注

命令
```
/tool fetch url=$url mode=http dst-path=$ednshost
```
只有在服务器返回有内容的响应才会执行成功，后面的代码才能继续运行。如果服务器返回一个内容为空的 200 响应（不是出错），这句也是执行不成功。

#### 参考
> [DDNS动态域名脚本（花生壳+3322公云）for ROS 6.x](http://www.roszj.com/526.html)  
> [如何在ROS中设置花生壳服务](http://service.oray.com/question/869.html)