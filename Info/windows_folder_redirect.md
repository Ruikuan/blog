# 新安装 Windows 如何保持 C 盘精简？

作为微软系的开发者，C 盘经常会被各种工具作为默认存储空间而塞满。那么新安装 Windows 之后，要进行哪些设定，让 C 盘不至于三两下就被撑爆了呢？


## 将 `%ProgramData%\Package Cache` 目录移到其他分区

参见 [解决 Package Cache 越来越大的问题](../Coding/solve_package_cache.md)

```
reg.exe add HKLM\Software\Policies\WiX\Burn /v PackageCache /d {X:\PackageCache}
```

## 将 Nuget 相关目录转移到其他分区

参见 [Managing the global packages, cache, and temp folders](https://docs.microsoft.com/en-us/nuget/consume-packages/managing-the-global-packages-and-cache-folders)，分别设置 Nuget 对应的环境变量对应的目录。

```
setx NUGET_PACKAGES "D:\nuget\packages"
setx NUGET_HTTP_CACHE_PATH "D:\nuget\v3-cache"
setx NUGET_PLUGINS_CACHE_PATH "D:\nuget\plugins-cache"
```

## 将 `C:\Windows\Installer` 目录移到其他分区

先将 `Installer` 目录剪切到其他目录，例如 `D:\Installer`，然后删除 `C:\Windows\Installer` 并使用 `mklink` 建立符号链接

```
mklink /j "C:\Windows\Installer" "D:\Installer"
```

必须建立符号链接，否则安装卸载程序的时候有可能出错。

## 将 `Microsoft SDKs` 目录转移到其他分区

`Microsoft SDKs` 在 `C:\Program Files` 中以及 `C:\Program Files (x86)` 中，如果需要开发 Windows 应用程序譬如 UWP 之类的，对应 Windows 版本的 SDK 会放置到对应的目录。也是通过上面建立符号链接的方式把它挪走即可。

## 将 OneDrive 目录设置到其他分区

在 OneDrive 自己的设置界面设置即可。

## 将 Downloads 目录设置到其他分区

Downloads 和 我的文档 之类的，现在在 Windows 中都表现成了库，右键点击库，选择更改位置设定即可。


## 注意事项

采用上文建立符号链接来转移的目录，在升级 Windows 的年度功能更新之后，譬如更新 1903 之类的大版本，需要重新设置。