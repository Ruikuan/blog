# SQL Server CTE 无限递归的解决方法

假设有表 `Node`，具有 `Id` 和 `ParentId` 两字段，并且子节点的 `ParentId` 关联到父节点的 `Id`。现在我们要从一个节点出发，遍历出它所有的祖先节点，这时候我们一般用 [CTE](https://docs.microsoft.com/en-us/sql/t-sql/queries/with-common-table-expression-transact-sql) 来做递归查询：

```sql
;with Tree as
(
	select ParentId
	from Node
    where Id = @id

	select ParentId
	from Tree T
		inner join Node n on Node.Id = T.ParentId
)
```

但有时候问题在于这些 `Node` 的关系并不是一棵单纯的树，而是之间可能有环形关联的图。比如 A->B->C->D，还有 D->A->C，这样遍历的话，由于存在环形连接，数据库会无线递归查找下去，最后抛出一个达到最大递归值错误。为了解决这个问题，我们可以使用下面这样的一个小花招。目的是检查节点的 `Id` 是否出现过在当前查找过的路径中，如果有，就不能再选择它。

```sql
;with Tree as
(
	select ParentId
        , cast(ParentId as varchar(max)) as linkedIds
	from Node
    where Id = @id

	select n.ParentId
        , T.linkedIds + '>' + cast(n.ParentId as varchar(max)) as linkedIds
	from Tree T
		inner join Node n on Node.Id = T.ParentId and ( T.linkedIds not like '%'+ cast(n.ParentId as varchar(max)) + '%')
)
```

这个 trick 不高效，也不直观，但能够简单应对数据量不是太多的情况。不过这种方法虽然不会进入无限递归循环，但不能保证同样的 Node 在结果中只出现一次，由于有不同的路径通向同样的 Node，所以它会出现多次。若独一性很重要，就需要在外层查询语句中使用 `distinct` 。