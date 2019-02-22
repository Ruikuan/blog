# MikroTik RouterOS 将http请求重定向到https

> 更新：现在看来这个方案弱爆了，完全属于脱裤子放屁，拿着锤子找钉子。更好的办法是给 chrome 装个 Tampermonkey，然后写个脚本将 `http://` 替换成 `https://`

由于墙的存在，访问国外的http网站经常被重置，而且即使不重置，http请求也经常被运营商劫持，弹出个广告什么的，体验非常不好。

普通的访问可以通过使用浏览器访问https站点的方式使用，浏览器也可以收藏https站点，但对于google搜索出来的结果，例如stackoverflow的就结果，是http的，这样访问搜索出来的结果就要断个好几次才能访问到内容，实在是闹心。

因此需要进行一个配置，让特定网站的http访问自动转换为https方式。
实现方式可以通过浏览器扩展和路由器修改来实现。由于Edge当前正式版没有扩展支持，而且扩展方式没有路由器方式应用范围广，路由器设置好的话，所有设备所有浏览器都能够统一进行跳转，因此打算在路由器上动手脚。
现在使用路由器是MikroTik，系统是它家的RouterOS，能够很容易添加功能。

#### 实现思路：
* 路由器上建立一个webproxy
* 设置透明代理将所有http访问站点IP的数据包都转发到这个webproxy上
* webproxy禁止这些请求并返回一个错误页面
* 修改这个错误页面内容，用javascript实现跳转

#### 1. 在RouterOS上面设置一个WebProxy，允许匿名，并点击Reset HTML按钮，将生成默认的error.html，当访问被禁止的时候，会将此error.html的内容返回。
    
设置webproxy
```
[admin@MikroTik] ip proxy> set enabled=yes port=8080
```
![Web Proxy Setting](https://github.com/Ruikuan/blog/raw/master/Content/webproxy_setting.png?raw=true)  
![Web Proxy Error File](https://github.com/Ruikuan/blog/raw/master/Content/webproxy_error_file.png?raw=true)  
需要访问并修改路由器文件的话，开放路由器的FTP功能，使用FTP客户端访问比较方便。

禁用所有到达这个webproxy的访问，目的是将访问内容替换为error.html的内容
```
/ip proxy access add action=deny
```
![deny all](https://github.com/Ruikuan/blog/raw/master/Content/deny_all.png?raw=true)


#### 2. 设置透明代理，将路由器范围内的http访问自动重定向到代理服务器，不用浏览器显式进行代理设置

使用firewall将http请求转发到8080的代理端口
```
[admin@MikroTik] ip firewall nat
[admin@MikroTik] ip firewall nat> add chain=dstnat protocol=tcp dst-address=151.101.65.69 dst-port=80 action=redirect to-ports=8080
```
本来可以将所有请求都转发到webproxy，然后webproxy根据域名进行选择性重定向的，但这样webproxy要处理的请求太多，负荷大；而且访问经过一道中转，延迟大。
因此只将对应站点的访问重定向过去，例如 151.101.65.69 就是 stackoverflow.com 的IP

#### 3. 修改error.html的内容实现跳转

将error.html的内容更改为：
```html
<!DOCTYPE HTML>
<html>
<head>
    <meta charset="UTF-8">
    <script type="text/javascript">
        var url = "$(url)";
        if (url.toLowerCase().indexOf("http:") == 0) {
            url = "https" + url.substring(4, url.length);
            window.location.href = url;
        }
    </script>
    <title>To HTTPS</title>
</head>
<body>
</body>
</html>
```
其中的 $(url) 为路由器提供的变量，表示当前在访问的 url，其实由于路由器发送error.html内容时，不会改变浏览器的访问地址，所以也可以用window.location.href 来获取。
其他的修改就随便自己喜欢了。

#### 4. 将修改后error.html上传，覆盖原来的文件

然后访问http://stackoverflow.com 的任意一个url，都会跳转到https://stackoverflow.com 的对应url了。
https访问的是443端口，不会经过webproxy。

#### 5. Done

结合[将 RouterOS 用作 DNS 服务器](https://github.com/Ruikuan/blog/blob/master/RouterOS/custom_dns.md)使用非常方便。

##### 参考资料

> [Manual:IP/Proxy - MikroTik Wiki](http://wiki.mikrotik.com/wiki/Manual:IP/Proxy#Transparent_proxy_configuration_example)  
> [Howto to enable Mikrotik RouterOS Web Proxy in Transparent Mode](https://aacable.wordpress.com/2011/12/29/howto-to-enable-mikrotik-routeros-web-proxy-in-transparent-mode/)

