# 关于 ValueTask

C# 7 中引入了 `ValueTask`。（这句话不够严谨）目的是为了解决一些情形下 `Task` 会造成过多内存分配的问题。

众所周知，`Task` 是个 `class`，每次返回一个 `Task` 都会分配一个新的对象，在内存中积累起来，造成 gc 压力。如果它对应的是 IO 任务那还好，IO 导致的延迟和开销可能远超这个小小的内存分配，负面影响不显。但如果只是由于 API 设计的限制，需要使用类似 `Task.FromResult` 直接返回一个值，那就很浪费了。而且在某些类似缓存的场合，第一次操作是大开销的 IO 操作，之后的所有访问都是直接从内存中返回，这样也很浪费。特别是在循环里面 `await` 这个 `Task`。

为了某种程度上弥补这种负面影响，引入了 `ValueTask`。`ValueTask` 是一个 `struct`，所以通常它自己是不会在堆里面分配东西的。它很符合上述的缓存模型的使用方式，有两个构造函数，一个是 `ValueTask(TResult result)`，直接通过传入结果构造；另一个是 `ValueTask(Task<TResult> task)`，指向了另一个 `Task`，如果调用了这个构造函数，无疑还是会在堆里分配这个 `Task` 的。

[微软文档](https://docs.microsoft.com/en-us/dotnet/articles/csharp/whats-new/csharp-7#generalized-async-return-types)里面推荐的使用方式如下。感觉也是最适合它的使用方式：

```csharp
public ValueTask<int> CachedFunc()
{
    return (cache) ? new ValueTask<int>(cacheResult) : new ValueTask<int>(LoadCache());
}
private bool cache = false;
private int cacheResult;
private async Task<int> LoadCache()
{
    // simulate async work:
    await Task.Delay(100);
    cacheResult = 100;
    cache = true;
    return cacheResult;
}
```

### 主要优点

对于能够直接同步返回值的操作，直接返回 `new ValueTask(result)` 是非常好的，它将 `result` 直接返回，并不会导致在堆里分配内存。频繁调用也不会有什么副作用。

保持兼容性，如果是执行异步操作，那么就直接返回底下的 Task，操作方式跟原来的一样。当然，分配就无可避免了。

### 主要缺点

如果操作都是异步操作，那么就相当于给 `Task` 多加了一些开销，也避免不了分配。

`ValueTask` 不如 `Task` 那么直接。

由于 `ValueTask` 包含了 `ResultValue` 字段以及 `Task` 字段，所以 `await` 生成的状态机会更加复杂，导致性能可能下降。


所以 `ValueTask` 也不是万能药，需要综合多种因素（同步异步操作的比例，操作耗时等）来进行取舍，通常需要进行测试来验证是否值得将 `Task` 替换为 `ValueTask`。

### 2019-8-24 更新

由于 `ValueTask` 的异步操作委托给它自己对应的单独的一个 `Task` 只是一个实现细节，很可能在后续的实现中更改。目前 BCL 中正在尝试使用 `IValueTaskSource<TResult>` 来重新实现 `ValueTask` 的行为，池化它涉及异步操作的部分，以减少 alloc，可预计很快就会实现。[目前的修改和性能基准测试取得了很好的效果](https://github.com/dotnet/coreclr/pull/26310)。所以后面在 `ValueTask` 和 `Task` 的选择中，除非是为了使用并发的 `Task Parallel Library` 的功能，只是涉及异步的话，一律使用 `ValueTask`，这样可以乘上 BCL 和 CLR 的改进东风，什么都不用做就能获得大量的性能提升和更少的内存分配。不过需要注意的是，对所有的 `ValueTask`，只能 `await` 一次。因为池化后，`await` 过后，后面的资源就认为可以被其他任务复用了。再次 `await` 将会造成混乱，从而出异常。