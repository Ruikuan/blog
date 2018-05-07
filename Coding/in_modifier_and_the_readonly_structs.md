# in 修饰符和 readonly struct 以及伴随的性能影响

## 值类型的 readonly 字段和性能影响

`readonly` 修饰符在值类型和引用类型之间的表现有点不同。对于引用类型的 `readonly` 字段，编译器只保证它在构造方法之外不能重新指定，即不能再通过 `a = xxx` 来重新设定引用，不会管它内部进行了什么改变。而对于值类型的 `readonly` 字段，则意味着在实例的整个生命周期中，所有它的内部值都不会变化。为了避免潜在的变化，对于 `readonly` 的值类型字段，编译器每次调用方法或属性之前都会进行防御性复制。防御性复制带来可观的性能开销。

```cs
private FairlyLargeStruct _nonReadOnlyStruct = new FairlyLargeStruct(42);
private readonly FairlyLargeStruct _readOnlyStruct = new FairlyLargeStruct(42);
private readonly int[] _data = Enumerable.Range(1, 100_000).ToArray();
        
[Benchmark]
public int AggregateForNonReadOnlyField()
{
    int result = 0;
    foreach (int n in _data)
        result += n + _nonReadOnlyStruct.N;
    return result;
}
 
[Benchmark]
public int AggregateForReadOnlyField()
{
    int result = 0;
    foreach (int n in _data)
        result += n + _readOnlyStruct.N;
    return result;
}
```
```
                       Method |      Mean |    Error |    StdDev |
----------------------------- |----------:|---------:|----------:|
 AggregateForNonReadOnlyField |  87.92 us | 1.800 us |  3.677 us |
    AggregateForReadOnlyField | 148.29 us | 4.226 us | 12.460 us |
```

仅仅多了一个 `readonly` 修饰符就造成了大量的性能损失。  


解决这个问题至少有三个办法：

1. 使用字段而不是使用属性。编译器看到只是读取 struct 字段的操作时，知道不会有副作用，因此不会进行防御性复制。但这个对封装不好。  
2. 不要使用 `readonly` 修饰值类型。  
3. 使用 `readonly struct`

```cs
public readonly struct FairlyLargeStruct
{
    private readonly long l1, l2, l3, l4;
    public int N { get; }
    public FairlyLargeStruct(int n) : this() => N = n;
}
```

## `readonly struct`

C# 7.2 允许通过 `readonly struct` 表明值类型的 immutable，不但对性能有好处，而且能够更明确表示一种不可变的观点：值是 immutable 的。（不过可以通过某些肮脏的反射操作来破坏。）

`readonly struct` 强制如下行为：
1. 编译器检查 struct 是真的不可变并且只由 readonly 的字段和/或只读的属性。（像 `public int Foo {get; private set;}` 这种就不是只读的）  
2. 允许编译器在某些上下文省略防御性复制，像上面提到的 `readonly` 值类型字段之类的情况。

对于 `readonly struct FairlyLargeStruct` 的基准结果如下：

```
                       Method |     Mean |    Error |   StdDev |
----------------------------- |---------:|---------:|---------:|
 AggregateForNonReadOnlyField | 91.19 us | 1.811 us | 2.597 us |
    AggregateForReadOnlyField | 89.25 us | 1.775 us | 3.705 us |
```

## in 修饰符

之前 C# 有三种传参方式：值传递，传引用（ref），输出参数（out），实际上在内部 ref 和 out 是一样的。

C# 7.2 带来了新的传参方式：`in` 修饰符。`in` 的语义是只读的引用，在底下，参数被当作用`System.Runtime.CompilerServices.IsReadOnlyAttribute`修饰的引用传递。编译器确保在方法中不会修改这个参数。

```cs
public void Foo(in string s)
{
    // Cannot assign to variable 'in string' because it is a readonly variable
    s = string.Empty;
}
```

1. 不能用重载区分 `in`、`ref` 和 `out`，它们本质上是一样的

2. 不能将这三个用于迭代器和 async 方法

3. **可以**将 using 块的变量通过 `in` 传递，即使不能通过 `ref` 和 `out` 传递。因为通过 `in` 传递是安全的，编译器去除了这个限制。

