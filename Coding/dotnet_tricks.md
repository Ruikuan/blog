# dotnet core 的一些窍门

### 逆序访问数组避免 bound check

假如有个数组 array[n]，使用顺序访问的方式，即从 0...n-1 分别访问数组元素，那么每次访问都会有一个 bound check。但若先访问最后一个元素 array[n-1]，
再访问其他元素，则只在访问 [n-1] 元素时有一次 bound check，其他的不会。所以数组遍历顺序最好为 n-1...0。[这里]（https://github.com/aspnet/AspNetCore/pull/6784/commits/ec63f464eabca7011f1d1c6cf871646d74f1be43） 有个示例。
