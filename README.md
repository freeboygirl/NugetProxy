# Nuget代理
基于缓存的代理Nuget API V2程序。  
使用Asp.Net Core RC2

>广西电信百兆：  
>http://nuget.lzzy.net/  
>http://nuget.lzzy.net/api/v2

【AspNet Core】Nuget代理网站

因为访问Nuget太慢，在Dotnet Core RC2发布前，我就基于Asp.Net做了一个Nuget代理网站

这是网站地址：http://nuget.lzzy.net/

Nuget源：http://nuget.lzzy.net/api/v2

广西电信百兆带宽。

这个网站将会缓存所有访问过的API页面与包。

API页面缓存的原理，第一次访问会等待服务器从Nuget上下载页面信息

下载后会替换里面的网址并保存到数据库。

第二次访问会从数据库里取出页面兵判断过期时间

如果已过期，先返回页面信息，后台新建线程下载新页面。

这样国内访问的时候就会觉得非常快。

 

升级至AspNet Core
等到RC2发布后，我觉得，是时候研究Dotnet Core平台了

那么就从这个小小的Nuget代理网站开始吧。

在.Net FX下，开发的代理程序是通过IHttpHandler进行代理的

那么到了AspNet Core，就变成了通过中间件进行代理。

路由在AspNetCore下也变得不同了，使用起来比老版的更方便更简单。

 

路由的使用
注：路由的使用需要用到包Microsoft.AspNetCore.Routing

首先创建一个静态类，添加一个UseXXXMiddleware扩展方法，这是AspNetCore使用中间件的约定，当然你也可以起其它名字。

在方法里，我们需要new一个RouteBuilder实例。

在实例里调用各种映射方法。

然后调用RouteBuilder的Build方法，这将生成一个路由。

最后再使用路由中间件的扩展方法UseRouter，把刚刚Build出来的路由作为参数传递进去即可。

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
public static IApplicationBuilder UseNugetProxyMiddleware(this IApplicationBuilder builder, IConfigurationRoot config)
{
    if (!Directory.Exists("Packages"))
        Directory.CreateDirectory("Packages");
     
    var cacheSection = config.GetSection("Cache");
    _CacheMetadata = int.Parse(cacheSection.GetSection("Metadata").Value);
    _CacheList = int.Parse(cacheSection.GetSection("List").Value);
    _CacheDetail = int.Parse(cacheSection.GetSection("Detail").Value);
    _CacheDefault = int.Parse(cacheSection.GetSection("Default").Value);
    var proxySection = config.GetSection("Proxy");
    _Source = proxySection.GetSection("Source").Value;
    _Replace = proxySection.GetSection("Replace").Value;
    _Path = proxySection.GetSection("Path").Value;
 
    var routeBuilder = new RouteBuilder(builder);
    routeBuilder.MapGet(_Path + "/{action}", PageHandler);
    routeBuilder.MapGet(_Path, PageHandler);
    routeBuilder.MapGet(_Path + "/package/{id}/{version}", PackageHandler);
 
    _Path = "/" + _Path;
 
    var router = routeBuilder.Build();
    return builder.UseRouter(router);
}
 
private static async Task PageHandler(HttpContext httpContext)
{
 
}
 

路由下的中间件使用
使用RouteBuilder的映射方法时，需要传递处理HttpContext的委托

这里其实与AspNet Core本身的中间件方法其实是一样的。

只不过，我们需要在方法里获取各种数据，而不是像AspNet Core中间件一样可以在构造函数里获取到一些服务。

AspNet Core路由提供了HttpContext获取路由参数的扩展方法GetRouteValue。

1
2
3
4
5
6
private static async Task PackageHandler(HttpContext httpContext)
{
 
    string id = httpContext.GetRouteValue("id") as string;
    string version = httpContext.GetRouteValue("version") as string;
}
也有GetRouteData扩展方法，这里我们并不需要用到。

 

零零散散
RC2下，EntityFramework Core的使用出现了一些变化

在Startup.cs里，配置EF Core

1
2
3
4
5
6
7
8
public void ConfigureServices(IServiceCollection services)
{
    // Add framework services.
    services.AddDbContext<DataContext>(builder =>
    {
        builder.UseSqlServer(Configuration.GetConnectionString("DataContext"));
    });
}
在这里修改DbContext的配置。

要注意的是，继承DbContext的类，要使用带参数的构造函数并调用基类构造函数。

1
2
3
4
5
6
public class DataContext : DbContext
{
    public DataContext(DbContextOptions<DataContext> options) : base(options) { }
 
    public DbSet<Page> Page { get; set; }
}
 

剩下的也没什么特别值得要说的了，一些尽在源码里。

 

Github
这里是这个代理网站的Github：

https://github.com/Kation/NugetProxy

 

啊，对了，还有一点。

代理网站的配置。

配置文件：

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
{
  "Logging": {
    "IncludeScopes": false,
    "LogLevel": {
      "Default": "Debug",
      "System": "Information",
      "Microsoft": "Information"
    }
  },
  "ConnectionStrings": {
    "DataContext": "server=(local);database=nuget;uid=sa;pwd=123@abc"
  },
  "Cache": {
    "Metadata": "10080",
    "List": "60",
    "Detail": "1440",
    "Default": "30"
  },
  "Proxy": {
    "Source": "https://www.nuget.org/api/v2",
    "Replace": "https://www.nuget.org/api/v2",
    "Path": "api/v2"
  }
}
可以设定缓存时间，超出时间后才去下载新页面，单位是分钟。

Proxy的Source是Nuget源

Replace是要替换的字符串，将会替换为当前网站地址+Path

PS：我在美国架了一台20M的VPS反向代理Nuget

http://nuget.lzzy.net/api/v2使用的是我的VPS方向代理的源

效果比直接访问Nuget快……
