最近在工作中用到到了sql server的with..as，在此记录下它的用法

#### 1．WITH AS的含义

WITH AS短语，也叫做子查询部分（subquery factoring）。查询的结果集被称为公用表表达式(CTE), 公用表表达式可以包括对自身的引用， 这种表达式称为递归公用表表达式。
对于UNION ALL，使用WITH AS定义了一个UNION ALL语句，当该片断被调用2次以上，优化器会自动将数据放入一个临时表中，使用这种方式可以提高查询速度。

#### 2. WITH AS的用法

WITH AS的语法如下，还是比较简单的

```sql
[ WITH <common_table_expression> [ ,...n ] ]  
  
<common_table_expression>::=  
    expression_name [ ( column_name [ ,...n ] ) ]  
    AS  
    ( CTE_query_definition )
```


使用CTE的时候有几点是需要注意的
1. CTE后面必须直接跟使用它的SQL语句（如select、insert、update等）
2. CTE后面也可以跟其他的CTE，但只能使用一个with，多个CTE中间用逗号（,）分隔
3. CTE 可以引用自身，也可以引用在同一 WITH 子句中预先定义的 CTE。但不允许前向引用

#### 3. 使用WITH AS短语实现递归查询

表定义
```sql
CREATE TABLE [dbo].[Menu] (
  [Id] int NOT NULL,                  -- 主键，菜单Id
  [Name] nvarchar(20) NOT NULL,       -- 名称
  [ParentId] int DEFAULT(0) NOT NULL, -- 父级菜单Id
  PRIMARY KEY CLUSTERED ([Id])
)
```

向上递归查询
```sql
with temp as
(
	select Id, Name, ParentId
    from Menu
	union all
	select t.Id, t.Name, t.ParentId
	from Menu t, temp where t.Id = temp.ParentId
)
select * from temp 
```

向下递归查询
```sql
with temp as
(
	select Id, Name, ParentId
    from Menu
	union all
	select t.Id, t.Name, t.ParentId
	from Menu t, temp where t.ParentId = temp.Id
)
select * from temp 
```
区别在于：t.Id = temp.ParentId和 t.ParentId = temp.Id 表示了不同的查询方向

根据name进行模糊查询，同时返回Leaf表示是否为叶子节点，以供前端渲染
```sql
with temp as
(
	select Id, Name, ParentId
    from Menu p where name like '%Key%'
	union all
	select t.Id, t.Name, t.ParentId
	from Menu t, temp where t.Id = temp.ParentId
)
select distinct *,
 CAST(ISNULL((select top 1 0 from Menu with(nolock) where Menu.ParentId = t1.Id), 1) AS bit) Leaf 
 from (
	select distinct * from temp
) t1 
order by Id
```