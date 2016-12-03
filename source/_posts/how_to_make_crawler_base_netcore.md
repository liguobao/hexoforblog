---
layout: post
title: 手把手教你用.NET Core写爬虫
category: asp.net core
date: 2016-12-04 00:00:00
tags:
- asp.net core
- crawler
---

### 写在前面
自从上一个项目[58HouseSearch](https://github.com/liguobao/58HouseSearch)从.NET迁移到.NET core之后，磕磕碰碰磨蹭了一个月才正式上线到新版本。
然后最近又开了个新坑，搞了个[Dy2018Crawler](http://codelover.win/)用来爬dy2018电影天堂上面的电影资源。这里也借机简单介绍一下如何基于.NET Core写一个爬虫。
PS：如有偏错，敬请指明...
PPS:该去电影院还是多去电影院，毕竟美人良时可无价。

### 准备工作（.NET Core准备）
首先，肯定是先安装.NET Core咯。下载及安装教程在这里：[.NET - Powerful Open Source Development](https://www.microsoft.com/net/core)。无论你是Windows、linux还是mac，统统可以玩。

我这里的环境是：Windows10 + VS2015 community updata3 + .NET Core 1.1.0 SDK + .NET Core 1.0.1 tools Preview 2.

理论上，只需要安装一下 .NET Core 1.1.0 SDK 即可开发.NET Core程序，至于用什么工具写代码都无关紧要了。

安装好以上工具之后，在VS2015的新建项目就可以看到.NET Core的模板了。如下图：

![](https://www.microsoft.com/net/images/screenshots/FileNewProject.png)

为了简单起见，我们创建的时候，直接选择VS .NET Core tools自带的模板。

### 一个爬虫的自我修养
#### 分析网页
写爬虫之前，我们首先要先去了解一下即将要爬取的网页数据组成。

具体到网页的话，便是分析我们要抓取的数据在HTML里面是用什么标签抑或有什么样的标记，然后使用这个标记把数据从HTML中提取出来。在我这里的话，用的更多的是HTML标签的ID和CSS属性。

以本文章想要爬取的dy2018.com为例,简单描述一下这个过程。dy2018.com主页如下图：

![](http://7xrayk.com1.z0.glb.clouddn.com/123.png)

在chrome里面，按F12进入开发者模式，接着如下图使用鼠标选择对应页面数据，然后去分析页面HTML组成。

![](http://7xrayk.com1.z0.glb.clouddn.com/chromeF12_Select_HTML.png)


接着我们开始分析页面数据:

![](http://7xrayk.com1.z0.glb.clouddn.com/chrome_dy2018_lstmovie_divclass.png)

![](http://7xrayk.com1.z0.glb.clouddn.com/chrome_dy2018_a.png)

经过简单分析HTML，我们得到以下结论：

1. www.dy2018.com首页的电影数据存储在一个class为co_content222的div标签里面

2. 电影详情链接为a标签，标签显示文本就是电影名称，URL即详情URL

那么总结下来，我们的工作就是：找到class='co_content222' 的div标签，从里面提取所有的a标签数据。

#### 开始写代码...

之前在写[58HouseSearch项目迁移到asp.net core](https://zhuanlan.zhihu.com/p/22764927)简单提过AngleSharp库，一个基于.NET（C#）开发的专门为解析xHTML源码的DLL组件。

1. AngleSharp主页在这里：[https://anglesharp.github.io/](https://anglesharp.github.io/)，

2. 博客园文章：[解析HTML利器AngleSharp介绍](http://www.cnblogs.com/pandait/p/AngleSharp.html)，

3. Nuget地址:[Nuget AngleSharp](https://www.nuget.org/packages/AngleSharp) 安装命令：Install-Package AngleSharp

##### 获取电影列表数据
``` csharp
private static HtmlParser htmlParser = new HtmlParser();

private  ConcurrentDictionary<string, MovieInfo> _cdMovieInfo = new ConcurrentDictionary<string, MovieInfo>();
private void AddToHotMovieList()
{
	//此操作不阻塞当前其他操作，所以使用Task
	// _cdMovieInfo 为线程安全字典，存储了当期所有的电影数据
	Task.Factory.StartNew(()=> 
	{
	    try
	    {
	        //通过URL获取HTML
	        var htmlDoc = HTTPHelper.GetHTMLByURL("http://www.dy2018.com/");
	        //HTML 解析成 IDocument
	        var dom = htmlParser.Parse(htmlDoc);
	        //从dom中提取所有class='co_content222'的div标签
	        //QuerySelectorAll方法接受 选择器语法 
	        var lstDivInfo = dom.QuerySelectorAll("div.co_content222");
	        if (lstDivInfo != null)
	        {
	            //前三个DIV为新电影
	            foreach (var divInfo in lstDivInfo.Take(3))
	            {
	                //获取div中所有的a标签且a标签中含有"/i/"的
	                //Contains("/i/") 条件的过滤是因为在测试中发现这一块div中的a标签有可能是广告链接
	                divInfo.QuerySelectorAll("a").Where(a => 
	                a.GetAttribute("href").Contains("/i/"))
	                .ToList().ForEach(
	                a =>
	                {
	                    //拼接成完整链接
	                    var onlineURL = "http://www.dy2018.com" + a.GetAttribute("href");
	                    //看一下是否已经存在于现有数据中
	                    if (!_cdMovieInfo.ContainsKey(onlineURL))
	                    {
	                        //获取电影的详细信息
	                        MovieInfo movieInfo = FillMovieInfoFormWeb(a, onlineURL);
	                        //下载链接不为空才添加到现有数据
	                        if (movieInfo.XunLeiDownLoadURLList != null 
	                        && movieInfo.XunLeiDownLoadURLList.Count != 0)
	                        {
	                             _cdMovieInfo.TryAdd
	                             (movieInfo.Dy2018OnlineUrl,movieInfo);
	                        }
	                    }
	                });
	            }
	        }
	
	    }
	    catch(Exception ex)
	    {
	
	    }
	});
}

```

#### 获取电影详细信息
```csharp
 private MovieInfo FillMovieInfoFormWeb(AngleSharp.Dom.IElement a, 
 string onlineURL)
{
    var movieHTML = HTTPHelper.GetHTMLByURL(onlineURL);
    var movieDoc = htmlParser.Parse(movieHTML);
    //http://www.dy2018.com/i/97462.html 分析过程见上，不再赘述
    //电影的详细介绍 在id为Zoom的标签中
    var zoom = movieDoc.GetElementById("Zoom");
    //下载链接在 bgcolor='#fdfddf'的td中，有可能有多个链接
    var lstDownLoadURL = movieDoc.QuerySelectorAll("[bgcolor='#fdfddf']");
    //发布时间 在class='updatetime'的span标签中
    var updatetime = movieDoc.QuerySelector("span.updatetime"); 
    var pubDate = DateTime.Now;
    if(updatetime!=null && !string.IsNullOrEmpty(updatetime.InnerHtml))
    {
        //内容带有“发布时间：”字样，
        //replace成""之后再去转换，转换失败不影响流程
        DateTime.TryParse(updatetime.InnerHtml.Replace("发布时间：", 
        ""), out pubDate);
    }
    

    var movieInfo = new MovieInfo()
    {
        //InnerHtml中可能还包含font标签，做多一个Replace
        MovieName = a.InnerHtml.Replace("<font color=\"#0c9000\">","")
        .Replace("<font color=\"	#0c9000\">","")
        .Replace("</font>", ""),
        Dy2018OnlineUrl = onlineURL,
        MovieIntro = zoom != null ? WebUtility.HtmlEncode(zoom.InnerHtml) : "暂无介绍...",
        //可能没有简介，虽然好像不怎么可能
        XunLeiDownLoadURLList = lstDownLoadURL != null ?
        lstDownLoadURL.Select(d => d.FirstElementChild.InnerHtml).ToList() : null,
        //可能没有下载链接
        PubDate = pubDate,
    };
    return movieInfo;
}
```

#### HTTPHelper
这边有个小坑，dy2018网页编码格式是GB2312,.NET Core默认不支持GB2312，使用Encoding.GetEncoding("GB2312")的时候会抛出异常。

解决方案是手动安装System.Text.Encoding.CodePages包(Install-Package System.Text.Encoding.CodePages),

然后在Starup.cs的Configure方法中加入Encoding.RegisterProvider(CodePagesEncodingProvider.Instance),接着就可以正常使用Encoding.GetEncoding("GB2312")了。

```csharp
using System;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text;

namespace Dy2018Crawler
{
    public class HTTPHelper
    {

        public static HttpClient Client { get; } = new HttpClient();

        public static string GetHTMLByURL(string url)
        {
            try
            {
                System.Net.WebRequest wRequest = System.Net.WebRequest.Create(url);
                wRequest.ContentType = "text/html; charset=gb2312";

                wRequest.Method = "get";
                wRequest.UseDefaultCredentials = true;
                // Get the response instance.
                var task = wRequest.GetResponseAsync();
                System.Net.WebResponse wResp = task.Result;
                System.IO.Stream respStream = wResp.GetResponseStream();
                //dy2018这个网站编码方式是GB2312,
                using (System.IO.StreamReader reader =
                new System.IO.StreamReader(respStream,
                Encoding.GetEncoding("GB2312")))
                {
                    return reader.ReadToEnd();
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex.ToString());
                return string.Empty;
            }
        }
       
    }


}

```


### 定时任务的实现
定时任务我这里使用的是[Pomelo.AspNetCore.TimedJob](https://www.nuget.org/packages/Pomelo.AspNetCore.TimedJob/1.1.0-rtm-10026)。

Pomelo.AspNetCore.TimedJob是一个.NET Core实现的定时任务job库，支持毫秒级定时任务、从数据库读取定时配置、同步异步定时任务等功能。

由.NET Core社区大神兼前微软MVP[AmamiyaYuuko](https://www.nuget.org/profiles/AmamiyaYuuko)(入职微软之后就卸任MVP...)开发维护，不过好像没有开源，回头问下看看能不能开源掉。

nuget上有各种版本，按需自取。地址：https://www.nuget.org/packages/Pomelo.AspNetCore.TimedJob/1.1.0-rtm-10026

作者自己的介绍文章：[Timed Job - Pomelo扩展包系列](http://www.1234.sh/post/pomelo-extensions-timed-jobs)


#### Startup.cs相关代码
我这边使用的话，首先肯定是先安装对应的包：Install-Package Pomelo.AspNetCore.TimedJob -Pre

然后在Startup.cs的ConfigureServices函数里面添加Service,在Configure函数里面Use一下。

```csharp
// This method gets called by the runtime. Use this method to add services to the container.
public void ConfigureServices(IServiceCollection services)
{
    // Add framework services.
    services.AddMvc();
    //Add TimedJob services
    services.AddTimedJob();
}

 public void Configure(IApplicationBuilder app, 
 IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    //使用TimedJob
    app.UseTimedJob();

    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
        app.UseBrowserLink();
    }
    else
    {
        app.UseExceptionHandler("/Home/Error");
    }

    app.UseStaticFiles();

    app.UseMvc(routes =>
    {
        routes.MapRoute(
            name: "default",
            template: "{controller=Home}/{action=Index}/{id?}");
    });
    Encoding.RegisterProvider(CodePagesEncodingProvider.Instance);
}
        
```


#### Job相关代码

接着新建一个类，明明为XXXJob.cs,引用命名空间using Pomelo.AspNetCore.TimedJob，XXXJob继承于Job，添加以下代码。

```
    public class AutoGetMovieListJob:Job
    {
        
        // Begin 起始时间；Interval执行时间间隔，单位是毫秒，建议使用以下格式，此处为3小时；
        //SkipWhileExecuting是否等待上一个执行完成，true为等待；
        [Invoke(Begin = "2016-11-29 22:10", Interval = 1000 * 3600*3, SkipWhileExecuting =true)]
        public void Run()
        {
             //Job要执行的逻辑代码
             
            //LogHelper.Info("Start crawling");
            //AddToLatestMovieList(100);
            //AddToHotMovieList();
            //LogHelper.Info("Finish crawling");
        }
   }

```



### 项目发布相关

#### 新增runtimes节点

使用VS2015新建的模板工程，project.json配置默认是没有runtimes节点的.

我们想要发布到非Windows平台的时候，需要手动配置一下此节点以便生成。
```javascript

    "runtimes": {
    "win7-x64": {},
    "win7-x86": {},
    "osx.10.10-x64": {},
    "osx.10.11-x64": {},
    "ubuntu.14.04-x64": {}
  }

```

#### 删除/注释scripts节点
生成时会调用node.js脚本构建前端代码，这个不能确保每个环境都有bower存在...注释完事。
```javascript

    //"scripts": {
    //  "prepublish": [ "bower install", "dotnet bundle" ],
    //  "postpublish": [ "dotnet publish-iis --publish-folder %publish:OutputPath% --framework %publish:FullTargetFramework%" ]
    //},
```

#### 删除/注释dependencies节点里面的type

```javascript
"dependencies": {
    "Microsoft.NETCore.App": {
      "version": "1.1.0"
      //"type": "platform"
    },
```

project.json的相关配置说明可以看下这个官方文档：[Project.json-file](https://github.com/aspnet/Home/wiki/Project.json-file),
或者张善友老师的文章[.NET Core系列 ： 2 、project.json 这葫芦里卖的什么药](http://www.cnblogs.com/shanyou/p/5693453.html)

#### 开发编译发布
```shell
//还原各种包文件
dotnet restore;

//发布到C:\code\website\Dy2018Crawler文件夹
dotnet publish -r ubuntu.14.04-x64 -c Release -o "C:\code\website\Dy2018Crawler";

```

最后，照旧开源......以上代码都在下面找到：

Gayhub地址：[https://github.com/liguobao/Dy2018Crawler](https://github.com/liguobao/Dy2018Crawler)

在线地址：[http://codelover.win/](http://codelover.win/)


PS:回头写个爬片大家滋持不啊...























