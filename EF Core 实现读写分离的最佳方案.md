## EF Core 实现读写分离的最佳方案

#### 前言
公司之前使用Ado.net和Dapper进行数据访问层的操作, 进行读写分离也比较简单, 只要使用对应的数据库连接字符串即可. 而最近要迁移到新系统中,新系统使用.net core和EF Core进行数据访问. 所以趁着国庆假期拿出一两天时间研究了一下如何EF Core进行读写分离.

#### 思路
根据园子里的Jeffcky大神的博客, 参考 
[EntityFramework Core进行读写分离最佳实践方式，了解一下（一）？]("https://www.cnblogs.com/CreateMyself/p/9241523.html")
[EntityFramework Core进行读写分离最佳实践方式，了解一下（二）？]("https://www.cnblogs.com/CreateMyself/p/9261435.html")

最简单的思路就是使用手动切换EF Core上下文的连接, 即context.Database.GetDbConnection().ConnectionString = "xxx", 但必须要先创建上下文, 再关闭之前的连接, 才能进行切换
另一种方式是通过监听Diagnostic来将进行查询的sql切换到从库执行, 这种方式虽然可以实现无感知的切换操作, 但不能满足公司的业务需求. 在后台管理或其他对数据实时性要求比较高的项目里,查询操作也都应该走主库,而这种方式却会切换到从库去. 另一方面就是假若公司的库比较多,每种业务都对应了一个库, 每个库都对应了一种DbContext, 这种情况下, 要实现自动切换就变得很复杂了.

上面的两种方式都是从切换数据库连接入手,但是频繁的切换数据库连接势必会对性能造成影响. 我认为最理想的方式是要避免数据库连接的切换, 且能够适应多DbContext的情况, 在创建上下文实例时,就指定好是访问主库还是从库, 而不是在后期再进行数据库切换. 因此, 在上下文实例化时,就传入相应的数据库连接字符串, 这样一来DbContext的创建就需要交由我们自己来进行, 就不是由DI容器进行创建了. 同时仓储应该区分为只读和可读可写两种,以防止其他人对从库进行写操作. 

#### 实现

```c#
	public interface IReadOnlyRepository<TEntity, TKey>
        where TEntity : class, IEntity<TKey>
        where TKey : IEquatable<TKey>
	{}

	public interface IRepository<TEntity, TKey> : IReadOnlyRepository<TEntity, TKey>
    where TEntity : class, IEntity<TKey>
    where TKey : IEquatable<TKey>
	{}
```
IReadOnlyRepository接口是只读仓储接口,提供查询相关方法,IRepository接口是可读可写仓储接口,提供增删查改等方法, 接口的实现就那些东西这里就省略了.

