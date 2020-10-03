## Razor实现代码生成器Demo

因为想搞一个工具来生成一些dapper的模型类，T4的语法不太熟悉，而且T4需要预编译，所以就想着用大家都会用的Razor来实现代码生成器。

首先来看下如何来获取数据库表的信息，这里只用sql server来举例
```sql
select b.name [Schema], a.name [Name], c.value [Description] from test.sys.tables a
inner join sys.schemas b on a.schema_id = b.schema_id
left join sys.extended_properties c on c.major_id=a.object_id and c.minor_id=0 and c.class=1 
where b.name = 'dbo' and a.Name = 'Test1'
```
上面的这段sql可以获取数据库的schema、表名和备注，那么如何获取一张表的所有字段信息呢？这里就直接贴代码了
```sql
select  
		a.Name [Name],  --字段名
		isnull(e.[value],'') [Description],  --备注
		b.Name [SqlType],  --字段类型  
		case when is_identity=1 then 1 else 0 end [IsIdentity],  --是否为标识列
		case when exists(select 1 from sys.objects x join sys.indexes y on x.Type=N'PK' and x.Name=y.Name  
            join sysindexkeys z on z.ID=a.Object_id and z.indid=y.index_id and z.Colid=a.Column_id)  
            then 1 else 0 end [IsKey],  --是否为主键  
		case when a.is_nullable=1 then 1 else 0 end [IsNullable], --是否为可空的
		isnull(d.text,'') [DefaultValue] --默认值  
from Test.INFORMATION_SCHEMA.COLUMNS s
inner join  
		sys.columns a on s.COLUMN_NAME COLLATE Chinese_PRC_CI_AS = a.name
left join 
		sys.types b on a.user_type_id=b.user_type_id  
inner join 
		sys.objects c on a.object_id=c.object_id and c.Type='U' 
left join 
		syscomments d on a.default_object_id=d.ID  
left join
		sys.extended_properties e on e.major_id=c.object_id and e.minor_id=a.Column_id and e.class=1   
where s.TABLE_SCHEMA = 'dbo' and c.name = 'Test1' and s.TABLE_NAME = 'Test1'
order by a.column_id
```
![](https://github.com/zzq424/Blogs/blob/master/images/20201003221926.png?raw=true)


具体的思路是使用Razor来生成C#代码，再使用Roslyn来编译并执行C#代码，把输出保存到文件中。

表结构
![](https://github.com/zzq424/Blogs/blob/master/images/20201003224835.png?raw=true)

```C#
    class Program
    {
        static async Task Main(string[] args)
        {
            var generator = new SqlServerCompiler();
            await generator.GenerateAsync(new ModelConfig
            {
                ConnectionString = "Persist Security Info=False;User ID=sa;Password=xxx;Initial Catalog=test;Data Source=localhost;",
                Database = "test",
                Schema = "dbo",
                Table = "Test1",
                NameSpace = "DataAccess.Model",
                FilePath = Directory.GetCurrentDirectory()
            });
            Console.ReadKey();
        }
    }
```

生成的C#文件
![](https://github.com/zzq424/Blogs/blob/master/images/20201003225143.png?raw=true)


demo已上传至[github]("https://github.com/zzq424/demo/tree/master/code-generator")

