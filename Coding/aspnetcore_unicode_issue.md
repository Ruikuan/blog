# asp.net core Razor 对非英文字符输出编码的问题

## 问题

在 asp.net core 中，这样写代码：

```cshtml
@{
    string title = "标题"; // this is Chinese
}
<title>@title</title>
```
生成 html 后，会变成

```html
<title>&#x4E2D;&#x6587;</title>
```
中文被编码了。虽然浏览器能够识别出里面的中文，显示没有问题，但这样还存在几个问题：
* 本来两个字符，现在变成了一大堆，显著增加了页面的大小。
* 页面对人可读性差，看页面源码时造成困扰。
* 假如你需要跟 javascript 进行配合，直接用 '@title' 输出某个值为 js 某个变量的值，则会造成乱码。

## 原因

在 asp.net core 中，基于防范 xss 攻击的安全考虑，默认将所有非基本字符（U+0000..U+007F）的字符进行编码。因此基本除了英文字母那一部分，其他的全被编码了。

这个控制来源于 `HtmlEncoder` 中的 `UnicodeRange` 被设置成 `UnicodeRanges.BasicLatin`。

## 解决办法

配置将 UnicodeRange 范围放宽。在 Startup 的 ConfigureServices 加入：

```cs
services.Configure<WebEncoderOptions>(options =>
{
    options.TextEncoderSettings = new TextEncoderSettings(UnicodeRanges.All);
});
```
中文就可以正常输出了。

> https://github.com/aspnet/HttpAbstractions/issues/315