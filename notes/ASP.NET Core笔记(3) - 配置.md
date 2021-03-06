
- 配置规则
- 命令行配置提供程序
- 环境变量配置提供程序
- 文件配置提供程序
- 配置的读取
- 自定义配置数据源

ASP.NET Core中的配置项可以通过命令行、环境变量、json/xml/ini配置文件来提供。

Web应用在生成主机时会调用CreateDefaultBuilder方法，这个方法按照下面的顺序添加各种配置提供程序：
- 文件配置提供程序，用于加载appsetting.json
- 文件配置提供程序，用于加载appsetting.{Environment}.json
- 环境变量配置提供程序
- 命令行配置提供程序

对于同一配置，后面加载的配置提供程序会覆盖前面加载的内容，所以配置的默认优先级从低到高依次为：文件 -> 环境变量 -> 命令行。

### 配置规则
#### 分层配置数据
配置 API 能够通过在配置键中使用分隔符来展平分层数据以保持分层配置数据。
比如，这样的json文件：
```
{
  "section0": {
    "key0": "value",
    "key1": "value"
  },
  "section1": {
    "key0": "value",
    "key1": "value"
  }
}
```
配置提供程序加载时，将使用冒号 (:) 展平节和键，以这样的方式创建唯一键以保持配置源的原始分层数据结构。
```
section0:key0
section0:key1
section1:key0
section1:key1
```


#### 关于键和值的约定
配置键采用以下约定：
- 不区分大小写；
- 如果由相同或不同的配置提供程序设置相同键的值，则键上设置的最后一个值就是所使用的值；
- 分层键默认使用冒号分隔符，但在环境变量中，冒号分隔符可能无法适用于所有平台，可以用双下划线代替。

配置值的约定：
- 值是字符串；
- NULL值不能存储在配置中或绑定到对象。


### 命令行配置提供程序
可以通过ConfigureHostConfiguration和ConfigureAppConfiguration方法来指定配置提供程序或者设置提供程序的优先级。
如果不需要CreateDefaultBuilder默认添加的配置提供提供程序，可以直接删除
```
.ConfigureAppConfiguration((hostingContext, config) =>
{
    config.Sources.Clear();
})
```

在ConfigurationBuilder 的实例上调用 AddCommandLine 扩展方法可以激活命令行配置，不过CreateDefaultBuilder已经自动调用了AddCommandLine，并且命令行配置的优先级最高。若要添加其他配置提供程序并保持能够用命令行参数替代这些提供程序的配置，可以在ConfigureAppConfiguration中添加完其他提供程序后，再调用一次AddCommandLine。
```
.ConfigureAppConfiguration((hostingContext, config) =>
{
    // Call other providers here
    config.AddCommandLine(args);
})
```

命令行配置的设置方式为：
```
dotnet run CommandLineKey=CommandLineValue
```
这种设置方法使用了等号（=），也可以用--或/作为前缀，并空格代替等号，下面的写法都是合理的：
```
dotnet run --CommandLineKey2=value2
dotnet run --CommandLineKey2 value2
dotnet run /CommandLineKey3=value3
dotnet run /CommandLineKey3 value3
```

### 环境变量配置提供程序
在ConfigurationBuilder 的实例上调用 AddEnvironmentVariables 扩展方法可以激活环境变量配置，CreateDefaultBuilder也已经默认启用了环境变量配置提供程序。

AddEnvironmentVariables方法还有一个重载支持加载指定前缀的环境变量。
比如
```
.AddEnvironmentVariables("CUSTOM_")
```
会只加载前缀为CUSTOM_的环境变量。

环境变量的设置方法：
- 命令行，set CUSTOM_ENV1=value1，但这种设置只在当前命令行窗口生效
- 设置全局系统变量；
- 通过visual studio，在launchSettings.json文件中设置。

