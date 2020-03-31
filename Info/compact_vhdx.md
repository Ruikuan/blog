# 压缩 vhdx 文件

vhdx 文件按需扩展大小，但如果删掉里面的文件，处于性能的考量，它并不会随之变小。有时候有很多空余空间的 vhdx 文件的大小相当庞大，需要压缩一下。

可以使用如下 powershell 命令压缩。执行这些命令的前提是安装了 hyper-V 的所有 role.


```
Mount-VHD .\dyndisk.vhdx -ReadOnly
Optimize-VHD .\dyndisk.vhdx -Mode Full
Dismount-VHD .\dyndisk.vhdx
```