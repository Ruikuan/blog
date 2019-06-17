# 字符串性能小技巧

原文 [Some performance tricks with .NET strings](https://www.meziantou.net/some-performance-tricks-with-dotnet-strings.htm)

作者给 AspNetCore 提交了 [Pull Request](https://github.com/aspnet/AspNetCore/pull/6784)，针对相关字符串 Id 生成做了优化，优化后的代码比原来性能快一倍多。

## 原始代码

代码将一个 `long` 值编码为 base 32 的字符串。而且这段代码已经经过优化，优化方式可见注释。

```cs
private static readonly string s_encode32Chars = "0123456789ABCDEFGHIJKLMNOPQRSTUV";

public static unsafe string GenerateId(long id)
{
    // The following routine is ~310% faster than calling long.ToString() on x64
    // and ~600% faster than calling long.ToString() on x86 in tight loops of 1 million+ iterations
    // See: https://github.com/aspnet/Hosting/pull/385

    // stackalloc to allocate array on stack rather than heap
    char* buffer = stackalloc char[13];
    buffer[0] = s_encode32Chars[(int)(id >> 60) & 31];
    buffer[1] = s_encode32Chars[(int)(id >> 55) & 31];
    buffer[2] = s_encode32Chars[(int)(id >> 50) & 31];
    buffer[3] = s_encode32Chars[(int)(id >> 45) & 31];
    buffer[4] = s_encode32Chars[(int)(id >> 40) & 31];
    buffer[5] = s_encode32Chars[(int)(id >> 35) & 31];
    buffer[6] = s_encode32Chars[(int)(id >> 30) & 31];
    buffer[7] = s_encode32Chars[(int)(id >> 25) & 31];
    buffer[8] = s_encode32Chars[(int)(id >> 20) & 31];
    buffer[9] = s_encode32Chars[(int)(id >> 15) & 31];
    buffer[10] = s_encode32Chars[(int)(id >> 10) & 31];
    buffer[11] = s_encode32Chars[(int)(id >> 5) & 31];
    buffer[12] = s_encode32Chars[(int)id & 31];
    return new string(buffer);
}
```

## 使用 `Span<T>`

第一个 commit 是用 `Span<T>` 来取代原来的 unsafe 代码。基本对性能没影响，但不再是 unsafe 了。

```cs
private static readonly string s_encode32Chars = "0123456789ABCDEFGHIJKLMNOPQRSTUV";

// Remove unsafe keyword
public static string GenerateId(long id)
{
    // Replace char* with Span<T>
    Span<char> buffer = stackalloc char[13];
    buffer[0] = s_encode32Chars[(int)(id >> 60) & 31];
    buffer[1] = s_encode32Chars[(int)(id >> 55) & 31];
    ...
    buffer[11] = s_encode32Chars[(int)(id >> 5) & 31];
    buffer[12] = s_encode32Chars[(int)id & 31];
    return new string(buffer, 0, 13);
}
```

## 使用 `string.Create`

第一个优化是使用 `string.Create`。在上面的代码，我们在 stack 上创建了一个 buffer，然后再使用这个 buffer 创建了一个在 heap 上的 `string`。这意味着 buffer 被复制到了新的字符串实例。我们可以使用 `string.Create` 来避免这个复制过程。这个方法直接在 heap 上分配字符串，而且你能过够使用 delegate 来直接设置字符串的内容。[参见 Github 上的源码](https://github.com/dotnet/corefx/blob/88ec3d985b4879bd77b7821f154663099e7bae29/src/Common/src/CoreLib/System/String.cs#L343)。

```cs
private static readonly string s_encode32Chars = "0123456789ABCDEFGHIJKLMNOPQRSTUV";

public static string GenerateId(long id)
{
    return string.Create(13, id, (buffer, value) =>
    {
        // Do not use "id" here as it would create a closure and allocate
        // instead use "value" (read hereafter for more details)
        buffer[0] = s_encode32Chars[(int)(value >> 60) & 31];
        buffer[1] = s_encode32Chars[(int)(value >> 55) & 31];
        ...
        buffer[11] = s_encode32Chars[(int)(value >> 5) & 31];
        buffer[12] = s_encode32Chars[(int)value & 31];
    }
}
```

你可能注意到我没有在 lambda 表达式中使用 `id`。因为这样会导致闭包。闭包在这里起到坏作用，会让 delegate 无法缓存，每次这个方法被调用都会实例化一个新的 delegate。最终会拖慢性能还给 GC 增加了压力。这就是为什么 `string.Create` 的第二个参数 `TState state` 会存在的原因。这个参数让我们可以避免使用闭包。你可以在 [the Jetbrains'blog](https://blog.jetbrains.com/dotnet/2014/07/24/unusual-ways-of-boosting-up-app-performance-lambdas-and-linqs/) 了解更多关于闭包的信息。

使用 `string.Create` 在基准测试中快了大约 35% !

[Steve Gordon](https://twitter.com/stevejgordon) 也写过一篇关于 `string.Create` 的文章：[Creating Strings with No Allocation Overhead Using String.Create](https://www.stevejgordon.co.uk/creating-strings-with-no-allocation-overhead-using-string-create-csharp)

## 反转赋值顺序

[David Fowler](https://twitter.com/davidfowl) 建议我把赋值的顺序倒置过来（譬如先赋值 `buffer[12]` 然后 `buffer[11]` 依此类推）。这样使 JIT 不再添加 [bounds checks](https://en.wikipedia.org/wiki/Bounds_checking)。毕竟，如果你可以访问 `buffer[12]`，更小的索引值就更不可能越界了。

```cs
private static readonly string s_encode32Chars = "0123456789ABCDEFGHIJKLMNOPQRSTUV";

public static string GenerateId(long id)
{
    return string.Create(13, id, (buffer, value) =>
    {
        // Assign buffer from 12 to 0 to avoid a bound check
        buffer[12] = s_encode32Chars[(int)value & 31];
        buffer[11] = s_encode32Chars[(int)(value >> 5) & 31];
        ...
        buffer[1] = s_encode32Chars[(int)(value >> 55) & 31];
        buffer[0] = s_encode32Chars[(int)(value >> 60) & 31];
    }
}
```

JIT 生成优化后的代码后，这优化后的版本在我的机器上的确跟上一个性能一模一样。 Günther Foidl 提了个看法，这可能是由于 [CPU 的分支预测](https://en.wikipedia.org/wiki/Branch_predictor)在这种情况下工作得非常好。这不意味着这个优化全无作用，其他的 CPU 可能不会对这种情况有那么好的分支预测。而且，新生成的代码更加少，总是件好事。

> Tip：使用 `BenchmarkDotNet` 你可以很容易通过使用 `[DisassemblyDiagnoser(printAsm: true, printSource: true)]` 声明得到汇编代码。

如下是优化后的汇编代码的对比：

![对比](https://www.meziantou.net/assets/boundscheck.png?v=45a668d7&utm_medium=social&utm_source=web)

全部汇编代码在[这里](https://gist.github.com/meziantou/7ab15c70f7e9fc0b55dec4b074fb3209#file-bounds-check-assembly-txt)。

## 复制字段到本地变量

JIT 必须针对每次访问 `s_encode32Chars` 生成加载代码，因为它是一个可变的数组（它的引用是 `readonly` 的，但其中的单个元素不是不可能改变的）因此从 JIT 的角度来看其中的元素可能会改变，所以每次必须加载**最新**的数据。

这可以通过在本地加载它一次来避免，这样 JIT 可以跟踪到没有任何改变发生，可以实现只读的访问。

```cs
private static readonly string s_encode32Chars = "0123456789ABCDEFGHIJKLMNOPQRSTUV";

public static string GenerateId(long id)
{
    return string.Create(13, id, (buffer, value) =>
    {
        // Copy the reference
        var encode32Chars = s_encode32Chars;

        buffer[12] = encode32Chars[(int)value & 31];
        buffer[11] = encode32Chars[(int)(value >> 5) & 31];
        ...
        buffer[1] = encode32Chars[(int)(value >> 55) & 31];
        buffer[0] = encode32Chars[(int)(value >> 60) & 31];
    }
}
```

基准测试表明这个版本比上个版本快 4% ！

![对比](https://www.meziantou.net/assets/loadstring.png?v=a077dd90&utm_medium=social&utm_source=web)

## 移除显式的类型转换

最后一个优化是移除显式的到 `int` 的类型转换。`string` 的索引器只接受 `int`，所以需要做这样的转换。但我们可以很简单地将 `string` 变为 `char[]`，这样我们就可以使用 `long` 来索引。去掉这个转换后，JIT 可以为 64 位系统生成更优的代码。对于 32 位系统，会生成附加的代码，从而导致较差的性能。但鉴于绝大多数的应用现在是在 64 位系统上面跑（尤其是 Web Server），所以这样的改动是完全可接受的。

```cs
// Change from string to char[], so we can use the long
private static readonly char[] s_encode32Chars = "0123456789ABCDEFGHIJKLMNOPQRSTUV".ToCharArray();

public static string GenerateId(long id)
{
    return string.Create(13, id, (buffer, value) =>
    {
        var encode32Chars = s_encode32Chars;

        // Remove explicit cast in the indexer
        buffer[12] = encode32Chars[value & 31];
        buffer[11] = encode32Chars[(value >> 5) & 31];
        ...
        buffer[1] = encode32Chars[(value >> 55) & 31];
        buffer[0] = encode32Chars[(value >> 60) & 31];
    }
}
```

基准测试表明现在代码比上一个版本快大约 5% ！

## 基准测试

我使用 [BenchmarkDotNet](https://www.meziantou.net/comparing-implementations-with-benchmarkdotnet.htm?utm_medium=social&utm_source=web) 创建了一个基准测试来测量这个 pull requst 的每次改动会造成怎样的性能变化。你可以在这里找到代码：[The code on Gist](https://gist.github.com/meziantou/7ab15c70f7e9fc0b55dec4b074fb3209#file-code-cs)

[测量结果](https://gist.github.com/meziantou/7ab15c70f7e9fc0b55dec4b074fb3209#file-results-md)

最终代码比最开始的代码快一倍，而且不再有 unsafe 代码。感谢 pull request 的所有 reviewer 帮助我发现新的 C# features！