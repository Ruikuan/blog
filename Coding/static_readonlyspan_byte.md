# ReadOnlySpan 访问静态数据优化

引入 `ReadOnlySpan` 之后，与之相关的优化在逐渐进行。

譬如下面的的 lookup table

```cs
private readonly static byte[] LookupTable = new byte[] {
    (byte)'0', (byte)'1', (byte)'2', (byte)'3', (byte)'4',
    (byte)'5', (byte)'6', (byte)'7', (byte)'8', (byte)'9',
    (byte)'A', (byte)'B', (byte)'C', (byte)'D', (byte)'E',
    (byte)'F'
};
```

`LookupTable` 初始化时会执行一个 `InitializeArray` 操作，这个操作内部会调用 `memcpy` 将对应的 `0123456789ABCDEF` 字节内容从运行映像的 `.text` 段复制到托管堆中，从而构建出一个字节数组对象。

而我们将代码稍微改动一下，改成对应的 `ReadOnlySpan` 版本:

```cs
private static ReadOnlySpan<byte> LookupTable => new byte[] {
    (byte)'0', (byte)'1', (byte)'2', (byte)'3', (byte)'4',
    (byte)'5', (byte)'6', (byte)'7', (byte)'8', (byte)'9',
    (byte)'A', (byte)'B', (byte)'C', (byte)'D', (byte)'E',
    (byte)'F',
};
```

因为 PR [Refer directly to static data when ReadOnlySpan wraps arrays of bytes](https://github.com/dotnet/roslyn/pull/24621) 的优化，编译器知道 `ReadOnlySpan<byte>` 在这里相当于一个 `const` 的语义，它的内容在编译器就已经能确定，而且由于是 `ReadOnlySpan`，运行时也不会被改变。因此将字节数组内容编译到 `.text` 段之后，在访问 `LookupTable` 时，也不需要再用 `memcpy` 构建数组，而是直接将访问的引用指向运行映像中只读的 `.text` 中的字节内容。减少了初始化的执行指令，也减少了不必要的内存分配。

除了在静态字段中可以这样使用，还可以用在方法中返回：

```cs
ReadOnlySpan<byte> GetBytes() {
    return new byte[] { ... };
}
```

还可以用在局部变量初始化中：

```cs
void Write200Ok(Stream s) {
    ReadOnlySpan<byte> data = new byte[] {
        (byte)'H', (byte)'T', (byte)'T', (byte)'P',
        (byte)'/', (byte)'1', (byte)'.', (byte)'1',
        (byte)' ', (byte)'2', (byte)'0', (byte)'0',
        (byte)' ', (byte)'O', (byte)'K'
    };
    s.Write(data);
}
```

只要字节内容是编译时能确定的，而且分配的目标对象是 `ReadOnlySpan<byte>`，都能用上这个优化。

不过对于字符串来说，这样构造字节数组可读性实在太差，写代码痛苦，也容易出现错误。后续还在继续研究怎样才能优化字符串的使用体验。[Extend ReadOnlySpan<byte> optimization for static data to work with ASCII/UTF8 strings](https://github.com/dotnet/csharplang/issues/2212)。

> 参考:  
> [C# ReadOnlySpan and static data](https://vcsjones.dev/2019/02/01/csharp-readonly-span-bytes-static/)  
> [Performance Improvements in .NET Core 3.0](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-core-3-0/)  