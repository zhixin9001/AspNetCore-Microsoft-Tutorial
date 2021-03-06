- 服务的生命周期
- 链式注入时，生存期的选择
- TryAdd与泛型注入
- 替换内置服务容器

ASP.NET Core提供了默认的依赖注入容器，可以在Startup.ConfigureServices方法中进行服务注入的配置。

### 服务的生命周期
默认的依赖注入容器提供了三种生命周期：
- 暂时（AddTransient），每次在向服务容器进行请求时都会创建新的实例，这种生存期适合轻量级、 无状态的服务。
- 范围内（AddScoped），为每个客户端请求创建一次实例。
- 单例（AddSingleton），第一次请求时（或者在运行Startup.ConfigureServices 并且使用服务注册指定实例时）创建，每个后续请求都使用相同的实例。如果应用需要单例行为，建议允许服务容器管理服务的生存期。 不要再自己实现单例设计模式。

下面试验一下这三种方式的差异：
注入配置：
```
services.AddSingleton<ISingletonTest, SingletonTest>();
services.AddTransient<ITransientTest, TransientTest>();
services.AddScoped<IScopedTest, ScopedTest>();

services.AddTransient<ScopeAndTransientTester>();
```

控制器：
```
public DefaultController(ISingletonTest singleton, ITransientTest transient, IScopedTest scoped, ScopeAndTransientTester scopeAndTransientTester
    )
{
    this.singleton = singleton;
    this.transient = transient;
    this.scoped = scoped;
    this.scopeAndTransientTester = scopeAndTransientTester;
}

[HttpGet]
public string Get()
{
    return $"singleton={singleton.guid}; \r\nscoped1={scoped.guid};\r\nscoped2={scopeAndTransientTester.ScopedID()};\r\ntransient={transient.guid};\r\ntransient2={scopeAndTransientTester.TransientID()};";
}
```
用于第二次注入的ScopeAndTransientTester类：
```
public class ScopeAndTransientTester
{
    public ISingletonTest singleton { get; }
    public ITransientTest transient { get; }
    public IScopedTest scoped { get; }

    public ScopeAndTransientTester(ISingletonTest singleton, ITransientTest transient, IScopedTest scoped)
    {
        this.singleton = singleton;
        this.transient = transient;
        this.scoped = scoped;
    }

    public Guid SingletonID()
    {
        return singleton.guid;
    }
    public Guid TransientID()
    {
        return transient.guid;
    }
    public Guid ScopedID()
    {
        return scoped.guid;
    }
}
```
第一次请求：
```
singleton=ebece97f-bd38-431c-9fa0-d8af0419dcff; 
scoped1=426eb574-8f34-4bd3-80b3-c62366fd4c74;
scoped2=426eb574-8f34-4bd3-80b3-c62366fd4c74;
transient=98f0da06-ba8e-4254-8812-efc19931edaa;
transient2=c19482f7-1eec-4b97-8cb2-2f66937854c4;
```
第二次请求：
```
singleton=ebece97f-bd38-431c-9fa0-d8af0419dcff; 
scoped1=f5397c05-a418-4f92-8c6d-78c2c8359bb5;
scoped2=f5397c05-a418-4f92-8c6d-78c2c8359bb5;
transient=59ed30fa-609b-46b1-8499-93a95ecd330b;
transient2=d2a8ea1c-ae0b-4732-b0a1-ca186897e676;
```

用Guid来表示不同的实例。对比两次请求可见AddSingleton方式的id值相同；AddScope方式两次请求之间不同，但同一请求内是相同的；AddTransient方式在同一请求内的多次注入间都不相同。

### TryAdd与泛型注入
另外还有TryAdd{Lifetime}方式，如果只希望在同类型的服务尚未注册时才添加服务，可以使用这种方法。如果直接使用Add{Liffetime}，则多次使用会重复注册。

除了Add{LIFETIME}<{SERVICE}, {IMPLEMENTATION}>()这种用法，还可以这另一个重载写法：
```
Add{LIFETIME}(typeof(SERVICE, typeof(IMPLEMENTATION)
```
这种写法还有个好处是可以解析泛型，像ILogger就是框架利用这种方式自动注册的：
```
services.AddSingleton(typeof(ILogger<>), typeof(Logger<>));
```
### 链式注入时，生存期的选择
以链式方式使用依赖关系注入时，每个请求的依赖关系相应地请求其自己的依赖关系。容器会解析这些依赖关系，构建“依赖关系树”，并返回完全解析的服务。在链式注入时需要注意依赖方的生命周期不能大于被依赖方的生命周期。
前面例子中的ScopeAndTransientTester把三种生命周期的服务都注入了，那么它就只能注册为暂时的，否则启动时会报错：
```
System.AggregateException: 'Some services are not able to be constructed (Error while validating the service descriptor '*** Lifetime: Singleton ImplementationType: ***': Cannot consume scoped service '***' from singleton 
```

### 替换内置服务容器
内置的服务容器旨在满足框架和大多数开发者应用的需求，一般使用内置容器就已足够，除非需要使用某些不受内置容器支持的特定功能如属性注入、基于名称的注入、子容器、自定义生存期管理、对迟缓初始化的 Func<T> 支持、基于约定的注册等。
