# 一种批量更新数据的 SQL Server 写法

2个表，表 A 有字段:  A1和A2  
      表 B 有字段:  B1和B2

欲更新 A1 字段，条件是 A2 = B2，更新内容为 B1。

可以采用如下写法：
```sql
UPDATE A
SET A1 = B1
FROM A, B
WHERE A.A2 = B.B2
```