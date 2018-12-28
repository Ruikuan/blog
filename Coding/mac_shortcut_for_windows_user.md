# 在 Mac 系统下使用 Windows 习惯的快捷键

首先通过 `System Preferences - Keyboard - Modifier Keys` 修改修饰键为如下：

![modifier keys setting](https://raw.githubusercontent.com/Ruikuan/blog/master/Content/modifierKeysSetting.png)

这样就可以像在 Windows 下一样使用熟悉的 Ctrl A，Ctrl C 和 Ctrl V 了。

然后，对于文本编辑用户或者码农，Mac 的 Home 和 End 键是直接跳转到文档的开头和结尾，也是令人崩溃的。要想它像 Windows 下那样跳到段头和段尾，需要像下面这么设置。

创建或修改 `~/Library/KeyBindings/DefaultKeyBinding.dict` 文件，输入如下内容并保存：

```json
{
    "\UF729" = moveToBeginningOfLine:; // home
    "\UF72B" = moveToEndOfLine:; // end
    "$\UF729" = moveToBeginningOfLineAndModifySelection:; // shift-home
    "$\UF72B" = moveToEndOfLineAndModifySelection:; // shift-end
    "@\UF729" = moveToBeginningOfDocument:;
    "$@\UF729" = moveToBeginningOfDocumentAndModifySelection:;
    "@\UF72B" = moveToEndOfDocument:;
    "$@\UF72B" = moveToEndOfDocumentAndModifySelection:;
}
```
然后注销重新登录，就可以了。

### 附录
|Character code|Description|
|----------- | ----------- |
|\UF700	|Arrow up|
|\UF701	|Arrow down|
|\UF702	|Arrow left|
|\UF703	|Arrow right|
|\UF728	|Forward delete|
|\U007F	|Backward delete|
|\UF729	|Home|
|\UF72B	|End|
|\UF72C	|Page up|
|\UF72D	|Page down|
|\U001B |	Escape|
|\U0009	|Tab|
|\U0019	|Backtab (shift tab)|
|\U000D	|Return|
|\U0003	|Enter|

|Modifier Key Code|Description|
|----------- | ----------- |
|$|Shift|
|@|Command|
|^|Control|
|~|Option|
 
默认的系统映射文件在这里：`System ▸ Library ▸ Frameworks ▸ AppKit.framework ▸ Resources ▸ StandardKeyBinding.dict`
