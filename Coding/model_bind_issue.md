# Asp.net core model bind 的一个问题

项目中有个 model 类 A，开始使用是没什么问题的。后来为了方便 view 里面根据状态进行界面控制，给它加了个只有 get 的属性 A，代码示例如下：

```csharp
class A
{
    public bool CanDelete()
    {
        //....
        if (someObject.IsOver()) return false;
        //....
    }
}
```

而这个 someObject 在我的所有代码路径中， CanDelete 被调用的时候，一定是存在一个非空的值的。但在 post 到一个 action 的时候，页面抛出了 NullReference 异常。检查了几遍代码也不得其解。

考虑到这个 model 是那个 action 的一个参数，框架会自动对它进行绑定，因此推断它在绑定的时候自作主张访问了 CanDelete 的值，由于绑定的时候，someObject 还没有存在，因此出现空引用。

```csharp
[HttpPost]
public IActionResult CreateA(A model)
{
    if (ModelState.IsValid)
    //....
}
```

解决方法，由于框架因为绑定的目的，只会读取对应 model 的属性，因此将其更改为方法，问题解决。

```csharp
public bool CanDelete()
{
    //....
    if (someObject.IsOver()) return false;
    //....
}
```

看来框架干的事情比想象中的多，之前还以为绑定的时候，只需要查看属性类型和调用对应属性的 set 方法呢，没想到连 get 也读。这也提醒了我以后在 model 中非绑定的内容的话，最好用方法，而不是属性。