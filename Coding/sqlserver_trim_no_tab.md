# SQL Server 的 ltrim 和 rtrim 函数并不会清除 tab 空格

使用 Entity framework 时，写如下代码：

```cs

var toFindTitle = "xxx";
var blogs = blogs.Where(x => x.Title.Trim() == toFindTitle.Trim()).ToList();

```

Entity Framework 在翻译上面 Lambda 表达式时会将 `x.Title.Trim()` 翻译为:

```sql
(LTRIM(RTRIM([blog].[Title])) = @toFindTitle_Trim)
```

即将 C# 的 `Trim()` 翻译为 SQL 的 `LTRIM(RTRIM())`，然而这两者并不是一样的，C# 字符串的 `Trim()` 支持清除头尾的 `tab` 空格，而 SQL Server 的 `LTRIM` 和 `RTRIM` 函数并不会。因此万一存储的 blog 的 Title 头尾为 `tab` 空格，那么执行的结果将不符合预期，直接复制存储的 Title 来进行查找也找不到内容。

那么，SQL Server 中如何清除头尾的 `tab` 呢？

```sql
ltrim(rtrim(replace(@str, char(9), '    ')))
```