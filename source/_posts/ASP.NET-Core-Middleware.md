---
layout: post
title: ASP.NET Core Middleware
category: .net
---

在ASP.NET 时代，一般来说我们很少会用到HttpModule/HttpHandler，然而有些场景我们使用HttpModule/HttpHandler倒方便快捷完成我们的需求。有兴趣了解HttpModule/HttpHandler以及使用场景的话，可以看下面这个链接的内容。

[选择HttpHandler还是HttpModule？](http://www.cnblogs.com/fish-li/archive/2013/01/04/2844908.html)

来到ASP.NET Core时代，类似功能的内容可能我们看得就要多得多了。因为在ASP.NET Core时代，微软将HttpModule“变更”之后，并为它授予了更灵活应用场景。

##### 这就是这个文章要介绍的主角：Middleware（中间件）。

## Middleware

为了使用跨平台，ASP.NET Core整个架构和代码都重写了一遍，所以 HttpModule 自然也就不存在了。但是相似的功能还是有的，它的名字叫： Middleware。和以前不同，在ASP.NET Core中我们将会经常看到 Middleware的存在，因为现在的每一个服务都是用Middleware的方式呈现在ASP.NET Core 管道中。不仅如此，meddleware比起之前的HttpModule也更弹性易用了。

首先先来看看什么是middleware。

```csharp

public void Configure(IApplicationBuilder app, IHostingEnvironment env, 
                      ILoggerFactory loggerFactory)
{
    loggerFactory.AddConsole(Configuration.GetSection("Logging"));
    loggerFactory.AddDebug();

    app.UseStaticFiles();

    app.UseMvc(routes =>
    {
        routes.MapRoute("default",
        "{controller=Home}/{action=Index}/{id?}");
    });
}

```
看过ASP.NET Core项目的话，相信大家对Satarup.cs并不会陌生。在Starup.cs里面便有一个Configure()函数用于定义项目需要使用哪些middleware。

上面的例子使用了两个middleware，一个是 UseStaticFiles，另一个是 UseMvc。这两个都是core自带的middleware，所以我们可以直接使用。UseStaticFiles 是为HTTP Request提供存取网站的文件，简单理解就是使得网站上的静态文件可访问，而UseMvc就是启用MVC routing机制。有了这两个middleware，我们的的网站就有了MVC routing和读取静态文件的功能。

如果我们把UseMvc去掉，那么MVC routing也就不存在了，我们输入 http://website/[Controller]/[Action] 类似的地址也就无效了。

### 和HttpModule的不同之处

在使用HttpModule的时候，我们是在实现/重写接口，这个时候就要求我们在适当的地方做适当的事情。例如，要做 authorization 的话就最好在 HttpModule 定义好的 Authorization 事件 (AuthorizatRequest) 中完成这个功能。在 ASP.NET life cycle 的文件里我们可以查到 HttpModule  定义了那些事件，每一個事件都有哪些特別的功能。因此我们需要全面了解之后再来选择实现/重写我们需要的事件。而在Middleware中，完全没有这样的限制，也不存在这样的事件，我们可以自行设计实现我们的机制。

## Middleware 流程
[https://docs.asp.net/en/latest/fundamentals/middleware.html](https://docs.asp.net/en/latest/fundamentals/middleware.html) 这个文章中说明了基本的middleware概念。目前asp.net docs里面有不少的内容都是开源社区开发者贡献的

在这个文章里面有一个简单的流程图说明了ASP.NET runtime中middleware的执行过程。

![middleware执行过程](http://7xread.com1.z0.glb.clouddn.com/0b6d43ad-7d95-48ea-bbf3-ea7a93c4a366)

在 middleware 里面一定要定义 Invoke()函数，因为这是让 engine 默认调用 middleware 的Incoke函数。Middleware 里面所需要做什么事情就放在 Invoke() 里面，同时 Invoke() 里面还需要调用下一个 middleware。因此执行内容就如上图所示。Middleware 之间除了必须传送 HttpContext之外，也可以自定义传入其他的参数，这比以前的HttpModule方便多了。


所以当 HTTP request 进来之后，engine 便会呼叫第一个 middleware 的 Invoke()，同时把传入HttpContext，然后第一个 middleware 可以再接着呼叫第二个 middleware 的 Invoke()，同时再把 HttpContext 继续传入，一直到最后一个middleware 的 Invoke() 结束之后，整个 HttpContext 的內容可能在 middleware 里面新增或被改变了，最后再按照整個原先的 call stack 从最后一个 middleware 回到第一个 middleware，再通过  engine 回传到client 端，完成request.


下来通过一个例子我们一起来了解一下Middleware。

## 编写简单的 Middleware 

```csharp
public class SampleMiddleware
{
    private readonly RequestDelegate _next;

    public SampleMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task Invoke(HttpContext context)
    {
        if (string.IsNullOrEmpty(context.User.Identity.Name))
        {
            context.Response.Redirect("/NoName.html");
            return;
        }
        await _next.Invoke(context);
    }
}

```


这一个middleware的名字叫SampleMiddleware。它有一个构造函数以及Invoke函数，而Invoke()只接收一个参数HttpContext。

_next是一个叫 RequestDelegate类型，换言之这就是一个delegate，用于代表下一个middleware是谁。所以在构造函数中要把下一个middleware delegate传入。看到这里或许会觉得奇怪，我们的middleware在执行过程中怎么会知道下一个middleware是谁？这一部分稍后解释。


在 Invoke() 里面，在 await _next.Invoke() 之前都是当前middleware的逻辑代码，从上面流程图来看的话就是由左自右的方向． await _next.Invoke() 之后的代码是就是流程图上由右至右的方向，因此，透過这样简单的设计，开发者就能很明确地控制什么样逻辑要先做或后做了。

在 SampleMiddleware 之中，这里只做了一個很简单的动作，如果 username 是空白的话，就将该连接重定向到到 NoName.html 然后中断 middleware 的执行。

为了能让这个middleware作为 ApplicationBuilder来使用，我们另外需要写一个扩展方法。代码如下：

```csharp

public static partial class MiddlewareExtensions
{
    public static IApplicationBuilder UseSampleMiddleware(
    this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<SampleMiddleware>();
    }
}
```

这给扩展方法建立了UseSampleMiddleware()，使得我们可以让ApplicationBuilder 去读 SampleMiddleware。

这是回到Startup.cs中，在 Configure() 里面我们就可以把 SampleMiddleware 加入到我们的 pipeline中了。具体代码如下：
```csharp

public void Configure(IApplicationBuilder app,IHostingEnvironment env, 
                      ILoggerFactory loggerFactory)
{
    loggerFactory.AddConsole(Configuration.GetSection("Logging"));
    loggerFactory.AddDebug();

    app.UseStaticFiles();

    app.UseSampleMiddleware();   // <-- SampleMiddleware

    app.UseMvc(routes =>
    {
        routes.MapRoute("default",
        "{controller=Home}/{action=Index}/{id?}");
    });
}

```
把 SampleMiddleware 放在 UseStaticFiles 和 UseMvc 之间，也就是说在 http request 还沒进入到 MVC routing 之前，就会先检查 HttpContext 里面是不是有空白的 username。很显然username肯定是空白的，因为我并沒有加入任何使用者验证代码这里面，所以利用 dotnet run 來运行这个项目的时候，你就会看到 Http code 302 出現，它的意思就是 http redirect，也就是 SampleMiddleware 里面面所做的 redirect 发生作用了。

![http redirect](http://7xread.com1.z0.glb.clouddn.com/1d7e4c87-9fff-4401-8933-36dcbf857199)

## Middleware 的执行顺序很重要

前面解释了 middleware 执行过程是一个接着一个的．不同的 middleware 对 HttpContext 的內容都可能有不同的处理或更改，因此执行舒服便格外重要。举个例子，如果将上面 Configure() 的代码变更如下:

```csharp

public void Configure(IApplicationBuilder app, IHostingEnvironment env,
                       ILoggerFactory loggerFactory)
{
    loggerFactory.AddConsole(Configuration.GetSection("Logging"));
    loggerFactory.AddDebug();

    app.UseSampleMiddleware();   // SampleMiddleware

    app.UseStaticFiles();        // StaticFiles

    app.UseMvc(routes =>
    {
        routes.MapRoute("default","{controller=Home}/{action=Index}/{id?}");
    });
}
```
我们把SampleMiddleware 放到StaticFiles 之前。这就导致在 SampleMiddleware 里重定向到 NoName.html会失败。

为什么会失败呢? 因为我们的 ApplicationBuilder 执行到行到 SampleMiddleware 时候重定向到NoName.html，也就是做读取静态页面，而这个功能服务方是在下一个 middleware (StaticFiles) 才会提供的，因此 ApplicationBuilder 无法找到 NoName.html，所以在浏览器上也就看不到 NoName.html 的內容。

#### Middleware 这样的设计带来了很大的方便和弹性，同時我们自己也要小心 middleware 前后相依性的问题。

## Middleware 背后原理

现在 ASP.NET Core 已是开源项目了，所以最后说明一下 middleware 原理的基本概念．整個 ASP.NET fundamental 的部份用了许多的 function delegate , task, denepdency injection 的编写方法，所以要看 source code 之前，建议先对这三个东西先行了解，这样对理解 ASP.NET Core源码很有帮助．

在前面的代码中，我们看到 RequestDelegate,  顾名思义就知道这是一个delegate（委托），它是用来代表 middleware 的 delegate. 它的 source code 在 [RequestDelegate.cs](https://github.com/aspnet/httpabstractions/blob/master/src/Microsoft.AspNet.Http.Abstractions/RequestDelegate.cs)

IApplicationBuilder interface 是一個相当重要的接口，它定义了整個APP要用哪些服务和參數，当然也包含要使用那些 middleware，它的 souce code 在 [IApplicationBuilder.cs](https://github.com/aspnet/httpabstractions/blob/master/src/Microsoft.AspNet.Http.Abstractions/IApplicationBuilder.cs)。

其中你可以看到 Use()，通过 Use() 的实例就可以把 middleware delegate 注册到 host engine 上。

另外一个就是 UseMiddlewareExtensions ，前面的代码曾用了 builder.UseMiddleware<SampleMiddleware>(); 它会检查你写的 middleware 是不是合法的，比如有沒有 Invoke()，是不是只有一个Invoke()，Invoke() 的参数有沒有一个是 HttpContext type，所有的检查都通过之后便建立出该middleware instance 的 delegate。

因此，当你的 ASP.NET Core APP刚启动的时候，在 Startup.cs 的 Configure() 就会把所有的 middleware delegate 建立起來，然后依序地放到內部的 stack 结构中。以上面的范例来说， stack 结构第一个元素是 StaticFiles,  然后是 SampleMiddleware 最后是 Mvc。接着每個 middleware 要被建立时是做 stack pop 的操作，所以 Mvc 的 _next 是 engine 里一些內部的 middleware 处理器，然後 pop 出 SampleMiddleware 时，就把 SampleMiddleware 的 _next 指向前面一個 pop 出來的 Mvc。依照着这样的逻辑一直到最前面的 middleware。所以在 host engine 在 Build() 之前这些动作都会完成，然后 host engine 才能执行Run()。有关 host engine 可參考 
[WebHostBuilder.cs](https://github.com/aspnet/hosting/blob/master/src/Microsoft.AspNet.Hosting/WebHostBuilder.cs)


全文完。


本文整理于[https://dotblogs.com.tw/aspnetshare/2016/03/20/201603191](https://dotblogs.com.tw/aspnetshare/2016/03/20/201603191)并已征得作者同意。
感谢Bruce的分享。