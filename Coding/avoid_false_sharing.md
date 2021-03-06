# 避免并行程序中的 false sharing

我们知道，编写并发程序时，如果要访问共享的状态，需格外留意共享状态上的读写竞争，通常需要对共享状态进行排他锁定来进行修改，以保证系统行为正确，并发安全。对共享状态的访问锁定会导致整个系统的并发度下降，扩展性差。因此设计并发程序的一个要点就是尽量避免存在竞争的共享状态。

但在某些并行程序中，即使不存在显式的状态竞争，也可能由于 false sharing，造成并行度降低，吞吐量下降，系统性能糟糕。

考虑下面的程序：

```cs
class Counters
{
    long[] m_longs;
    
    internal Counters(int n)
    {
        m_longs = new long[n];
    }

    public void Increment(int i)
    {
        m_longs[i] ++;
    }
}

class Program
{
    static void Main(string[] args)
    {
        int p = int.Parse(args[0]);
        const int iterations = int.MaxValue / 4;
        ManualResetEvent mre = new ManualResetEvent(false);

        Counters c = new Counters(p);

        Thread[] ts = new Thread[p];

        for (int i = 0; i < ts.Length; i++)
        {
            int tid = i;
            ts[i] = new Thread(delegate ()
            {
                SetThreadAffinityMask(GetCurrentThread(), new UIntPtr(1u << tid));
                mre.WaitOne();
                for (int j = 0; j < iterations; j++)
                {
                    c.Increment(tid);
                }

            });
            ts[i].Start();
        }

        Stopwatch sw = Stopwatch.StartNew();
        mre.Set();
        foreach (Thread t in ts) t.Join();
        Console.WriteLine(sw.ElapsedTicks);
    }

    [DllImport("kernel32.dll")]
    static extern IntPtr GetCurrentThread();

    [DllImport("kernel32.dll")]
    static extern UIntPtr SetThreadAffinityMask(IntPtr hThread, UIntPtr dwThreadAffinityMask);
}
```

程序很简单，就是多个 counter 组成一个数组，然后根据我们需要的处理能力，使用不同数量的线程去并行处理这些 counter，在本例中就是将它累加。为了保证这些线程并行执行，使用 `SetThreadAffinityMask` 将处理线程跟系统的 CPU（核心） 进行绑定，让每个线程单独占用一个 CPU（核心）。

表面看起来，各个 CPU（核心）各自独立 happy 地处理自己对应的那一个数组元素，并没有跟其他 CPU 有共享数据。所以期望的处理结果是，只要处理的线程数量在 CPU 个数范围内，处理能力都能按 CPU 个数线性扩展。即使同时处理 n 个任务，总处理时间应该与处理一个任务的时间，大致保持不变。

实际上运行的结果令人大跌眼镜：

|n|运行时间|比例|
|--|---:|---:|
|1|22425789|100%|
|2|42023726|187%|
|4|175828522|784%|
|8|333906288|1489%|

运行时间跟设想中的完全不是一回事。随着任务增加，处理时间迅速攀升，比直接单线程处理同等规模的运行还糟！而且线程越多，性能越糟。

为啥？原因在于这段代码中隐藏着一个 sharing。

内存数据被 CPU 处理前会先读到 CPU Cache 中（L1、L2、L3），每次读取时是都按照一个 cache line 大小来读的。通常在 intel CPU 上 cache line 大小为 64 bytes，也存在一些 CPU 的 cache line 大小有所不同，譬如 128 bytes 等，不过这种很少。

现在看回我们上面的代码，一个 long 是 8 个 byte，那么 8 个元素的 `long[8]` 数组会在同一个 cache line 中被一次读入到 CPU 的 cache 中。多线程处理时，同一个 `long[8]` 会被分别读到每个 CPU 自己的 cache 里面（为了简化概念，不考虑存在共享 cache 的情况）。那么某个 CPU 改变了其中的某一个元素的值时，整个 cache line 的数据都被污染了。由于多个 CPU 在 cache line 层面共享这个数组，因此需要将这个 cache line 的数据都写回内存；然后其他 CPU 要对这个数组的其他元素进行操作，又要重新将这个数组的全部内容加载进自己的 cache 中。如此不断在 累加一个元素 -> 写回内存 -> 其他 cpu cache 失效 -> 读取内存 循环。相当于所有 CPU 都在竞争这一小块内存的使用，由于大量的数据要在内存和 CPU 缓存间不断传输，比单线程串行处理还糟糕。这就是 false sharing 的危害。

![False sharing occurs when threads on different processors modify variables that reside on the same cache line. This invalidates the cache line and forces a memory update to maintain cache coherency.](https://software.intel.com/sites/default/files/m/d/4/1/d/8/5-4-figure-1.gif)

那么，应该如何应对这种情况呢？

处理方法不难。由于 CPU 读取是按照 cache line 读取的，只要不相关的元素不处于同一个 cache line 上就行了。落实到这个数组上，就是将元素的间隔拉开到 64 字节以上，中间填充 0。

对于 C# 的数组，还有一个需要额外注意的点：数组的内存布局中，第一个元素紧跟在 `length` 存储位置的后面，因此所有对数组元素的访问都会因为用到 `length` 来做 bound check，从而跟第一个元素有了 false sharing。因此第一个元素跟 `length` 位置也要拉开间隔。

![数组内存布局](../Content/array_layout.jpg)

修改 `Counters` 代码如下：

```cs
class Counters
{
    long[] m_longs;
    internal Counters(int n)
    {
        m_longs = new long[(n+1)*16]; // long 为 8 字节，所以这里每个有效元素的距离拉开到 8 * 16 = 128 个字节
    }

    public void Increment(int i)
    {
        m_longs[(i+1)*16]++;    //对有效元素进行处理
    }
}
```

这样拉开处理之后，各个要处理的 `counter` 就不再在同一个 cache line 上了。并行处理也不会再导致 false sharing。性能就能够按 CPU 数目线性提高了。测量结果如下：

|n|运行时间|比例|
|--|---:|---:|
|1|21914250|100%|
|2|21900392|100%|
|4|21865781|100%|
|8|21934008|100%|

随着任务数量的增加，总处理时间不变，处理能力成比例增加。由此我们可以看出 false sharing 的性能破坏力，不正确处理会严重影响整个系统的处理性能。

除了数组之外，我们也可以利用 `struct` 的 layout 设定能力来避免 false sharing。使用 `Explicit` 的 layout，我们可以将 `struct` 字段的内存距离拉到 64 字节以上。

```cs
[StructLayout(LayoutKind.Explicit)]
struct Counters
{
    [FieldOffset(0)]
    public long counter1;

    [FieldOffset(128)]
    public long counter2;

    [FieldOffset(256)]
    public long counter3;

    [FieldOffset(384)]
    public long counter4;

    [FieldOffset(512)]
    public long counter5;

    [FieldOffset(640)]
    public long counter6;

    [FieldOffset(768)]
    public long counter7;

    [FieldOffset(896)]
    public long counter8;
}
```

可以通过  Intel® VTune™ Performance Analyzer 或者 Intel® Performance Tuning Utility、Visual Studio Profiler 以及 性能计数器 的表现来发现潜在的 false sharing。

参考： 
> [False sharing is no fun](http://joeduffyblog.com/2009/10/19/false-sharing-is-no-fun/)  
> [Avoiding and Identifying False Sharing Among Threads](https://software.intel.com/en-us/articles/avoiding-and-identifying-false-sharing-among-threads)  
> [False Sharing](https://msdn.microsoft.com/magazine/cc872851.aspx)