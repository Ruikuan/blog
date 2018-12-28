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
    "$UF729" = moveToBeginningOfLineAndModifySelection:; // shift-home
    "$\UF72B" = moveToEndOfLineAndModifySelection:; // shift-end
}
```
然后注销重新登录，就可以了。
