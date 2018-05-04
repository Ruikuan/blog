# All about Span

## 什么是 `Span<T>`

`System.Span<T>` 是表示任意的连续内存的一个 value type，无论是托管内存、栈分配的内存还是 interop 的 native code 内存。提供高性能且安全的 access。

通过 `Span<T>` 访问数组

```cs
var arr = new byte[10];
Span<byte> bytes = arr; // Implicit cast from T[] to Span<T>
```

访问其中的一部分

```cs
Span<byte> slicedBytes = bytes.Slice(start: 5, length: 2);
slicedBytes[0] = 42;
slicedBytes[1] = 43;
Assert.Equal(42, slicedBytes[0]);
Assert.Equal(43, slicedBytes[1]);
Assert.Equal(arr[5], slicedBytes[0]);
Assert.Equal(arr[6], slicedBytes[1]);
slicedBytes[2] = 44; // Throws IndexOutOfRangeException
bytes[2] = 45; // OK
Assert.Equal(arr[2], bytes[2]);
Assert.Equal(45, arr[2]);
```

访问栈上分配的内存

```cs
Span<byte> bytes = stackalloc byte[2]; // Using C# 7.2 stackalloc support for spans
bytes[0] = 42;
bytes[1] = 43;
Assert.Equal(42, bytes[0]);
Assert.Equal(43, bytes[1]);
bytes[2] = 44; // throws IndexOutOfRangeException
```

访问 native heap 内存

```cs
IntPtr ptr = Marshal.AllocHGlobal(1);
try
{
  Span<byte> bytes;
  unsafe { bytes = new Span<byte>((byte*)ptr, 1); }
  bytes[0] = 42;
  Assert.Equal(42, bytes[0]);
  Assert.Equal(Marshal.ReadByte(ptr), bytes[0]);
  bytes[1] = 43; // Throws IndexOutOfRangeException
}
finally { Marshal.FreeHGlobal(ptr); }
```

`Span<T>` 索引器利用了 C# 7.0 的新特性 ref return，用 `ref T` 来定义返回值，因此返回的是实际存储位置的引用，而不是返回一个存储值的 copy。

```cs
public ref T this[int index] { get { ... } }
```

`Span<T>` 的另一个变体是 `System.ReadOnlySpan<T>`，用来进行只读访问。返回值为 C# 7.2 新特性提供的 `ref readonly T`，用来操作类似 `string` 等不可变序列。

```cs
string str = "hello, world";
string worldString = str.Substring(startIndex: 7, length: 5); // Allocates 
ReadOnlySpan<char> worldSpan = str.AsReadOnlySpan().Slice(start: 7, length: 5); // No allocation
Assert.Equal('w', worldSpan[0]);
worldSpan[0] = 'a'; // Error CS0200: indexer cannot be assigned to
```

`Span<T>` 支持重新解释（reinterpret）的转换，也即可以将 `Span<byte>` 转换到 `Span<int>`，其中 `Span<int>[0]` 对应 `Span<byte>[0][1][2][3]`，这样可以读取一个 byte buffer，然后将它传到方法里分组为 int 进行处理，安全且高效。

## `Span<T>` 的实现

`Span<T>` 是一个包含一个 ref 和 length 的值类型，接近如下声明：

```cs
public readonly ref struct Span<T>
{
  private readonly ref T _pointer; // There is not ref in C# or MSIL
  private readonly int _length;
  ...
}
```

不过在 C# 和 MSIL 中都无法定义一个 ref 字段，实际上是 JIT 对 `Span<T>` 进行了特别对待，为它生成对应 ref 字段的 JIT 指令。这里的 ref 使用跟方法的 ref 参数类似，就是直接指向 location 的引用。这种直接或间接包含有这种引用的类型被称之为 `ref-like type`，C# 7.2 允许通过 `ref struct` 来定义这种类型。

1. `Span<T>` 被定义为一种跟数组一样高效的方式：对 `Span<T>` 进行索引操作是直接的，无需根据开始指针和偏移来计算。（` ArraySegment<T>` 就需要进行这样的换算。）
2. 由于 `Span<T>` 是 `ref-like type`，所以它会收到限制。

## `Memory<T>` 是什么以及我们为什么需要它

`Span<T>` 不仅能指向对象或数组的开头，也能指向它们中间的一段，这种引用称之为内部指针，而 GC 处理这种情况代价不轻，所以只允许这种 ref 在栈中出现。另外，`Span<T>` 大小超过了一个字长，这意味着读和写 `Span<T>` 不是一个原子操作。如果允许多个线程同时操作它的字段，可能引起撕裂。因此，`Span<T>` 的实例被限制为只能在栈上生存，而不能在堆中。这意味着不能装箱`Span<T>`，也不能将它用于反射 API（需装箱），不能在类中包含 `Span<T>` 的字段，甚至不能在非 ref sturct 中包含 `Span<T>` 字段。而且即使是隐式会将 `Span<T>` 转化为此类字段的使用场景都不行，例如通过捕获它们到 lambda 中 / async 方法中 / 迭代器中。也不能将 `Span<T>` 作为泛型参数使用。

这种限制在很多场景下都无所谓，特别是对于只搞计算的同步方法。但对异步方法就很麻烦了，所以引入了 `Memory<T>`。还有它对应的 `ReadOnlyMemory<T>`。

```cs
// Memory<T> looks very much like an ArraySegment<T>:
public readonly struct Memory<T>
{
  private readonly object _object;
  private readonly int _index;
  private readonly int _length;
  ...
}
```

可以在它们之间互相转换使用。

## `Span<T>` 和 `Memory<T>` 如何跟类库进行整合

