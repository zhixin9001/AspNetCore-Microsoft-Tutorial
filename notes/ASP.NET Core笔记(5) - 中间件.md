
- 中间件管道模型
- 中间件的配置
- 自定义中间件


中间件是一类装配在应用管道的代码，负责处理请求和响应。每个中间件都可在管道中的下一个组件前后执行工作，并选择是否将请求传递到管道中的下一个中间件。在Startup.Configure方法中可以进行中间件的装配。

### 中间件管道模型
中间件管道模型如下图所示：
![中间件管道模型](https://docs.microsoft.com/zh-cn/aspnet/core/fundamentals/middleware/index/_static/request-delegate-pipeline.png?view=aspnetcore-3.1)

ASP.NET Core请求管道包含一系列请求委托，沿黑色箭头依次被调用执行，每个委托均可在下一个委托前后执行操作。这种模型也被形象地称为“俄罗斯套娃”。

一个中间件可以是匿名方法的显示嵌入到管道中，也可以封装为单独的类便于重用，嵌入式的中间件就像这样：
```
public void Configure(IApplicationBuilder app)
{
	app.Run(async context =>
	{
		await context.Response.WriteAsync("Hello, World!");
	});
}
```

### 中间件的配置
配置中间件会使用到三个扩展方法：
- Use
- Run
- Map

#### Use
Use用来将多个中间件按添加顺序链接到一起：
```
app.Use(async (context, next) =>
{
	await context.Response.WriteAsync("middleware1 begin\r\n");
	await next.Invoke();
	await context.Response.WriteAsync("middleware1 end\r\n");
});

app.Use(async (context, next) =>
{
	await context.Response.WriteAsync("middleware2 begin\r\n");
	await next.Invoke();
	await context.Response.WriteAsync("middleware2 end\r\n");
});

app.Run(async context =>
{
	await context.Response.WriteAsync("end of pipeline.\r\n");
});
```
这三个中间件配置到管道后，输出的结果与管道模型的图示是一致的：
```
middleware1 begin
middleware2 begin
end of pipeline.
middleware2 end
middleware1 end
```

可以看到除了最后一个中间件，前面的中间件都调用了*await next.Invoke()*，next参数表示管道中的下一个委托，如果不调用 next，后面的中间件就不知执行，这称为管道的短路。
通常中间件都应该自觉调用下一个中间件，但有的中间件会故意造成短路，比如授权中间件、静态文件中间件等。

#### Run
Run的委托中没有next参数，这就意味着它会称为最后一个中间件（终端中间件），此外可以故意短路的Use委托也可能会成为终端中间件。

#### Map，提供了创建管道分支的能力
Map扩展用作约定来创建管道分支，会基于给定请求路径的匹配项来创建请求管道分支，如果请求路径以给定路径开头，则执行分支。
```
private static void HandleMapTest1(IApplicationBuilder app)
{
	app.Run(async context =>
	{
		await context.Response.WriteAsync("Map Test 1");
	});
}

private static void HandleMapTest2(IApplicationBuilder app)
{
	app.Run(async context =>
	{
		await context.Response.WriteAsync("Map Test 2");
	});
}

public void Configure(IApplicationBuilder app)
{
	app.Map("/map1", HandleMapTest1);

	app.Map("/map2", HandleMapTest2);

	app.Run(async context =>
	{
		await context.Response.WriteAsync("Hello from non-Map delegate. <p>");
	});
}
```
这段代码为管道创建了三个分支：
URL | Response
-|-
localhost:1234	|Hello from non-Map delegate.
localhost:1234/map1	|Map Test 1
localhost:1234/map2	|Map Test 2

##### Map还支持嵌套
```
app.Map("/level1", level1App => {
    level1App.Map("/level2a", level2AApp => {
        // "/level1/level2a" processing
    });
    level1App.Map("/level2b", level2BApp => {
        // "/level1/level2b" processing
    });
});
```

##### MapWhen，满足条件时创建分支
```
app.MapWhen(context => context.Request.Query.ContainsKey("branch"),
                               branch => {
    });
```

此外，UseWhen也可以根据条件创建分支，它与MapWhen的区别在于，使用UseWhen创建的分支如果不包含终端中间件，则会重新汇入主管道。

### 自定义中间件
通常使用内置的中间件可满足大多数场合，但如果有需要也可以创建自定义中间件。
假设有这样一个嵌入式中间件，可以通过查询字符串设置当前请求的区域，比如https://localhost:5000/?culture=zh-cn，会将CurrentCulture设置为Chinese Simplified。现在要将其封装为可重用的独立的中间件。
```
public void Configure(IApplicationBuilder app)
{
	app.Use(async (context, next) =>
	{
		var cultureQuery = context.Request.Query["culture"];
		if (!string.IsNullOrWhiteSpace(cultureQuery))
		{
			var culture = new CultureInfo(cultureQuery);

			CultureInfo.CurrentCulture = culture;
			CultureInfo.CurrentUICulture = culture;
		}

		// Call the next delegate/middleware in the pipeline
		await next();
	});

	app.Run(async (context) =>
	{
		await context.Response.WriteAsync(
			$"Hello {CultureInfo.CurrentCulture.DisplayName}");
	});

}
```

在开始封装前，先了解一下对中间件的要求：
- 中间件必须具有类型为RequestDelegate的参数的公共构造函数，之前代码中看到next.Invoke()，其中next就是下一个中间件的RequestDelegate;
- 名为 Invoke 或 InvokeAsync 的公共方法。而且这个方法的返回类型为Task，且第一个参数的必须是HttpContext，其它的参数由DI容器解析。

基于上述要求，编写的中间件为：
```
public class RequestCultureMiddleware
{
    private readonly RequestDelegate _next;

    public RequestCultureMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var cultureQuery = context.Request.Query["culture"];
        if (!string.IsNullOrWhiteSpace(cultureQuery))
        {
            var culture = new CultureInfo(cultureQuery);

            CultureInfo.CurrentCulture = culture;
            CultureInfo.CurrentUICulture = culture;
        }

        // Call the next delegate/middleware in the pipeline
        //await _next.Invoke(context);
        await _next(context);
    }
}
```

然后就可以使用了：
```
app.UseMiddleware<RequestCultureMiddleware>();
```

还可以进一步封装为IApplicationBuilder的扩展方法：
```
public static class RequestCultureMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestCulture(
        this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<RequestCultureMiddleware>();
    }
}
```
然后就可以像内置的中间件一样了：
```
app.UseRequestCulture();
```



	
	