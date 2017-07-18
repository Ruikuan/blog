# Asp.net core 的一个 model binding 问题

在做一些简单的基础数据 CRUD 时，出了个怪问题：有个 post 的 action 的参数无论如何都没法绑定上。

方法的签名大概如下：

```csharp
public async Task<IActionResult> EditProductCatalog(ProductCatalogModel model)
{
    if (ModelState.IsValid)
    {
        // ……
    }
}
```
ModelState 没有问题，但 model 虽然不是为 null，但里面的所有属性都是默认值。也就是简单地 new 了一个对象出来而已。而其他类似的方法都能绑定成功。

找了一下原因，发现是是因为 ProductCatalogModel 有个属性叫 Model，估计是因为这个原因，Binder 没有能够正确的区分 action 方法参数的 model 和对象属性里面的 Model，导致绑定出错。

```csharp
public class ProductCatalogModel
{
    // ……
    public string Model { get; set; }
    // ……
}
```

解决方法：将方法前面里面的参数名从 model 改成 m 即可。
