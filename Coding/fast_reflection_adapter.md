# 一个快速反射读取私有字段小窍门（花招）

众所周知，反射很慢，但有时候需要读取私有字段，不采用反射好像没什么好办法。好吧，下面就是一个能快速读取私有字段的小花招。

首先，创建一个跟拥有私有字段的类一样 layout 的类，在这个类中将要读取的私有字段设置为 `public`，然后，参考如下代码：

```cs
class O1 // 我们要读取私有字段的类
{
    private string myPrivateField;
    public O1(string p) => myPrivateField = p;
}

class O2 //我们创建的跟 O1 layout 一样的类
{
    public string publicField;
}

[StructLayout(LayoutKind.Explicit)]
struct FastReflectionAdapter
{
    [FieldOffset(0)] //设置为同样的 Offset
    internal object o1;

    [FieldOffset(0)] //设置为同样的 Offset
    internal object o2;
}

[Test]
public void TestFactReflection()
{
    var adapter = new FastReflectionAdapter{ o1 = new O1( p: "Foo")};
    var privateValueExposed = adapter.o2.publicField; //通过 o2 来获取 o1 的值
    Asset.AreEqual(expected: "Foo", actual: privateValueExposed);
}
```

关键是在 `struct` 的 layout 布局上，把 `o1` 和 `o2` 放到了同一个位置，因此它们实际上读取的是同一个对象。这种技巧只能非常小心地使用，不然很容易出现类型不匹配问题。而且这样利用对象的 layout 技巧，属于利用了内部实现，万一 clr 做了改变，就完蛋了。特别是 clr 的更新并不在这段代码人员的掌控之中，即使有了单元测试，也处理不了这样的问题。