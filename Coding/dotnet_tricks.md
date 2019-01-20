# dotnet core 性能相关的一些小技巧

这里的技巧只针对目前 dotnet core 2.2 的版本。后面可能带来更多的优化，可能会带来不同。

### 避免 bound check

如果使用 for 循环访问，则 bound check 会被优化掉。
```cs
for(int i = 0; i < array.Length; i++)
{
  result += array[i]; //clr了解这种模式，消除了这里的 bound check
}
```

使用顺序访问的方式，即从 0...n-1 分别访问数组元素，那么每次访问都会有一个 bound check。
```cs
public static void Test1(char[] array)
{
  //每一个数组元素访问都会导致 bound check
  array[0] = 'F';
  array[1] = 'a';
  array[2] = 'l';
  array[3] = 's';
  array[4] = '3';
  array[5] = '.';
}
```
但若先访问最后一个元素 array[n-1]，再访问其他元素，则只在访问 [n-1] 元素时有一次 bound check，其他的不会。
```cs
public static void Test1(char[] array)
{
  array[5] = '.'; //只有这次访问有 bound check
  array[0] = 'F';
  array[1] = 'a';
  array[2] = 'l';
  array[3] = 's';
  array[4] = '3';
}
```
也可以先进行一下数组长度判断，可以避免 bound check，但需要注意将 `array.Length` 转换为 `uint` 才行。

```cs
public static void Test1(char[] array)
{
  // if(array.Length > 5) 无法消除 bound check
  if((uint)array.Length > 5) //转化为 uint 之后就行了。不知道为什么。
  {
    array[0] = 'F';
    array[1] = 'a';
    array[2] = 'l';
    array[3] = 's';
    array[4] = '3';
    array[5] = '.';
  }
}
```

[asp.net core 代码库的这里](https://github.com/aspnet/AspNetCore/pull/6784/commits/ec63f464eabca7011f1d1c6cf871646d74f1be43) 有个示例。



### 分配一个临时数组

```cs
char[] pool = null;
Span<char> span =
  count <= 512 ?
  stackalloc char[512] :
  (pool = ArrayPool<char>.Shared.Rent(count));
  
if (pool != null)
  ArrayPool<char>.Shared.Return(pool);
```
### 类型转换的代价并不小

```cs
object value = new List<string> {};

var t0 = (List<string>)value;         //0.3 ns 转化为自身实体类型
var t1 = (ICollection<string>)value;  //2.0 ns 接口
var t2 = (IList)value;                //3.0 ns 
var t3 = (IEnumerable<string>)value;  //30.5 ns 转化为协变接口 IEnumerable<out T>
```


