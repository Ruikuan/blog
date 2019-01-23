# 在 C# 中实现只有泛型约束不同的方法重载

从[伟大的 @xoofx 的推特里](https://twitter.com/xoofx/status/1087770199854665729)看到的。 

在 C# 中可以做到同样的方法签名（相同的参数列表，参数类型），而只有对其中泛型约束不同的方法重载。譬如 `Add(Entity e, T data) where T : IXXX` 和 `Add(Entity e, T data2) where T : IYYY`，编译器会根据泛型类型选择不同的方法来执行。

如何做到呢？答案是通过扩展方法，而且每个不同约束的方法要写在不同的 `static class` 里。示例如下：

```cs
/*  我们想暴露出方法：
    1. void Add(Entity entity, ComponentType data)
    2. void Add<T>(Entity entity, T data) where T : IComponentData
    3. void Add<T>(Entity entity, T data) where T : IShareComponentData
*/

public class TypeManager
{
    public void Add(Entity entity, ComponentType data)
    {
    }

    internal void AddComponentData<T>(Entity entity, T data) where T : IComponentData
    {
    }

    internal void AddSharedComponentData<T>(Entity entity, T data) where T : IShareComponentData
    {
    }
}

public static class TypeManagerComponentExtensions
{
    public static void Add<T>(this TypeManager manager, Entity entity, T data) where T : IComponentData
    {
        manager.AddComponentData(entity, data);
    }
}

public static class TypeManagerSharedComponentExtensions
{
    public static void Add<T>(this TypeManager manager, Entity entity, T data) where T : IShareComponentData
    {
        manager.AddSharedComponentData(entity, data);
    }
}
```

是个很 geek 的小技巧，说不定什么地方有用。
