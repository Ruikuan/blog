# asp.net core 1.1 Razor taghelper intellisense 启用办法

从 taghelper 加入到 asp.net mvc 中，在 visualstudio 中编辑 view 时就支持 intellisense。例如，当你在一个 &lt;a> 中输入 asp- 时，它会自动列出 a 对应的 taghelper 供你自动补全，是一个挺方便的功能。  
![taghelper intellisense](https://github.com/Ruikuan/blog/raw/master/Content/taghelper.png)

问题是自从开始进入 asp.net core 升级到 1.0 之后，这个 intellisense 就没有了，在 visualstudio 2015 update3 中没有，为这个我还升级到了 visualstudio 2017 RC3，还是没有，等 2017 正式版出来，也还是没有。翻看一下 taghelper github 的 issue，开发团队说还没有搞好，正在努力开发中。看来靠不住了。虽然没有它也能照常开发，事实上我项目中的绝大部分都是在没有的情况下开发出来的，但有这东西能省不少力气，而且一眼看过去就知道是不是拼写错了，符合 taghelper 的属性是另一种颜色。现在只能靠肉眼看，写错了不管编译还是运行都不会有什么错误，只是生成的 html 不是预想中的，这样就给找问题带来了麻烦。

vs2017 发布的时候，微软给了一个 walkthrough，说可以通过一个 Razor Language Extension 的扩展解决这个问题。但安装完之后，我的项目仍然没有 intellisense，这就奇怪了。

我另外新建一个 web 项目，在上面测试了下， taghelper 的 intellisense 工作的很不错！  

没办法，我首先对照下两者的 csproj 文件，看看是不是当前项目是旧版本 xproj + project.json 升级上来的有什么问题。根据新项目对现有项目删掉一些项增加一些项之后，情况没有改善。

为什么呢？我将 cshtml 文件关闭，再打开，不行；关闭，重新编译，再打开，还是不行。我不禁陷入了深思。

这时候，我想起之前对 _ViewImports.cshtml 做了些更改，给它加了一句引入我自己自定义的 taghelper（其实到现在我还没有实现一个自定义的 taghelper 呢，只是事先占个坑）。

```
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
@addTagHelper *, MyProject
```

是不是那个添加自己的 taghelper 语句有问题呢？毕竟我还没有 taghelper。将它删掉看看。

删掉之后，果然 taghelper 立刻正常了！终于可以又用上它的 intellisense 了。以后真加入自己的 taghelper 再说了。