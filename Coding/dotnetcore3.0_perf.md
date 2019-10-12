# .Net Core 3.0 性能改进

## Span 和它的小伙伴

* 从 `{ReadOnly}Memory<T>` 中获得 `{ReadOnly}Span` 性能改善
* 有 返回 `ref T` 的方法以及扩展方法 `GetPinnableReference` 的类型，可以使用 C# 7.3 的语法 `fixed (T* = t)` 来 pin 住内存。pin 住 span 性能提升
    ```cs
     fixed (byte* p = s) // equivalent to `fixed (byte* p = &s.GetPinnableReference())`
    ```
* 使用 `MemoryMarshal.GetReference` 能提高非空 `span` 获取引用的性能，因为少一个 null check
    ```cs
    fixed (byte* p = &MemoryMarshal.GetReference(s))
    ```
* 使用 AVX2 和 SSE2 再加速 `Span<T>` 的 `SequenceCompareTo` 和 `SequenceEqual` 方法 [[PR]](https://github.com/dotnet/coreclr/pull/22127)
* 技巧演示，将 byte 比较或查找转化为 long 操作，在 64 位系统得到将近十倍性能提升。然后使用 `Vector` 硬件优化再提高一倍。还可以使用 AVX 或 SSE 进一步同时处理更多数量的元素。
    ```cs
    public int LoopLongs()
    {
        byte[] buffer = _buffer;
        int remainingStart = 0;

        if (IntPtr.Size == sizeof(long))
        {
            Span<long> longBuffer = MemoryMarshal.Cast<byte, long>(buffer);
            remainingStart = longBuffer.Length * sizeof(long);

            for (int i = 0; i < longBuffer.Length; i++)
            {
                if (longBuffer[i] != 0)
                {
                    remainingStart = i * sizeof(long);
                    break;
                }
            }
        }

        for (int i = remainingStart; i < buffer.Length; i++)
        {
            if (buffer[i] != 0)
                return i;
        }

        return -1;
    }
    ```
    ```cs
    public int LoopVectors()
    {
        byte[] buffer = _buffer;
        int remainingStart = 0;

        if (Vector.IsHardwareAccelerated)
        {
            while (remainingStart <= buffer.Length - Vector<byte>.Count)
            {
                var vector = new Vector<byte>(buffer, remainingStart);
                if (!Vector.EqualsAll(vector, default))
                {
                    break;
                }
                remainingStart += Vector<byte>.Count;
            }
        }

        for (int i = remainingStart; i < buffer.Length; i++)
        {
            if (buffer[i] != 0)
                return i;
        }

        return -1;
    }
    ```
* `Span.CopyTo` 性能提升
* 向量化处理 `Span<byte>.IndexOfAny` 性能提升 [[PR]](https://github.com/dotnet/coreclr/pull/20738)
* 同样的向量化优化应用到 `Char` 上，也同样应用到其他具有同样 size 的基本类型上
* 向量化 `Span<Char>.To{Upper/Lower}{Invariant}`
* `ReadOnlySpan<char>.Trim{Start/End}` 性能大幅度提高
* 删除不必要的 globalization 处理代码，让 `{ReadOnly}Span<Char>` 的相关性能提升

## Arrays and Strings

* `Span.Reverse` 和 `Array.Reverse` 性能提升
* 处理对齐问题，提高了 `Array.Clear` 的性能 [[PR]](https://github.com/dotnet/coreclr/pull/24302)
* 向量化优化 `Array.{Last}IndexOf` 在 `byte` 和 `char` 上的性能 [[PR]](https://github.com/dotnet/coreclr/pull/21116)，以及把 `IndexOf` 优化推到跟 `char` 和 `byte` 同样 size 的基本类型
* 向量化 `string.IndexOf(Char, StringComparison.Ordinal)` [[PR]](https://github.com/dotnet/coreclr/pull/21076) 并去除 allcate [[PR]](https://github.com/dotnet/coreclr/pull/19788)
* 优化 `String.GetHashCode(StringComparison.Ordinal{IgnoreCase})` 的性能 [[PR]](https://github.com/dotnet/coreclr/pull/20309/)
* 向量化优化 `String.Equals(string, StringComparison.OrdinalIgnoreCase)` [[PR]](https://github.com/dotnet/coreclr/pull/20734)
* const 化 `ReadOnlySpan<byte>` [[PR]](https://github.com/dotnet/roslyn/pull/24621)
    ```cs
    static ReadOnlySpan<byte> s_byteData => new byte[] { … /* constant bytes */ }
    ```
* 去掉 `StringBuilder` Append `ReadOnlyMemory<char>` 的装箱 [[PR]](https://github.com/dotnet/coreclr/pull/20773)。不过 `StringBuilder` 只是出于方便使用的，需要注重性能的场合，应该考虑使用 `String.Create`。另外，内部使用了一个 `ValueStringBuilder`，应该很快会公开了
* 由于去掉 `StringBuilder` 的使用，下面这些得到了性能优化：`Dns.GetHostEntry`、`DateTime.ToString(Hebrew)`、`PhysicalAddress.ToString`、`X509Certificate`的各属性
* 将 `String.SubString` 替换为 `Span.Slice` 去掉 `FileSystemWatcher`，将字符串的 allocation 推迟到必要的时候，同理 `WebUtility.HtmlDecode`
* 加入 `String.Concat` 接受 `ReadOnlySpan<char>` 参数的版本，减少由此造成的 `string` 分配。`Uri.DnsSafeHost`、`Path.ChangeExtension` 受益
* 优化 `Encoding.Unicode.GetString` 和 `Encoding.Preamble`，翻新 `UTF8Encoding` 和 `AsciiEncoding`

## Parsing/Formatting

* 优化 `Int32`、`Int64`、`UInt32`、`UInt64` 的 `Parse`
* 在不需要考虑文化的条件下优化 `ToString`
* 大改 `System.Decimal`，改成全托管实现，加减乘除等各操作性能提升
* `0..9` 的 `ToString()` 直接返回 cache 好的字符串，因为这样的字符串在现实程序中大量存在
* 改进 `Enum.Parse` 和 `Enum.TryParse`，优化 `[Flags]` 枚举的 `ToString`，因此大幅度提高了枚举相关性能
* `DateTime{Offset}` 的 `Parse` 对于 `o` 和 `r` 格式大幅度提高
* `TimeSpan`、`Guid` 的 `Parse` 性能提高
* 向量化优化 `Guid` 针对 `byte` 数组和 `span` 的构造函数 [[PR]](https://github.com/dotnet/coreclr/pull/21336)

## 正则表达式

* 将 `System.Text.RegularExpressions` 中的 `StringBuilder` cache 实现替换为基于 `ref struct` 的 builder 实现，新 builder 利用 stack allocated 空间以及池化的 buffer [[PR]](https://github.com/dotnet/corefx/pull/30474)。然后进一步使用 `Span` 推进。最大的改进来自于避免 `RegexOptions.Compiled Regex` 无端端的读取线程当前文化相关信息 [[PR]](https://github.com/dotnet/corefx/pull/32899)，这个跟 `RegexOptions.IgnoreCase` 一起使用更佳。

## Threading