```cs
struct Disposable : IDisposable
{
    public void Dispose() { }
}
 
public void DisposableSample()
{
using (var d = new Disposable())
{
    // Ok
    ByIn(d);
    // Cannot use 'd' as a ref or out value because it is a 'using variable'
    //ByRef(ref d);
}
 
void ByRef(ref Disposable disposable) { }
void ByIn(in Disposable disposable) { }
```

4. `in` 参数可以有默认值，`ref` 和 `out` 不行。

```cs
public int ByIn(in string s = "") => s.Length;
```

5. **可以**进行只有 `in` 修饰符的不同的重载。

```cs
public int Foo(in string s) => s.Length;
public int Foo(string s) => s.Length;
```

在有这样重载实现的时候，在 C# 7.2 中 `Foo(s)` 调用的是 `Foo(in string s)`，而在 C# 7.3 之后，调用的是 `Foo(string s)`，看起来是 C# 7.2 的实现存在语义上的问题。

但对于不是重载的情况，只有 `in` 实现的情况下，对于调用方，`in` 参数的 `in` 修饰符是可选的，这样对于 API 的提供方很方便，可以将自己的实现改成通过 `in` 参数传入大结构，而无需自己的所有调用方都修改自己的调用代码，就可以获得性能的提高。不过不能通过 `in` 字面量的方式进行调用。

```cs
public int ByIn(in string s) => s.Length;

string s = string.Empty;
ByIn(in s); // Works fine
ByIn(s); // Works fine as well!
// Fail?!?! An expression cannot be used in this context because it may not be passed or returned by reference
ByIn(in "some string");
ByIn("some string"); // Works fine!
```

## `in` 修饰符的性能特性

`in` 参数跟 `readonly` 字段相当类似，为了避免破坏 struct 的 `readonly/in` 语义，在调用 struct 的属性和方法之前，编译器会做一次防御性复制，从而导致性能降低。因此，**绝不应该通过 `in` 来传递非 `readonly struct` 结构体**！非 `readonly struct` 通过 `in` 传递常常导致频繁的防御性复制，让性能变得更糟。

```cs
public struct FairlyLargeStruct
{
    private readonly long l1, l2, l3, l4;
    public int N { get; }
    public FairlyLargeStruct(int n) : this() => N = n;
}
 

private readonly int[] _data = Enumerable.Range(1, 100_000).ToArray();
 
[Benchmark]
public int AggregatePassedByValue()
{
    return DoAggregate(new FairlyLargeStruct(42));
 
    int DoAggregate(FairlyLargeStruct largeStruct)
    {
        int result = 0;
        foreach (int n in _data)
            result += n + largeStruct.N;
        return result;
    }
}
 
[Benchmark]
public int AggregatePassedByIn()
{
    return DoAggregate(new FairlyLargeStruct(42));
 
    int DoAggregate(in FairlyLargeStruct largeStruct)
    {
        int result = 0;
        foreach (int n in _data)
            result += n + largeStruct.N;
        return result;
    }
}
```
结果

```
                 Method |      Mean |     Error |    StdDev |
----------------------- |----------:|----------:|----------:|
 AggregatePassedByValue |  71.24 us | 0.3150 us | 0.2278 us |
    AggregatePassedByIn | 124.02 us | 3.2885 us | 9.6963 us |
```

## 结论

* `readonly struct` 对于设计和性能角度都很有用。
* 如果 struct 的大小比 `IntPtr.Size` 大，应该通过 `in` 来传递获得性能提高。
* 可以通过使用 `in` 来传递引用类型，让自己的设计意图更清晰。（其实也无所谓）
* 绝不使用 `in` 来传递非 `readonly struct`，因为对性能会造成负面的印象，而且常常是不容易发觉的。

> 来源：[The ‘in’-modifier and the readonly structs in C#](https://blogs.msdn.microsoft.com/seteplia/2018/03/07/the-in-modifier-and-the-readonly-structs-in-c/)  
> 参考：[Reference semantics with value types](https://docs.microsoft.com/en-us/dotnet/csharp/reference-semantics-with-value-types)