无数新的使用 `{ReadOnly}Span<T>` 和 `{ReadOnly}Memory<T>` 的 API 被加入到类库中。例如 int 之类的基元类型，都加入了接受 `ReadOnlySpan<char>` 的 `Parse` 方法，以进行 0 分配的操作。这些包含了新 API 重载的类型包括 `System.Random`，`System.Text.StringBuilder`，`System.Net.Sockets`。而且很多 API 不但支持新的 `Span<T>`，而且将返回值变成了 `ValueTask<T>`，以在同步操作或者已经有缓存数据的时候直接返回时不需要分配新的对象。例如 `Stream.ReadAsync`。

另外，`Span<T>` 允许框架包含一些以往会引起内存安全忧虑的方法，比如以往生成一个随机字符串，如下处理：

```cs
int length = ...;
Random rand = ...;
var chars = new char[length];
for (int i = 0; i < chars.Length; i++)
{
  chars[i] = (char)(rand.Next(0, 10) + '0');
}
string id = new string(chars);
```

用 `Span<char>` 可以在栈上分配，并避免 `unsafe` 的使用，结合新的接受 `ReadOnlySpan<char>` 的字符串构造方法，可以将生成算法修改为：

```cs
int length = ...;
Random rand = ...;
Span<char> chars = stackalloc char[length];
for (int i = 0; i < chars.Length; i++)
{
  chars[i] = (char)(rand.Next(0, 10) + '0');
}
string id = new string(chars);
```

这样就避免了在堆上做临时的字符串数组分配，但还是需要一次额外的复制操作将生成数据从栈上复制到字符串中。而且这种使用方式只适用于字符串小的情况，如果长的话就会导致栈溢出。那么能否直接写道字符串所在的内存呢？`Span<T>` 让你可以这么做。在 `string` 新的构造函数之外，它还多了个 `Create` 方法：

```cs
public static string Create<TState>(
  int length, TState state, SpanAction<char, TState> action);
...
public delegate void SpanAction<T, in TArg>(Span<T> span, TArg arg);
```

这个方法允许你填充可写的 `Span<T>` 来生成 `string`。由于 `Span<T>` 只能在栈上生存的特质，这里可以保证在字符串构造完之前，对应的 `Span<T>` 就已经终结了，无法再在其他地方使用，无法在字符串构造完成之后再来更改字符串。

```cs
int length = ...;
Random rand = ...;
string id = string.Create(length, rand, (Span<char> chars, Random r) =>
{
  for (int i = 0; chars.Length; i++)
  {
    chars[i] = (char)(r.Next(0, 10) + '0');
  }
});
```

这样就可以直接在字符串空间中任意填充了，不仅避免了复制，而且没有了大小的限制。

除了核心的框架类型有了新的成员之外，还有很多的新的 .NET 类型被开发出来跟 `Span<T>` 一起对特定的场景进行高效处理。例如，开发人员发现如果他们在 UTF-8 上处理字符串不需要 encode 和 decode 的话会获得显著的性能提升。新的类型如 `System.Buffers.Text.Base64`，`System.Buffers.Text.Utf8Parser` 和 `System.Buffers.Text.Utf8Formatter` 被加了进来。它们对字节进行处理，不单避免了 Unicode 的编解码，还允许它们直接在各种网络栈很底层中很常见的 native buffer 上进行操作：

```cs
ReadOnlySpan<byte> utf8Text = ...;
if (!Utf8Parser.TryParse(utf8Text, out Guid value,
  out int bytesConsumed, standardFormat = 'P'))
  throw new InvalidDataException();
```

除了公开出来的这些方法，框架内部也开始利用这些新的基于 `Span<T>` 和 `Memory<T>` 的方法来获得更好的性能。.NET Core 内的调用已经切换到使用新的 `ReadAsync` 重载以避免不必要的分配。

ASP.NET Core 现在也重度依赖 `Span<T>`，例如 Kestrel Server 的 HTTP parser 现在就是在它之上构建的。将来，很可能 span 会在更底层的公开 API 中暴露出来，例如在中间件管道中。

## .NET Runtime

.net runtime 致力于消除不必要的边界检测，以提高 `Span<T>` 的使用性能。

## C# 和 编译器

C# 7.2 中引入了跟 span 相关的若干特性（事实上 C# 7.2 的编译器就必须使用 `Span<T>`）。

**Ref struct**。只能生存在栈上的会传染的值类型。所有想要在字段中包含 ref-like type 的类型都必需定义为 ref struct。例如你想要定义一个用于 `Span<T>` 的 Enumerator：

```cs
public ref struct Enumerator
{
  private readonly Span<char> _span;
  private int _index;
  ...
}
```

**对 `Span<T>` 的 stackalloc**。在以往的 C# 版本中，stackalloc 结果保存在本地的一个指针变量中。在 C# 7.2 中，stackalloc 能作为表达式的一部分使用并且能指向一个 `span`，而且不需要使用 `unsafe` 关键字。所以，以往的写法：

```cs
Span<byte> bytes;
unsafe
{
  byte* tmp = stackalloc byte[length];
  bytes = new Span<byte>(tmp, length);
}
```
变成了更简单的：

```cs
Span<byte> bytes = stackalloc byte[length];
```

对需要一些小空间来进行操作但又想避免在堆中分配的情形特别有用。

```cs
Span<byte> bytes = length <= 128 ? stackalloc byte[length] : new byte[length];
... // Code that operates on the Span<byte>
```

**`Span` 使用验证**。避免 `Span` 从使用栈帧逃离出去引起内存安全问题。

