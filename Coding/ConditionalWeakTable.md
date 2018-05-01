# `ConditionalWeakTable<TKey,TValue>` 概括

详情参见 [ConditionalWeakTable&lt;TKey,TValue> Class](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.conditionalweaktable-2)

一个不持有对应 Key 强引用的类 Dictionary 数据结构，如果 Table 外的 Key 已经被干掉了，那么这个 Table 里的这行 Key 连带 Value 记录都会被删掉。Table 外的意思包括，即使 Table 里面存在指向 Key 的引用，也不会阻止删除行为。所以，这个数据结构的行为更像一个 Table，而不是 Dictionary。

这种特性可以用于给运行中的对象附加额外的属性。需要使用对象额外属性的时候，可以很轻松查找，而原对象已经不需要的时候，也不用费心去处理 Table 里面的数据。