### 文件配置提供程序
以Json文件为例（INI、XML文件类似）：
```
.ConfigureAppConfiguration((hostingContext, config) =>
{
    config.AddJsonFile("MyConfig.json", 
            optional: true, 
            reloadOnChange: true);
          .AddJsonFile($"MyConfig.{env.EnvironmentName}.json", 
            optional: true, 
            reloadOnChange: true);
})
```
以这种顺序添加，后面MyConfig.{env.EnvironmentName}.json中的配置会覆盖MyConfig.json中的同名配置。

### 配置的读取
#### GetValue<T> 
从配置中提取一个具有指定键的值，并将它转换为指定的类型
```
var number = Configuration.GetValue<int>("NumberKey", 99);
```

#### GetSection
会返回具有指定子节键的配置子节
```
section1 = configuration.GetSection("section1");
```
对于GetSection的结果，可以使用GetValue<T>来获取单个键的值，也可以使用GetChild来获取获取子节键下的所有配置。

#### 使用Get<T>可以直接把配置内容绑定到类
```
var starship = _config.GetSection("starship").Get<Starship>();
```
```
{
  "starship": {
    "name": "USS Enterprise",
    "registry": "NCC-1701",
    "class": "Constitution",
    "length": 304.8,
    "commissioned": false,
    "Array": [ "12", "23", "d3" ],
    "ShipLog": {
      "ID": "1"
    }
  }
  ...
}
```
```
public class Starship 
{
	public string Name { get; set; }
	public string Registry { get; set; }
	public string Class { get; set; }
	public decimal Length { get; set; }
	public bool Commissioned { get; set; }
	public ShipLog ShipLog { get; set; }
	public string[] Array { get; set; }
}
public class ShipLog
{
	public string ID { get; set; }
}
```

### 自定义配置数据源
除了上述配置提供程序，还可以自定义配置数据源，比如可以统一从配置中心获取配置。
要自定义配置源，需要实现IConfigurationSource和IConfigurationProvider。

实现IConfigurationSource：
```
class MyConfigurationSource : IConfigurationSource
{
    public IConfigurationProvider Build(IConfigurationBuilder builder)
    {
        return new MyConfigurationProvider();
    }
}
```
实现IConfigurationProvider，具体方式可以通过继承ConfigurationProvider，然后重写它的虚方法。下面的示例重写了Load方法，在这里可以自行编码获取配置、设置到this.Data，Data是一个键和值类型都是string的字典，用于存储配置。还用了一个定时器来模拟配置的变更，在发生配置变更时，触发base.OnReload();
```
class MyConfigurationProvider : ConfigurationProvider
{
    Timer timer;

    public MyConfigurationProvider() : base()
    {
        timer = new Timer();
        timer.Elapsed += (a, b) => Load(true);
        timer.Interval = 3000;
        timer.Start();
    }

    public override void Load()
    {
        //加载数据
        Load(false);
    }

    void Load(bool reload)
    {
        this.Data["lastTime"] = DateTime.Now.ToString();            
        if (reload)
        {
            base.OnReload();
        }
    }
}
```
使用的时候：*builder.Add(new MyConfigurationSource());*

但这种方法会暴露具体的实现，可以封装为扩展方法。如果把这个扩展类的命名空间指定为Microsoft.Extensions.Configuration，这样在分发到第三方后，就可以直接使用，而不必using程序集的命名空间，而前面的实现类默认为internal，第三方无法访问到，实现了整个类库只暴露出一个扩展方法的效果。
```
namespace Microsoft.Extensions.Configuration
{
    public static class MyConfigurationBuilderExtensions
    {
        public static IConfigurationBuilder AddMyConfiguration(this IConfigurationBuilder builder)
        {
            builder.Add(new MyConfigurationSource());
            return builder;
        }
    }
}
```
监听配置变更的方法：
```
builder.AddMyConfiguration();
var configRoot = builder.Build();
ChangeToken.OnChange(() => configRoot.GetReloadToken(), () =>
{
    //监听到配置变更后的处理代码
});
```





