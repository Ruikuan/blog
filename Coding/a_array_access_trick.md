# 一个提高 dotnet 数组访问性能的小技巧

在 dotnet 中，`array` 是区分类型的，生成一个 `T[]` 数组，就不能在其中存入不是 `T`（及其子类）的元素。对于引用类型，CLR 会对存入的元素做类型检查。不过由于值类型是不能继承的，所以值类型的数组就不存在这样的检查。可以利用这一点来避免对引用类型数组进行类型检查的开销，从而提高存入性能。虽然数组性能已经很不错了，但有时候性能的这一点提高也有帮助。

我们可以创建一个对引用类型的值类型包装，如下：

```cs
public struct ObjectWrapper
{
    public readonly object Instance;
    public ObjectWrapper(object instance)
    {
        Instance = instance;
    }
}
```

然后使用这个值类型来创建数组，这样在设置数组值 `ObjectWrapperArray[i] = objectWrapperInstance` 时就不会有检查了。性能会有一定的提升。

Roslyn 代码库里面的 `ObjectPool<T>` 也利用了这样的技巧：

```cs
internal class ObjectPool<T> where T : class
{
    [DebuggerDisplay("{Value,nq}")]
    private struct Element
    {
        internal T Value;
    }
 
    // Storage for the pool objects. The first item is stored in a dedicated field because we
    // expect to be able to satisfy most requests from it.
    private T _firstItem;
    private readonly Element[] _items;
 
    // other members ommitted for brievity
}
```