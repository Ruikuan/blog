# .Net Core 3.0 性能改进

太多了，只挑一部分感兴趣的吧。原文：https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-core-3-0/

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

现在基本不直接跟 `thread` 打交道了，所有留足了优化的空间

* `UnsafeQueueUserWorkItem` 重载支持 `IThreadPoolWorkItem` 参数，让参数本身可以重用以避免分配
* `System.Threading.Channels` 的 `ReadAsync` 通过使用可重用的 `IValueTaskSource` 来支撑 `ValueTask` 返回，同时实现了 `IThreadPoolWorkItem` 并将自身当作加入队列，从而避免某种情况下的内存分配 [[PR]](https://github.com/dotnet/corefx/pull/33080)
* `async` 和 `await` 的实现利用 `IThreadPoolWorkItem` 进一步减少分配，尤其在 ` TaskCreationOptions.RunContinuationsAsynchronously` 的 `TaskCompletionSource<T>` 中
* `await Task.Yield()` 无分配
* 将 `timer` 注册队列分为即将触发和好一会才会触发两个，这样在触发 `timer` 时要处理的东西大幅度减少，而且避免了大量的锁争用

## Collections

* `ConcurrentDictionary<TKey, TValue>` 的 `IsEmpty` 认为如果随便一个 bucket 非空，就无需锁定直接返回非空，避免每次都要获取锁从而争用。
* 解决先判断 `ConcurrentQueue<T>` 在先判断 `Count` 再 `TryDequeue` 代码的性能问题
* `ImmutableDictionary<TKey, TValue>` 的查询变得更快，虽然还是没有 `Dictionary<TKey, TValue>` 那么快
* 优化了 `BitArray` 的大量操作，`Set` 和 `Get` 进一步优化，`Or`、`And` 和 `Xor` 向量化
* `SortedSet<T>` 的 `GetViewBetween` 优化
* 重整了对 enum 的 `Comparer` 的实现 [[PR]](https://github.com/dotnet/coreclr/pull/21604)

## Networking

* 在服务端给出 `Content-Length` 时，`HttpClient` 使用更大的 buffer，在快速连接和大量数据的情况下，可以减少相当多的系统调用，从而性能大幅提高 [[PR]](https://github.com/dotnet/corefx/pull/32820)
* `SslStream` 初始化优化，尤其在于分配方面
* 使用 `IThreadPoolWorkItem` 优化 `Socket` 在 Unix 系统下的表现，消除异步操作的 allocated
* `BinaryPrimitives.ReverseEndianness` 使用 `intrinsic` 实现

## System.IO

* `Stream.CopyTo` 被改为 virtual 以允许子类优化。`DeflateStream.CopyTo` 优化
* 池化 `BrotliStream` 的 buffer，优化 `ReadByte` 和 `WriteByte` 避免分配
* `StreamWriter` 具有更多专门类型的重载避免分配
* 使用 `ArrayPool<byte>.Shared` 替代 `System.IO.Pipelines` 中的 `MemoryPool<byte>.Shared`，减少了间接层
* 一大堆 PR 作用到 `System.IO.Pipelines` 上，进一步削减了 cpu 和内存的使用

## System.Console

* cache 了光标位置，然后根据用户输入改变光标位置
* buffer size 从 256 增加到 4096

## System.Diagnostics.Process

## LINQ

* 集成 `IPartition<T>` 到 `TakeLast` 促进性能提升 [[PR]](https://github.com/dotnet/corefx/pull/36051)
* `Enumerable.Range(...).Select(…)` 现在直接知道在 `Enumerable` 上操作，不需要额外 loop 处理
* `Enumerable.Empty<T>()` 的后接操作不处理

## Interop

* 处理 `SafeHandle` 生命周期的操作从非托管代码转移到托管代码，避免每次传送的开销
* 在大量 interop 中移除了 `StringBuilder` 的使用
* 优化 `FileSystemWatcher` 的 interop

## 杂七杂八

* Lower bounds explicitly provided to `Array.Copy`
* Cheaper copying to new arrays
* `Nullable<T>` 的 `GetValueOrDefault` 比 `Nullable<T>.Value` 更快因为不需要检查有没有 Value
* 将所有代码库里面的空数组替换为 `Array.Empty<T>()`
* Avoiding lots of little allocations all over the place
* 避免显式的 static 构造函数。显式的构造函数而不是在定义静态变量时对字段做初始化，C# 编译器不会把类型标记为 `beforefieldinit`，而类型标记为 `beforefieldinit ` 对性能有好处，这样 runtime 就可以更灵活地处理初始化过程，以及决定要不要在访问类型的静态方法时加锁。[[PR]](https://github.com/dotnet/coreclr/pull/21718) [[PR]](https://github.com/dotnet/coreclr/pull/21715)
* 使用更轻的足够的同等替代。使用 `Contains` 替代 `IndexOf`；`SocketsHttpHandler` 使用 `Environment.TickCount` 替代 `DateTime.UtcNow` 来确定连接是否能重用，分辨率足够了
* Simplifying interop
* Avoiding unnecessary globalization，很多情况下使用 ordinal 就足够了
* Avoiding unnecessary `ExecutionContext` flow
* Centralized / optimized bit operations，引入 `BitOperations` 处理位操作

## GC

* 更好的 free list 管理
* Large page 支持
* 优化了 write barrier
* 对具有大数量 CPU 的改进 [[blog]](https://blogs.msdn.microsoft.com/maoni/2019/04/03/making-cpu-configuration-better-for-gc-on-machines-with-64-cpus/)
* 优化对 container 的支持

## JIT

* Tiered compilation
* 将 `static readonly` 的基本类型当作 `const` 看待，对其进行优化。对于 `static readonly` base type 实例化为 derived type 的情况，调用其中虚拟方法时知道它的实际类型从而 devirtualize 甚至 inline 它
    ```cs
    private static readonly Base s_base;

    static Program() => s_base = new Derived();

    [Benchmark]
    public void AccessStatic() => s_base.Method();

    private sealed class Derived : Base { public override void Method() { } }
    private abstract class Base { public abstract void Method(); }
    ```
* 改进 stack zero
* 减少值类型包装值类型的不必要的复制
    ```cs
    [Benchmark]
    public int WrapUnwrap() => ValueTuple.Create(ValueTuple.Create(ValueTuple.Create(42))).Item1.Item1.Item1;
    ```
