# 解决 `Package Cache` 越来越大的问题

微软系的开发者的 C 盘铁定是不够用的，即使把 visual studio 安装在别的分区，C 盘的使用空间还是不断增长，再加上 dotnet core 各个版本还有其他相关的开发工具等一系列全家桶装下来，C 盘空间每况愈下捉襟见肘。

微软系的开发相关的软件的安装包，绝大部分都是采用 WiX 的安装工具链 Burn 来构建的。这个工具链生成的安装包，在软件顺利安装之后，为了保证以后修复、重新安装或者卸载等动作的顺利进行，会把安装程序包也塞到 `%ProgramData%\Package Cache` 目录下，即使软件安装的目标目录不是 C 盘也是如此，所以 C 盘就这样没有节制地膨胀起来。为解决这个问题，就需要想办法将 `%ProgramData%\Package Cache` 里面地那一坨东西放到其他盘，并且让应用知道到其他地方找这些玩意。

## 基于注册的策略重定向

新版本的 WiX 工具支持基于注册的策略重定向，即先注册一个重定向策略，将另一个位置设置为 Cache 的根目录，以后的软件 update 和安装都会使用新的根目录，不会再写进 `%ProgramData%\Package Cache`，从而很好地避免 C 盘膨胀的问题。

进行注册在提升权限的命令行中输入下面命令即可：

```
reg.exe add HKLM\Software\Policies\WiX\Burn /v PackageCache /d {X:\PackageCache}
```

其中 `{X:\PackageCache}` 替换为自己实际需要放置 Cache 的目录。

需要注意的是，在执行注册之前安装的程序，还是会使用原来的 `%ProgramData%\Package Cache` 放东西，而且执行注册不会自动将这些安装包自动转移过去，手动转移过去也是不行的。注册动作只会影响以后的行为。也就意味着如果你想将 visual studio 的安装包换个地方，那还得先卸载了再重新安装一次。当然头脑正常的人都不会这样做。

## `mount` 一个 vhd 虚拟硬盘硬件到 `%ProgramData%\Package Cache`

主要的操作思路是：

1. 将 `%ProgramData%\Package Cache` 的内容复制到其他地方譬如 `D:\PackageCache`
2. 将 `D:\PackageCache` 制作成 vhd 虚拟硬盘文件
3. 将 vhd 文件 mount 到 `%ProgramData%\Package Cache`，这样访问 `%ProgramData%\Package Cache` 实际上访问的就是 vhd 文件空间了。

这个方法比较复杂，详细可以参考 [How to relocate the Package Cache](https://blogs.msdn.microsoft.com/heaths/2014/02/11/how-to-relocate-the-package-cache/)。

## 使用 `Junction` 目录来重定向

主要操作思路类似上面 mount 的，但不需要复杂的命令制作 vhd 文件和 mount。基本步骤如下：

1. 将 `Package Cache` 从 `C:\ProgramData` 移动到 `D:\`
    ```
    move "C:\ProgramData\Package Cache" "D:\"
    ```

2. 创建符号链接
    ```
    mklink /j "C:\ProgramData\Package Cache" "D:\Package Cache"
    ```

在 `C:\ProgramData` 下就出现了一个 `Junction` 目录，链接到 `D:\Package Cache`。程序读写 `C:\ProgramData\Package Cache` 中的内容实际上读写的是 `D:\Package Cache` 中的东西，但这个链接是文件系统上面的抽象，访问过程对于系统和应用是透明的，也就是系统和应用会以为一直有一个正常的 `C:\ProgramData\Package Cache` 在那里。

这个方法的问题在于 WiX 安装包卸载时不认识 `Junction` 目录，卸载时会把 `C:\ProgramData\Package Cache` 这个 link 删掉（虽然 `D:\Package Cache` 的内容还在，删除的只是 link，而不是内容），这样就导致后续系统和应用尝试使用 `C:\ProgramData\Package Cache` 时会出问题。目前没有特别好的解决方法，只好在卸载之后再创建一次关连。

## 我自己怎么搞呢？

首先，还是要使用最新的基于注册策略重定向功能来实现目的，这个功能是工具提供方特地提供的，就是为了解决 `Package Cache` 占空间的痛点，支持是最好的。  

另外，由于这个方法不究以往，因此我同时还创建一个 `Junction` 链接到上面重定向的目标目录（将旧的内容都先复制过去），这样，不管是先前的安装包还是后面的安装包，都能放到同一个目录中，而且都能正常找到了。只需要在卸载程序的时候留意下 link 有没有被破坏，破坏了就重新创建一下就好了。用起来之后，肯定是旧的东西越来越少，新的东西全都走注册重定向，link 被删除的可能性就越来越小了。

## 号外

上面几种方法，除了基于注册策略的，其他两种可以用于转移 `C:\Windows\Installer` 目录，这个目录也巨大无比。其他超大的目录也可以视情况使用。

> 参考  
> [Redirect the Package Cache using registry-based policy](https://blogs.msdn.microsoft.com/heaths/2015/06/09/redirect-the-package-cache-using-registry-based-policy/)  
> [How to relocate the Package Cache](https://blogs.msdn.microsoft.com/heaths/2014/02/11/how-to-relocate-the-package-cache/)