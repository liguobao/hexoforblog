---
layout: post
title: ASP.NET Core 启动方式（Hosting）
category: .net core
date: 2016-07-28 00:00:00
---

之前版本的ASP.NET程序必须依赖IIS来启动，而IIS上会为挂载在其中的ASP.NET 注册一个ISAPI filter。每当http请求过来时，IIS则会启动w3wp的worker process来开始整个ASP.NET runtime程序。相信大家都这样的流程都有相应的了解。在.net core之前，ASP.NET的主要场景都是运行在Windows平台的，IIS也就是web server的首选了。虽然也有类似于jexus的linux webserver可用，但是基于mono的.NET 总体还是不够Microsoft 原生的强。

不过到了现在，一切都不同了。

###### 新版ASP.NET Core有了.NET Core的支援后已经开始了它的跨平台之旅了，因此ASP.NET Core的启动方式也得开始重新设计以适应新需求了。


### 1、Kestrel 和 IIS platform handler

在 ASP.NET Core 中，整个runtime都是重写过的，所以它和IIS之间的关系也有所改变。而ASP.NET Core为了跨平台，它现在的执行方式就如一般的Console app一样。ASP.NET Core自带一个高性能的I/O组件 - Kestrel，使得它可以不依赖IIS的存在便启动了runtime。不过Kestrel 也只是一个I/O组件，并没有想IIS提供其它的功能来保护和管理ASP.NET 应用程序。ASP.NET Core同样可以通过IIS进行处理。但是如果通过IIS来进行处理的话，这个时候我们便需要一个“中间人”的角色来负责这个功能了。这个“中间人”的名字叫 Http Platform Handler，主要表现在web.config文档中的设置，其中包括启动ASP.NET Core 程序的的路径和名称，需要传入的参数以及一些其他的设置选项。Http Platform Handler的具体设置例子如下：
```csharp
<system.webServer>
    <handlers>
      <add name="httpPlatformHandler" path="*" verb="*"
      modules="httpPlatformHandler" resourceType="Unspecified"/>
    </handlers>
    <httpPlatform processPath="WebApp.exe" arguments="" 
    stdoutLogEnabled="false" startupTimeLimit="3600"/>
  </system.webServer>

```
关于Http Platform Handler的相关资料可以看这个链接：
[http://www.iis.net/downloads/microsoft/httpplatformhandler](http://www.iis.net/downloads/microsoft/httpplatformhandler)


从上面的例子可以看出来，ASP.NET Core编译之后便是一个EXE程序，使得你可以直接运行。因此，当HTTP请求进来时，IIS先接受请求，然后根据你设置的web.config的内容将请求转发给WebApp.exe（你的ASP.NET Core程序），然后WebApp.exe开始执行时便会启动Kestrel，接着这个HTTP请求便进入了ASP.NET Core runtime的世界。这样看来，IIS这时候只是一个简单的proxy/forwarder角色。

PS：在我翻译整理这个文章的时候，世界已经发生了变化：下个版本的asp.net core 将会有全新的IIS module来取代Http Platform Handler。

具体详细资料见：[https://github.com/aspnet/IISIntegration/issues/105](https://github.com/aspnet/IISIntegration/issues/105)



### 2、Main()
ASP.NET Core和其他的.NET 程序一样拥有一个static void Main(),这是整个runtime的进入点。下面看一个样例：
```csharp
 public static void Main(string[] args)
        {
            var host = new WebHostBuilder()
                .UseServer("Microsoft.AspNetCore.Server.Kestrel")
                .UseContentRoot(Directory.GetCurrentDirectory())
                .UseDefaultConfiguration(args)
                .UseIISPlatformHandlerUrl()
                .UseStartup<Startup>()
                .Build();

            host.Run();
        }
```

ASP.NET Core host engine 的建立者是 WebHostBuilder．

它实质上是一个了 IWebHostBuilder interface，其中 UseServer() 是用于指定使用什么样的 server。

其中 UseServer() 有一个扩展方法UseServer(string assemblyName)。这样的话我们可以直接传入Kestrel的程序集名称:"Microsoft.AspNetCore.Server.Kestrel"。当然，这只是其中一种选项。你也可以自己实现一个自己的 server，只要你的 server 实现了 IServerFactory interface 即可。这样的设计提供了一个很大的弹性空间让我们自行选择hosting server（托管服务）。

##### UseContentRoot() 
- 这个扩展方法是让我们指定应用程序的工作目录（working directory），如果我们没有指定的话，则会默认为我们的应用（webapp.exe）所在目录为工作目录。

##### UseDefaultConfiguration() 
- 这个扩展方法使得我们在IWebHostBuilder 建立可以传入一些参数，比如 application key, environment name, server factory location, content root path 等等．因此，当我们在运行 WebApp.exe 的时候，同时可以带入我们需要用到的hosting参数（PS：这样的做法就像运行命令行程序时带入参数，多好玩）。这些参数也可以写在appsettings.json里面通过Configuration来读取。
所以，UseDefaultConfiguration() 也不一定非要存在于 Main() 之中。（？？？个人不是很理解）

PS：原作者原话，"如果我沒记错的话，在写这个文章的时候，UseDefaultConfiguration() 已经被改为成了UseDefaultHostingConfiguration()．显然这个名称更能清楚明白．"(？？？个人还没实践)


##### UseIISPlatofmrHandleUrl() 
- 这个 IWebHostBuilder 的 扩展方法比较特殊。如果你要把 ASP.NET Core 放在 IIS 下，这个扩展方法会读取 IIS http platform handler 的 server port 和 application path，用于作为 ASP.NET Core 的启动位置，如 http://localhost:5000/start．如果你沒用 IIS，这个扩展方法对你来说基本是用不上的．

##### UseStartup<>()

- 这是 WebHostBuilder 里相当重要的一个扩展方法。它的方法签名如下：

```csharp

public static IWebHostBuilder UseStartup<TStartup>
(this IWebHostBuilder hostBuilder) where TStartup : class

```
这里你可以很清楚地看到 <> 里面要放的就是一个 class。

在我们这里的范例中，它的名字是 Startup，里面最重要的就是需要定义要使用那些服务(service)以及要使用那些中间件(middleware)。


### 3、Startup
这是一个非常非常重要的class,在ASP.NET Core范例中一般都把它命名成Startup。其实我们把它命名成其他名字也是可以的，或者设定多个Startup。上面的内容可以看到，UseStartup()指定了谁是starup class。然后在Build()便会实例化starup class，之后便执行里面两个重要的方法：ConfigureServices() 和 Configure()。

我们先來看 Startup 的构造函数.

```csharp
 public Startup(IHostingEnvironment env)
        {
            // Set up configuration sources.
            var builder = new ConfigurationBuilder()
                .AddJsonFile("appsettings.json")
                .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
                .AddEnvironmentVariables();
            Configuration = builder.Build().ReloadOnChanged("appsettings.json");
        }
```

Host engine 在被执行 Build()时已经知道 startup type 是 Startup class，所以在 Build() 的时候会先创建期示例。在我们这里的是调用Startup类带参数的构造函数。

在我们这个例子中选择的是传入IHostingEnvironment示例，它为我们带来了环境变量（EnvironmentName）。

我们这里主要目的是把Configuration实例化。这是一个蛮重要的基础组件，以后会有文章来说明它。

在这里我们特别说明一下，上面的示例代码中，我们执行了两次 AddJsonFile()，而且第二个json file的参数和第一个的还不太一样。这样的目的是为了让开发者可以把开发环境使用的环境参数和其他环境使用的参数有所区别。比如，你使用的开发环境用的是appsettings.json，这个文件只存在于你的电脑中。另一个文档是appsettings.production.json，这是正式环境使用的参数设定文档。第二個 AddJsonFile() 第二個参数是 true，也就是可能不存在的意思。所以若遇到重复名称参数时，appsettings.production.json 会覆盖 appsettings.json 的內容。这样使得开发环境和生产环境得以区分。

接下来，在 IWebHostBuilder 的 Build()里面会执行 host engine 初始化的程序，其中就会去找Startup class里面的两个方法： ConfigureServices() 和 Configure()。

 ConfigureSerivces() 是定义了这个 web application 要使用那些服务，然后将这些服务放在 service container (IServiceCollection) 裡面。如下面的样例：
 
 ```csharp
 
   public void ConfigureServices(IServiceCollection services)
        {
            // add entity framework
            services.AddEntityFramework()
                    .AddDbContext<BlogsContext>(o => o.UseSqlServer(Configuration["Data1:DefaultConnection:ConnectionString"]))
             
            // Add framework services.
            services.AddMvc();
        }

```

它定义了entity framework和mvc两个服务。这里所谓的服务（services）的意思也就是通过它们带入更庞大的程序代码。这听起来好像有点搞笑，但也真的如此。像Entity framework 里面有这么多的代码，一定都需要带入许多定义好的物件或者参数，而不只是一个程序的进去点而已，所以services 的目的就是在这里。


Configure() 主要是定义了中间件（middleware）以及它们的顺序．

```csharp
        public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
        {
            loggerFactory.AddConsole(Configuration.GetSection("Logging"));
            loggerFactory.AddDebug();

            app.UseStaticFiles();

            app.UseMvcWithDefaultRoute();
        }

```


### 4、Build 和 Run
最后，在IHostWebBuilder里最后的两个动作便是：Build and Run. 


##### Build() 
- 这个方法做的工作便是建立 hosting service，把 Startup 中定义的的 services 和 middleware 接收过来，然后确定content root path 和 application name，接着一句前面这些资料再加上Configuration过来的数据来初始化host engine (WebHost.cs)．

##### Run() 
- 这个是启动 host engine 的 扩展方法，它在启动之前加入了一个 CancelKeyPress 的事件．因为在 Run() 方法 中传入入了 CancellationTokenSource() ，让我们有一个方法可以随时中断host engine的执行。

目前的做法就用是 CancelKeyPress 事件，所以你可以按下 Ctrl+C  來中止 host engine 的执行．

比较特別的是，这一段中止的文字说明居然是用 hard code，參考如下:

host.Run(cts.Token, "Application started. Press Ctrl+C to shut down.");

不过这样的话，这里你也不能写中文...

本文整理于[https://dotblogs.com.tw/aspnetshare/2016/03/28/20160327](https://dotblogs.com.tw/aspnetshare/2016/03/28/20160327)并已征得作者同意。
感谢Bruce的分享。
 