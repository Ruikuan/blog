# Razor 预编译与 Partial View 搜索

今天在访问发布系统的适合，打开某个页面，系统报异常：

```
The partial view '_OnetimeServiceEditInfo' was not found
```

之前都是用得好端端的，在本地开发环境一试，却又一切正常，没有任何问题。

系统是用 asp.net core 1.1 开发的，本机环境跟发布环境不同之处在于发布环境的 view 都是通过预编译生成了一个 dll 文件直接部署的。想到这点，我猜想是不是 dll 里面没有包含 _OnetimeServiceEditInfo 这个 view？用 ILSpy 看了下，并不是这个原因，这个 view 的确已经编译进去了。我又检查了下本地的 view 文件，view 也是存在的。

莫非 asp.net core 的预编译有 bug？一般情况下我是不会有这种想法的，因为这些框架的代码质量比我们这些个人的代码质量靠谱得多。不过之前在用 asp.net mvc 的时候，我的确遇到过不能通过 VirtualPathProvider 使用 precompiled view 的问题，所以产生了怀疑。google 了下，没发现什么有价值的信息。

那肯定是自己的问题！再回头仔细看了下使用这个 partial view 的页面代码，双击 _OneTimeServiceEditInfo.cshtml 文件名，复制左边，粘贴到 @Html.Partial 里面，一看，没有任何区别啊。不管，先保存，咦，文件内容居然有改动！比较一下，才发现，有个大小写的问题。本来应该是 

```cshtml
@Html.Partial("_OneTimeServiceEditInfo", Model)
```

的，写成了 

```cshtml
@Html.Partial("_OnetimeServiceEditInfo", Model) 
```

T 和 t 混淆了。

而原因就很清晰了，在没有预编译的情况下，ViewEngine 寻找对应 view 的适合，是**不**区分大小写的。而使用预编译之后，ViewEngine 寻找 view 时，就区分大小写了。所以造成了行为不一致。开发的时候发现不了。而且之前发布的环境也没有问题，搞不清楚是不是因为升级某个版本之后，行为才改变了。虽然的确是自己代码的 bug，但升级真的还是有风险哪。

看来的确应该用 container 来开发、测试和部署的环境差异问题，确保环境一致，以免出现这种环境不一致排查问题困难的情况。