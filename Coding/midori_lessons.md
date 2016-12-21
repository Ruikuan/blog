# Midori 学习片段

[Asynchronous Everything](http://joeduffyblog.com/2015/11/19/asynchronous-everything/)

> 但我想说最主要的指示是避免像瘟疫一样的大量分配对象，即使是短生命周期的那些。早期有一个关于 .NET 的箴言：Gen0 回收是免费的。不幸的是，用这搞出来了大量的库的这句，完全是废话。Gen0 回收导致暂停，
污染缓存（ CPU Cache），还给高并行的系统带来频率节拍问题。  
> 
> 第一个关键的优化，当然是，如果一个 async 方法不进行 await ，就不应该为它分配任何对象。  