```c#
    public interface IRepositoryFactory
    {
        IRepository<TEntity, TKey> GetRepository<TEntity, TKey>(IUnitOfWork unitOfWork)
            where TEntity : class, IEntity<TKey>
            where TKey : IEquatable<TKey>;
         IReadOnlyRepository<TEntity, TKey> GetReadOnlyRepository<TEntity, TKey>(IUnitOfWork unitOfWork)
                where TEntity : class, IEntity<TKey>
                where TKey : IEquatable<TKey>;
    }
    public class RepositoryFactory : IRepositoryFactory
    {
        public RepositoryFactory()
        {
        }

        public IRepository<TEntity, TKey> GetRepository<TEntity, TKey>(IUnitOfWork unitOfWork)
            where TEntity : class, IEntity<TKey>
            where TKey : IEquatable<TKey>
        {
            return new Repository<TEntity, TKey>(unitOfWork);
        }

        public IReadOnlyRepository<TEntity, TKey> GetReadOnlyRepository<TEntity, TKey>(IUnitOfWork unitOfWork)
            where TEntity : class, IEntity<TKey>
            where TKey : IEquatable<TKey>
        {
            return new ReadOnlyRepository<TEntity, TKey>(unitOfWork);
        }
    }
```
RepositoryFactory提供仓储对象的实例化
```c#
    public interface IUnitOfWork : IDisposable
    {
        public DbContext DbContext { get; }

        /// <summary>
        /// 获取只读仓储对象
        /// </summary>
        IReadOnlyRepository<TEntity, TKey> GetReadOnlyRepository<TEntity, TKey>()
            where TEntity : class, IEntity<TKey>
            where TKey : IEquatable<TKey>;

        /// <summary>
        /// 获取仓储对象
        /// </summary>
        IRepository<TEntity, TKey> GetRepository<TEntity, TKey>()
            where TEntity : class, IEntity<TKey>
            where TKey : IEquatable<TKey>;
        int SaveChanges();
        Task<int> SaveChangesAsync(CancellationToken cancelToken = default);
    }
    
    public class UnitOfWork : IUnitOfWork
    {
      	private readonly IServiceProvider _serviceProvider;
        private readonly DbContext _dbContext;
        private readonly IRepositoryFactory _repositoryFactory;
        private bool _disposed;

        public UnitOfWork(IServiceProvider serviceProvider, DbContext context)
        {
            Check.NotNull(serviceProvider, nameof(serviceProvider));
            _serviceProvider = serviceProvider;
            _dbContext = context;
            _repositoryFactory = serviceProvider.GetRequiredService<IRepositoryFactory>();
        }
        public DbContext DbContext { get => _dbContext; }
        public IReadOnlyRepository<TEntity, TKey> GetReadOnlyRepository<TEntity, TKey>()
            where TEntity : class, IEntity<TKey>
            where TKey : IEquatable<TKey>
        {
            return _repositoryFactory.GetReadOnlyRepository<TEntity, TKey>(this);
        }

        public IRepository<TEntity, TKey> GetRepository<TEntity, TKey>()
            where TEntity : class, IEntity<TKey>
            where TKey : IEquatable<TKey>
        {
            return _repositoryFactory.GetRepository<TEntity, TKey>(this);
        }
        
        public void Dispose()
        {
            if (_disposed)
            {
                return;
            }

            _dbContext?.Dispose();
            _disposed = true;
        }
        
        // 其他略
    }
```
```c#
    /// <summary>
    /// 数据库提供者接口
    /// </summary>
    public interface IDbProvider : IDisposable
    {
        /// <summary>
        /// 根据上下文类型及数据库名称获取UnitOfWork对象, dbName为null时默认为第一个数据库名称
        /// </summary>
        IUnitOfWork GetUnitOfWork(Type dbContextType, string dbName = null);
    }
```
IDbProvider 接口, 根据上下文类型和配置文件中的数据库连接字符串名称创建IUnitOfWork, 在DI中的生命周期是Scoped,在销毁的同时会销毁数据库上下文对象, 下面是它的实现, 为了提高性能使用了Expression来代替反射.
```c#
public class DbProvider : IDbProvider
    {
        private readonly IServiceProvider _serviceProvider;
        private readonly ConcurrentDictionary<string, IUnitOfWork> _works = new ConcurrentDictionary<string, IUnitOfWork>();
        private static ConcurrentDictionary<Type, Func<IServiceProvider, DbContextOptions, DbContext>> _expressionFactoryDict =
            new ConcurrentDictionary<Type, Func<IServiceProvider, DbContextOptions, DbContext>>();

        public DbProvider(IServiceProvider serviceProvider)
        {
            _serviceProvider = serviceProvider;
        }

        public IUnitOfWork GetUnitOfWork(Type dbContextType, string dbName = null)
        {
            var key = string.Format("{0}${1}$", dbName, dbContextType.FullName);
            IUnitOfWork unitOfWork;
            if (_works.TryGetValue(key, out unitOfWork))
            {
                return unitOfWork;
            }
            else
            {
                DbContext dbContext;
                var dbConnectionOptionsMap = _serviceProvider.GetRequiredService<IOptions<FxOptions>>().Value.DbConnections;
                if (dbConnectionOptionsMap == null || dbConnectionOptionsMap.Count <= 0)
                {
                    throw new Exception("无法获取数据库配置");
                }

                DbConnectionOptions dbConnectionOptions = dbName == null ? dbConnectionOptionsMap.First().Value : dbConnectionOptionsMap[dbName];

				var builderOptions = _serviceProvider.GetServices<DbContextOptionsBuilderOptions>()
                     ?.Where(d => (d.DbName == null || d.DbName == dbName) && (d.DbContextType == null || d.DbContextType == dbContextType))
                     ?.OrderByDescending(d => d.DbName)
                     ?.OrderByDescending(d => d.DbContextType);
                if (builderOptions == null || !builderOptions.Any())
                {
                    throw new Exception("无法获取匹配的DbContextOptionsBuilder");
                }

                var dbUser = _serviceProvider.GetServices<IDbContextOptionsBuilderUser>()?.FirstOrDefault(u => u.Type == dbConnectionOptions.DatabaseType);
                if (dbUser == null)
                {
                    throw new Exception($"无法解析类型为“{dbConnectionOptions.DatabaseType}”的 {typeof(IDbContextOptionsBuilderUser).FullName} 实例");
                }
                
                var dbContextOptions = dbUser.Use(builderOptions.First().Builder, dbConnectionOptions.ConnectionString).Options;
                if (_expressionFactoryDict.TryGetValue(dbContextType, out Func<IServiceProvider, DbContextOptions, DbContext> factory))
                {
                    dbContext = factory(_serviceProvider, dbContextOptions);
                }
                else
                {
                    // 使用Expression创建DbContext
                    var constructorMethod = dbContextType.GetConstructors()
                        .Where(c => c.IsPublic && !c.IsAbstract && !c.IsStatic)
                        .OrderByDescending(c => c.GetParameters().Length)
                        .FirstOrDefault();
                    if (constructorMethod == null)
                    {
                        throw new Exception("无法获取有效的上下文构造器");
                    }

                    var dbContextOptionsBuilderType = typeof(DbContextOptionsBuilder<>);
                    var dbContextOptionsType = typeof(DbContextOptions);
                    var dbContextOptionsGenericType = typeof(DbContextOptions<>);
                    var serviceProviderType = typeof(IServiceProvider);
                    var getServiceMethod = serviceProviderType.GetMethod("GetService");
                    var lambdaParameterExpressions = new ParameterExpression[2];
                    lambdaParameterExpressions[0] = (Expression.Parameter(serviceProviderType, "serviceProvider"));
                    lambdaParameterExpressions[1] = (Expression.Parameter(dbContextOptionsType, "dbContextOptions"));
                    var paramTypes = constructorMethod.GetParameters();
                    var argumentExpressions = new Expression[paramTypes.Length];
                    for (int i = 0; i < paramTypes.Length; i++)
                    {
                        var pType = paramTypes[i];
                        if (pType.ParameterType == dbContextOptionsType ||
                            (pType.ParameterType.IsGenericType && pType.ParameterType.GetGenericTypeDefinition() == dbContextOptionsGenericType))
                        {
                            argumentExpressions[i] = Expression.Convert(lambdaParameterExpressions[1], pType.ParameterType);
                        }
                        else if (pType.ParameterType == serviceProviderType)
                        {
                            argumentExpressions[i] = lambdaParameterExpressions[0];
                        }
                        else
                        {
                            argumentExpressions[i] = Expression.Call(lambdaParameterExpressions[0], getServiceMethod);
                        }
                    }

                    factory = Expression
                        .Lambda<Func<IServiceProvider, DbContextOptions, DbContext>>(
                            Expression.Convert(Expression.New(constructorMethod, argumentExpressions), typeof(DbContext)), lambdaParameterExpressions.AsEnumerable())
                        .Compile();
                    _expressionFactoryDict.TryAdd(dbContextType, factory);

                    dbContext = factory(_serviceProvider, dbContextOptions);
                }

                var unitOfWorkFactory = _serviceProvider.GetRequiredService<IUnitOfWorkFactory>();
                unitOfWork = unitOfWorkFactory.GetUnitOfWork(_serviceProvider, dbContext);
                _works.TryAdd(key, unitOfWork);
                return unitOfWork;
            }
        }

        public void Dispose()
        {
            if (_works != null && _works.Count > 0)
            {
                foreach (var unitOfWork in _works.Values)
                    unitOfWork.Dispose();
                _works.Clear();
            }
        }
    }
    
    public static class DbProviderExtensions
    {
        public static IUnitOfWork GetUnitOfWork<TDbContext>(this IDbProvider provider, string dbName = null)
        {
            if (provider == null)
                return null;
            return provider.GetUnitOfWork(typeof(TDbContext), dbName);
        }
    }
```

