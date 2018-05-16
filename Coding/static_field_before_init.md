# 避免 .net core 中访问 static field 的性能问题

由于规范的限制（`beforefieldinit`），clr 在访问 static field 之前要先确认它已经被初始化了，因此每次访问 static field 都会导致一次初始化检查，对性能敏感的应用会造成伤害。.net core 就存在这个问题，而 .net framework 4.7 比较激进，直接就不检查了，虽然性能比较好，但其实这并不是符合规范的行为。

考虑如下代码，在 .net framework 和 .net core 中运行的性能就有很大的差别。

```cs
using System;
using System.Runtime.CompilerServices;
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

namespace BenchmarkApp
{
    class Program
    {
        static void Main(string[] args)
        {
            BenchmarkRunner.Run<CustomBenchmarks>();
            Console.ReadLine();
        }
    }

    public class CustomBenchmarks
    {
        private static object _staticField = new object();
        [Benchmark] public object ReadStaticField() => _staticField;
    }
}
```

在 .net framework 4.7.1 的结果为：

```
             Method |      Mean |     Error |    StdDev |
------------------- |----------:|----------:|----------:|
    ReadStaticField | 0.0030 ns | 0.0086 ns | 0.0126 ns |
```

在 .net core 2.1 rc1 的结果为：

```
            Method |      Mean |     Error |    StdDev |
------------------ |----------:|----------:|----------:|
   ReadStaticField | 2.8088 ns | 0.1467 ns | 0.1372 ns |
```

显然 .net core 2.1 rc1 比 .net framework 4.7.1 慢得多。

分析它们 JIT 生成的代码分别为：

.framework 4.7.1
```asm
; static field access
00007FF9CCE90550  mov         rax,1F9EAE45A38h
00007FF9CCE9055A  mov         rax,qword ptr [rax]
00007FF9CCE9055D  ret
```

.net core 2.1 rc1
```asm
; static field access
00007FF9B3EF1510  sub         rsp,28h
00007FF9B3EF1514  mov         rcx,7FF9B3DD4B30h
00007FF9B3EF151E  mov         edx,3
00007FF9B3EF1523  call        00007FFA139E1D40        ; ensure the static fields are initialized
00007FF9B3EF1528  mov         rax,221AB092940h
00007FF9B3EF1532  mov         rax,qword ptr [rax]
00007FF9B3EF1535  add         rsp,28h
00007FF9B3EF1539  ret
```

多了跟比较相关的不少指令，因此慢了很多。

## 解决方法

那么，应该如何解决这个问题呢？

在非泛型的情况下，只需要简单地给 `CustomBenchmarks` 类加上一个 `static` 构造方法就行了。

```cs
public class CustomBenchmarks
{
    private static object _staticField = new object();
    private object _instanceField = new object();

    static CustomBenchmarks() { }       // <-- add this one
    // ...
}
```

.net core 2.1 rc1 就会生成直接访问的指令。

对于泛型的版本，在值类型的情况下，由于 jit 对每种值类型都会生成独立的实际实现，因此上面的方法也是可行的。对于引用类型，由于不同的引用类型共用同一个实现，所以 jit 还是会每次都会先检查是否已经初始化。建议在这种情况下，使用一个实例变量来将静态变量缓存起来，后面都使用实例变量来避免性能降低。

另外，还可以通过启用 `tiered compilation`，让 jit 能够根据实际的运行情况对生成的指令进行再优化来避免这种情况。目前这个选项需要手工打开，以后将会是默认的选项。

> 参考：  
> [Strange performance fluctuations with static fields access](https://github.com/dotnet/coreclr/issues/17981)
