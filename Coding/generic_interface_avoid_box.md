# 如何避免值类型通过泛型接口使用导致装箱

在 .NET 中，将 valueType 转化为接口会导致它装箱。那么如何避免装箱，又能使用 interface 的方法呢？fx 里面的代码提供了一种借鉴的思路。

譬如 fx 中，需要比较两个对象是不是相等时，需要类型实现 IEquatable&lt;T> 接口，然后调用 Equals 方法判断。但如果直接使用 IEquatable&lt;T>(t).Equals 来调用，如果 T 为值类型，必然导致装箱。fx 中很机灵地引入了一个中介类型  EqualityComparer&lt;T> 来解决这个问题，在 fx 的各种集合类中实际上都是用它来判断相等。

这个  EqualityComparer&lt;T> 有一个子类型 GenericEqualityComparer&lt;T>，平时比较主要都是承包给它。先看看它的实现：

```csharp
internal class GenericEqualityComparer<T> : EqualityComparer<T> where T : IEquatable<T>
{
        [Pure]
        public override bool Equals(T x, T y)
        {
            if (x != null)
            {
                if (y != null) return x.Equals(y);
                return false;
            }
            if (y != null) return false;
            return true;
        }
}
```

它的奥妙在于，不是将 x 和 y 转换为 IEquatable&lt;T> 来使用，而是利用了泛型的约束，保证 T 有接口的这个方法，但不是利用接口来调用，而是直接用 T.Equals 来调用，这样就避免在运行时进行接口转化，避免了值类型装箱。