```c#
    /// <summary>
    /// 业务系统配置选项
    /// </summary>
    public class FxOptions
    {
        public FxOptions()
        {
        }

        /// <summary>
        /// 默认数据库类型
        /// </summary>
        public DatabaseType DefaultDatabaseType { get; set; } = DatabaseType.SqlServer;

        /// <summary>
        /// 数据库连接配置
        /// </summary>
        public IDictionary<string, DbConnectionOptions> DbConnections { get; set; }

    }
    
    public class FxOptionsSetup: IConfigureOptions<FxOptions>
    {
        private readonly IConfiguration _configuration;

        public FxOptionsSetup(IConfiguration configuration)
        {
            _configuration = configuration;
        }

        /// <summary>
        /// 配置options各属性信息
        /// </summary>
        /// <param name="options"></param>
        public void Configure(FxOptions options)
        {
            SetDbConnectionsOptions(options);
            // ...
        }

        private void SetDbConnectionsOptions(FxOptions options)
        {
            var dbConnectionMap = new Dictionary<string, DbConnectionOptions>();
            options.DbConnections = dbConnectionMap;
            IConfiguration section = _configuration.GetSection("FxCore:DbConnections");
            Dictionary<string, DbConnectionOptions> dict = section.Get<Dictionary<string, DbConnectionOptions>>();
            if (dict == null || dict.Count == 0)
            {
                string connectionString = _configuration["ConnectionStrings:DefaultDbContext"];
                if (connectionString == null)
                {
                    return;
                }
                dbConnectionMap.Add("DefaultDb", new DbConnectionOptions
                {
                    ConnectionString = connectionString,
                    DatabaseType = options.DefaultDatabaseType
                });

                return;
            }

            var ambiguous = dict.Keys.GroupBy(d => d).FirstOrDefault(d => d.Count() > 1);
            if (ambiguous != null)
            {
                throw new Exception($"数据上下文配置中存在多个配置节点拥有同一个数据库连接名称，存在二义性：{ambiguous.First()}");
            }
            foreach (var db in dict)
            {
                dbConnectionMap.Add(db.Key, db.Value);
            }
        }
    }
    
    /// <summary>
    /// DbContextOptionsBuilder配置选项
    /// </summary>
    public class DbContextOptionsBuilderOptions
    {
        /// <summary>
        /// 配置DbContextOptionsBuilder, dbName指定数据库名称, 为null时表示所有数据库,默认为null
        /// </summary>
        /// <param name="build"></param>
        /// <param name="dbName"></param>
        /// <param name="dbContextType"></param>
        public DbContextOptionsBuilderOptions(DbContextOptionsBuilder build, string dbName = null, Type dbContextType = null)
        {
            Builder = build;
            DbName = dbName;
            DbContextType = dbContextType;
        }

        public DbContextOptionsBuilder Builder { get; }
        public string DbName { get; }
        public Type DbContextType { get; }
    }
```
FxOptions是业务系统的配置选项(随便取得), 在通过service.GetService<IOptions<FxOptions>>()时会调用IConfigureOptions<FxOptions>完成FxOptions的初始化. DbContextOptionsBuilderOptions用来提供DbContextOptionsBuilder的相关配置
```c# 
    public interface IDbContextOptionsBuilderUser
    {
        /// <summary>
        /// 获取 数据库类型名称，如 SQLSERVER，MYSQL，SQLITE等
        /// </summary>
        DatabaseType Type { get; }

        /// <summary>
        /// 使用数据库
        /// </summary>
        /// <param name="builder">创建器</param>
        /// <param name="connectionString">连接字符串</param>
        /// <returns></returns>
        DbContextOptionsBuilder Use(DbContextOptionsBuilder builder, string connectionString);
    }
    
    public class SqlServerDbContextOptionsBuilderUser : IDbContextOptionsBuilderUser
    {
        public DatabaseType Type => DatabaseType.SqlServer;

        public DbContextOptionsBuilder Use(DbContextOptionsBuilder builder, string connectionString)
        {
            return builder.UseSqlServer(connectionString);
        }
    }
```
IDbContextOptionsBuilderUser接口用来适配不同的数据库来源

