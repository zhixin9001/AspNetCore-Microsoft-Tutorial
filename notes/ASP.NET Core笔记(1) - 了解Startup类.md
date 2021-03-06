- Startup构造函数
- ConfigureServices方法
- Configure方法
- 在ConfigureWebHostDefaults中直接配置服务和请求管道

ASP.NET Core一般使用Startup类来进行应用的配置。在构建应用主机时指定Startup类，通常通过在主机生成器上调用WebHostBuilderExtensions.UseStartup<TStartup> 方法来指定 Startup类：
```
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });

```

Startup类中可以包含以下方法：
- Startup构造函数
- ConfigureServices方法，可选
- Configure方法

### Startup构造函数
在3.1中，使用泛型主机 (IHostBuilder) 时，Startup构造函数中只能注入这三种类型的服务：IWebHostEnvironment、IHostEnvironment、IConfiguration。
尝试注入别的服务时会抛出InvalidOperationException异常。
```
System.InvalidOperationException: 'Unable to resolve service for type '***' while attempting to activate '_1_Startup.Startup'.'
```

因为主机启动时，执行顺序为Startup构造函数 -> ConfigureServices方法 -> Configure 方法。在Startup构造函数执行时主机只提供了这三个服务，别的服务需要在ConfigureServices方法中添加。然后到了Configure方法执行的时候，就可以使用更多的服务类型了。

### ConfigureServices方法
主机会调用ConfigureServices方法，将需要的服务以依赖注入的方式添加到服务容器，使其在Configure方法和整个应用中可用。

ConfigureServices方法的参数中无法注入除IServiceCollection之外的服务。
具体使用时可以通过IServiceCollection的扩展方法为应用配置各种功能。

```
public void ConfigureServices(IServiceCollection services)
{

    services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlServer(
            Configuration.GetConnectionString("DefaultConnection")));
    services.AddDefaultIdentity<IdentityUser>(
        options => options.SignIn.RequireConfirmedAccount = true)
        .AddEntityFrameworkStores<ApplicationDbContext>();

    services.AddRazorPages();
}
```

### Configure方法
Configure 方法用于指定应用响应 HTTP 请求的方式。 可通过将中间件组件添加到 IApplicationBuilder 实例来配置请求管道。 Configure 方法参数中的IApplicationBuilder不需要在服务容器中注册就可使用，它已由主机创建好并直接传递给了Configure方法。
Configure方法由一系列的Use扩展方法组成：
```
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    app.UseRouting();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapGet("/", async context =>
        {
            await context.Response.WriteAsync("Hello World!");
        });
    });
}
```
每个Use扩展都在请求管道中添加了中间件。配置到请求管道中的中间件都会调用它之后的下一个中间件或者直接将管道短路。

在Configure方法参数中，可以根据自己的需要注入像IWebHostEnvironment, ILoggerFactory之类的服务，或者是在ConfigureServices方法中添加到DI容器中的服务。

### 在ConfigureWebHostDefaults中直接配置服务和请求管道
ASP.NET Core还提供了不使用Startup类而能够配置服务和请求管道的方式。也可以在ConfigureWebHostDefaults上调用它提供的ConfigureServices和Configure方法。
```
public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureAppConfiguration((hostingContext, config) =>
            {
            })
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.ConfigureServices((c, services) =>
                {
                    services.AddControllers();
                })
                .Configure(app =>
                {
                    var env = app.ApplicationServices.GetRequiredService<IWebHostEnvironment>();

                    if (env.IsDevelopment())
                    {
                        app.UseDeveloperExceptionPage();
                    }

                    app.UseRouting();

                    app.UseEndpoints(endpoints =>
                    {
                        endpoints.MapGet("/", async context =>
                        {
                            await context.Response.WriteAsync("Hello World!");
                        });
                    });
                });
            })
```



