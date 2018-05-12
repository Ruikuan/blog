# 不要使用 struct 默认的 GetHashCode

经常有人在 `Dictionary<TKey,TValue>` 或 `HashSet<T>` 中误用没有自定义 `GetHashCode` 的 `struct` 做 key，出现性能问题。这是默认的 `ValueType` 的 `GetHashCode` 实现存在的问题造成的。

`GetHashCode` 有个原则是：对于所有 `Equals` 的对象，它们的 `GetHashCode` 应该返回相等的值。但反之则没有要求，也就是并没有要求不一样的对象一定要返回不同的 `code`，这在物理上是不可能做到的，因为 `int` 的范围相当有限。

CLR 中对于 `struct` 的 `GetHashCode` 的默认实现为，先检查 `struct` 的所有字段，分两种情况处理：

* 对于所有字段都是值类型，而且中间没有间隙的（由于 `struct` 的对齐布局，字段中间可能产生间隙，像 `{bool,int}` 这样的结构就会在 `bool` 和 `int` 之间产生 3 个字节的间隙），则 CLR 会对其中所有的值的每 32 位做 `XOR`，从而生成 `hashcode`。这种生成的方式会利用到所有的字段内容，因为属于比较“好”的 `hashcode`。不过这里也存在一个鲜为人知的 bug：如果 `struct` 包含 ` System.Decimal`，由于 `System.Decimal` 的字节并不代表它的值，所以对于相同值的 `decimal`，可能生成的 `hashcode` 并不一样。*只是 `struct` 包含 'decimal' 的情况，而不是 `decimal` 自身的情况*。

```cs
struct Test { public decimal value; }

static void Main() {
    var t1 = new Test() { value = 1.0m };
    var t2 = new Test() { value = 1.00m };
    if (t1.GetHashCode() != t2.GetHashCode())
        Console.WriteLine("gack!");
}
```

* 对于字段中有引用类型，或者字段中间有间隙的，CLR 会枚举 `struct` 的所有字段，选择 **第一个** 可用于生成 `hashcode` 的字段（值类型字段或者非空的引用类型），使用它的 `GetHashCode` 方法获得它的 `hashcode`，再跟 `struct` 的 `MethodTable` 指针做 `XOR`，最终的结果就作为了整个 `struct` 的 `hashcode`。对于这种情况，因为它只选取了第一个字段来生成 `hashcode`，所以其他字段的内容跟生成的 `hashcode` 没有关系，即使这些内容完全不一样，生成的 `hashcode` 都是一样的，这样就造成了大量的 `hashcode` 冲突，从而严重影响性能。

```cs
struct Test
{
    public int i;
    public string s; //不管 s 的内容是什么都影响不了 hashcode
}
```

## 结论

由于 `struct` 的 layout 经常不可控（调换下字段的顺序就可能造成影响），以及情况 2 的存在，使用默认的 `GetHashCode` 实现是很不明智的。对于自定义的 `struct`，绝不要使用默认实现，应该自定义靠谱的 `GetHashCode`。

## 番外

对于 `Enum`，调用它的 `Equals` 会导致装箱（之前版本的 clr 调用 `GetHashCode` 也会，目前版本的改过来了）。为了避免这种情况，请调用 ` EqualityComparer<Enum>.Default.Equals(enum1, enum2)`。