# .net 中接口声明顺序会对性能造成影响

在 .net 中，将实例转换为接口，是按照接口的声明顺序进行线性查找的，因此某个类实现接口的声明顺序，会对它进行接口访问的性能有影响。对注重高性能的地方尤其需要注意，将注重性能和经常用到的接口放到前面，而将不常用的放到后面。

corefx 中就因为 string 实现接口的顺序变化，造成了[大约 30% 的性能倒退](https://github.com/dotnet/corefx/issues/29158)。[修复的方式](https://github.com/dotnet/coreclr/pull/17660/files)如下：  
![string interface order](https://github.com/Ruikuan/blog/raw/master/Content/interface_order.png)

