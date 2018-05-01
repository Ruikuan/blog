# .Net Core 2.1 性能改进

根据 [Performance Improvements in .NET Core 2.1](https://blogs.msdn.microsoft.com/dotnet/2018/04/18/performance-improvements-in-net-core-2-1/) 一文总结。比较的对象为 .Net Core 2.0。

## JIT

* `EqualityComparer<T>.Default.Equals` 能被 intrinsic 识别，devirtualize 并 inline，带来 ~2.5x 的性能提升。
* 受上面的影响，`Dictionary<Tkey,TValue>` 的 `ContainsValue` 也提升了 ~2.25x；上面基本上给所有的集合类搜索都带来了可观的提升。
* 需在代码中直接使用 `EqualityComparer<T>.Default.Equals`，而不是先将 `EqualityComparer<T>.Default` 放到一个变量，然后用那个变量的 `Equal` 方法，这种微优化由于 JIT 识别不出来，无法生成高效的代码从而变成了**负优化**。
* ` Enum.HasFlag` 也成了 intrinsic，从原来的臭名昭著的反射实现变成了 JIT 直接位操作，性能提升 ~50x，并且不会再有 allocated。
* JIT 将循环中的局部 return/break 移到热路径外，让循环体更加紧凑连续，不再需要使用 goto 等 trick 来优化，从而改善了代码的 layout，还改善了代码编写的 shape，并且性能将近 ~2x。
* 某些情况下将 [TEST 和 LSH 指令改为生成 BT 指令](https://github.com/dotnet/coreclr/pull/13626)，~1.4x。
* 识别类似 `((IAnimal)thing).MakeSound()` 这样通过接口来调用 struct 方法的代码（其中 thing 为 struct），避免装箱和接口方法调用，直接调用成员方法，并且尽可能 inline。~10x 的性能提升，并且不会 allocated。
* 所有的优化组合起来使用，对上层代码有显著的性能提升。

## Threading

* 优化了 threadstatic 的访问性能。
* 改善了 Monitor 在存在争用情况下的开销。
* 改善了 ReaderWriterLockSlim 的 scalability。
* 将 Timer 的所有操作（create, modify, fire, remove）从竞争一个全局锁的串行改成多个锁，提高并行度。
* 将 `CancellationTokenSource. CancellationToken` 的关注重点从 scalability 更改为 throughput 和 allocation，实现了 ~1.7x 的性能提升和 0 分配。
* await 同步返回的 async 方法的开销削减，性能提升 ~1.5x。
* await 异步的 async 方法的分配从 4 个对象缩减为 1 个，从而将分配空间减少一半。

## String

* 利用 `Span<T>.SequenceEqual` 来向量化 `String.Equal`，性能提升 ~1.6x。
* 同样向量化 `String.IndexOf ` 和 `String.LastIndexOf`, ~2.7x。
* `String.IndexOfAny` ~2.5x。
* `String.ToLower` 和 `ToUpper`（包括 `ToLower/UpperInvariant`）在需要转换时 ~3x；在不需要转换（即字符已经符合目标）时 ~2x，而且不再有 allocated。
* 对于 `String.Split`，视字符串的长度或使用栈上分配或使用 `ArrayPool<int>.Shared` 来消除原先用来记录字符串分段的 Int32[] 分配。再辅以 span 的优势来改善常见情形的 throughput，性能提升 ~1.5x，分配只为原来的 1/4。
* 即使边边角角的字符串方法都有改善，如 `String.Concat(IEnumerable<char>)` ~1.5x，分配为原来的 1/9。

## Formatting and Parsing

* `string.Format` 性能 ~1.3x，分配为 2/3。
* `StringBuilder.Append` 性能 ~2x，分配为 0。
* 将 coreclr 和 corert 中跟 string 和 span 相关的大部分代码用 C# 重写，以可以应用更好的优化，比原先 c++ 的实现相比有很大的性能改善。譬如 `Int32.ToString()` ~2x；`int.Parse` ~1.3x。
* 同理，`Int32`, `UInt32`, `Int64`, `UInt64`, `Single`, `Double` 的 `ToString()` 也一样改进。尤其在 unix 有将近 10 倍的提升。
* `BigInteger.ToString` ~12x！并且 allocated 只为原先的 1/17。
* `DateTime` 和 `DateTimeOffset` 的 `R` 和 `O` 的 `ToString`，分别提升 ~4x 和 ~2.6x。
* `Convert.FromBase64String` ~1.5x；`Convert.FromBase64CharArray` ~1.6x。

## Networking

* `Dns.GetHostAddressAsync` 在 windows 上变成了真·异步操作。
* `IPAddress.HostToNetworkOrder/NetworkToHostOrder` ~8x。
* `Uri` 的分配降低到原来的一半，初始化 ~1.5x。
* `Socket` 收发 ~2x。
* 消除大量 `SslStream` 中的分配，修正它在 unix 中的瓶颈。
* `HttpClient` 现在使用完全 C# 编写的 `SocketsHttpHandler`，一个用 `Socket` `SslStream` 和 `NetworkStream` 做服务端，用 `HttpClient` 做客户端通过 https 访问的示例，获得了 12.7 倍的性能飙升！同时分配也大大减少，也没了 Gen1 对象。

## And More

* `Directory.EnumerateFiles` ~3x，allocated 1/2。
* `System.Security.Cryptography.Rfc2898DeriveBytes.GetBytes` 由于 `Span` 的使用完全消除了计算过程中的分配。总的分配：1120120 B -> 176 B，性能有所提高。
* `Guid.NewGuid()` 在 unix 上有 4 倍的提升。
* [数组处理](https://github.com/dotnet/coreclr/pull/13962)、[LINQ](https://github.com/dotnet/corefx/pull/23368)、[Environment](https://github.com/dotnet/coreclr/pull/14502)、[collection](https://github.com/dotnet/corefx/pull/26087)、[globalization](https://github.com/dotnet/coreclr/pull/17399)、[pooling](https://github.com/dotnet/coreclr/pull/17078)、[SqlClient](https://github.com/dotnet/corefx/pull/27758)、[StreamWriter 和 StreamReader](https://github.com/dotnet/corefx/pull/22147) 等都有很大的改进。
* `Regex.Compiled` 回来了而且生效，Match 性能提高约一倍。