#### 使用
```json
{
    "FxCore": {
        "DbConnections": {
            "TestDb": {
                "ConnectionString": "xxx",
                "DatabaseType": "SqlServer"
            },
            "TestDb_Read": {
                "ConnectionString": "xxx",
                "DatabaseType": "SqlServer"
            }
        }
    }
}
```
```c#
 	class Program
    {
        static void Main(string[] args)
        {
            var config = new ConfigurationBuilder()
                 .AddJsonFile("appsettings.json")
                 .Build();
            var services = new ServiceCollection()
                .AddSingleton<IConfiguration>(config)
                .AddOptions()
                .AddSingleton<IConfigureOptions<FxOptions>, FxOptionsSetup>()
                .AddScoped<IDbProvider, DbProvider>()
                .AddSingleton<IUnitOfWorkFactory, UnitOfWorkFactory>()
                .AddSingleton<IRepositoryFactory, RepositoryFactory>()
                .AddSingleton<IDbContextOptionsBuilderUser, SqlServerDbContextOptionsBuilderUser>()
                .AddSingleton<DbContextOptionsBuilderOptions>(new DbContextOptionsBuilderOptions(new DbContextOptionsBuilder<TestDbContext>(), null, typeof(TestDbContext)));

            var serviceProvider = services.BuildServiceProvider();

            var dbProvider = serviceProvider.GetRequiredService<IDbProvider>();
            var uow = dbProvider.GetUnitOfWork<TestDbContext>("TestDb"); // 访问主库

            var repoDbTest = uow.GetRepository<DbTest, int>();
            var obj = new DbTest { Name = "123", Date = DateTime.Now.Date };
            repoDbTest.Insert(obj);
            uow.SaveChanges();
            
          	Console.ReadKey();
          	
            var uow2 = dbProvider.GetUnitOfWork<TestDbContext>("TestDb_Read");

             var uow2 = dbProvider.GetUnitOfWork<TestDbContext>("TestDb_Read"); // 访问从库
            var repoDbTest2 = uow2.GetReadOnlyRepository<DbTest, int>();
            var data2 = repoDbTest2.GetFirstOrDefault();
            Console.WriteLine($"id: {data2.Id} name: {data2.Name}");
            Console.ReadKey();
        }
    }
```
这里直接用控制台来做一个例子,中间多了一个Console.ReadKey()是因为我本地没有配置主从模式,所以实际上我是先插入数据,然后复制到另一个数据库里,再进行读取的.

#### 总结
本文给出的解决方案适用于系统中存在多个不同的上下文,能够适应复杂的业务场景.但对已有代码的侵入性比较大,不知道有没有更好的方案,欢迎一起探讨.