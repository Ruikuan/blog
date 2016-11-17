# 解决控制面板设备和打印机页面打开非常缓慢的问题

老婆的笔记本打开控制面板的设备和打印机页面非常缓慢，每次打开这个页面，需要花费至少3分钟才能显示内容，而且这段时间里 cpu 使用率飙升，风扇呼呼作响。 cpu 强劲的话，打开速度会快点，但风扇声依然可观。  
查看 eventvwr 没有相关信息，尝试将可能涉及的 windows 服务都禁用，也不起作用，将里面的设备，包括 Office 自带的 XPS printer，传真之类都删除，只剩下一个打印机，问题照旧。将多媒体设备全删掉，仍未解决。删掉打印机驱动再重装也不行。  
不得已， bing 了一下，遇到类似问题的人很多，通常使用 Windows 8 或者 Windows 10，微软自己的问答网站也有，但微软的回答没什么用。一般说法是涉及蓝牙设备，服务里面将蓝牙服务设置为自动启动就行了。其实是没什么用的。  
再仔细翻，有个人提及他经过一项项排查，发现可能是 Realtek 声卡驱动的问题，他装了旧版本的驱动没有问题，更新之后就出事了。受到启发，验证了一下，果然是 Realtex 声卡驱动的问题！真挺莫名其妙的，感觉风马牛不相及的问题啊。
![evil driver](https://github.com/Ruikuan/blog/raw/master/Content/audio_driver.png)

### 解决方法

打开设备管理器，声音、视频和游戏控制器里面找到 Realtek 声卡，右键，卸载，弹出卸载面版，选中删除此设备的驱动程序，确定。然后这个声卡就被删掉了。    
![uninstall driver](https://github.com/Ruikuan/blog/raw/master/Content/uninstall_audio.png)  

这时再扫描硬件改动，系统就会将声卡加回来，并且装上了微软自家的驱动。问题解决。现在声卡能正常使用，设备和打印机页面也秒开了。
![good driver](https://github.com/Ruikuan/blog/raw/master/Content/good_driver.png)
若过段时间驱动自动更新回 Realtek 家的，在驱动程序 tab 将驱动回滚到上一个版本，即微软的版本即可。微软的驱动还能自动区分扬声器和耳机分别设置音量， Realtek 的还没办法区分，版本还老，还好意思瞎更新。

单纯禁用 RealTek 声卡也能使面版打开加快。

之前看到 Realtek 的声卡也是导致了另一个诡异问题 [播放音乐导致计算机 cpu 性能越变越差](https://zhuanlan.zhihu.com/p/23337819) 的罪魁祸首，这家的驱动还真是不靠谱呢。