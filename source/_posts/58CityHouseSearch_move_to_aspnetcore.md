---
layout: post
title: 58HouseSearch项目迁移到asp.net core
category: asp.net core
date: 2016-10-04 00:00:00
tags:
- asp.net core
- 58City
---

### 前言
[58HouseSearch](https://github.com/liguobao/58HouseSearch)这个项目原本是基于ASP.NET MVC 4写的，开发环境是Windows+VS2015，发布平台是linux+mono+jexus，这样看来整个项目基本已经满足跨平台的需求。

这样一来，本来我是没什么动力去做迁移的，好好的东西闲着没事干才迁移呢。

不过，这不国庆了么？穷人不是在家穷游天下么？所以...真的有点闲着没事干了。


### 迁移可行性探讨
项目迁移前，我们还是先来讨论一下迁移可行性。为嘛要进行可行性探讨呢？原因是.NET CORE是一个跨平台的框架，和上一代的.NET存在不兼容。

个人总结一下，迁移的主要的问题在于：代码不兼容、类库不兼容、严重依赖Windows API或者COM组件等。

#### 代码不兼容

代码不兼容其实不算麻烦。毕竟代码是活的，你我也是活的，不就是一个改字罢了。花点时间慢慢改，总是能搞掂的。


#### 类库不兼容

要不就弃用，要不就找替代品。

#### 严重依赖Windows API或者COM组件
额？找替代品，找不到可用替代品的话。放弃吧，这个项目别考虑迁移了。

这个故事告诉我们，做跨平台项目的时候，少点用系统API或者组建。


回到58HouseSearch项目上面。

这个项目的代码基本都是我写的，所以重写代码没什么问题。
依赖的类库有下面几个:

+ [AngleSharp](https://github.com/FlorianRappl/AngleSharp)

+ [Newtonsoft.Json](http://www.newtonsoft.com/json)

+ [log4net](http://logging.apache.org/log4net/)

AngleSharp是用来解析HTML的类库，用linq的方式来操作HTML，用起来实在爽快。

如果这货在.net core上不能跑，我应该立马放弃了。
不过，这个实在给力...

![AngleSharp支持平台](http://7xread.com1.z0.glb.clouddn.com/1a950803-3d38-4b16-9761-6c9cd806b0b9)


Newtonsoft.Json

在这个项目里面主要是用来记录PV数据的，非核心功能，可有可无。不过看了下nuget上的介绍，也是支持.net core的。

剩下log4net...嗯，并不支持log4net。不过这个就更加是非核心内容了，直接丢了。
PS:考虑后期加入Nlog替代log4net。

至于依赖Windows API之类的，在这个项目里面基本没有，所以略过...


### 准备工作
+ [Visual Studio Community 2015 with Update 3 – Free](https://www.visualstudio.com/downloads/)
+ [.NET Core SDK](https://www.microsoft.com/net/download)
+ [.NET Core](https://www.microsoft.com/net/download)
+ [.NET Core 1.0.1 - VS 2015 Tooling Preview 2](https://go.microsoft.com/fwlink/?LinkId=827546)


友情提示：

1. Visual Studio Community 2015 with Update 3 下载镜像来安装。

错误操作如下：
![错误操作](http://7xread.com1.z0.glb.clouddn.com/1e723e08-b3d5-4dab-b4a6-1de70799c4c8)

正确打开方式：

![正确的打开方式-1](http://7xread.com1.z0.glb.clouddn.com/706771d8-d3f2-4122-a61d-5e961887121a)

![正确的打开方式-2](http://7xread.com1.z0.glb.clouddn.com/455ec5cc-b429-4563-a4fb-3a0c18608969)

2. 安装.NET Core SDK和.NET Core之后再安装.NET Core 1.0.1 - VS 2015 Tooling Preview 2

3. 安装.NET Core 1.0.1 - VS 2015 Tooling Preview 2 这货的可能会报错0x80072f8a未指定的错误

解决方案见下图：

![图片描述](http://7xread.com1.z0.glb.clouddn.com/47c517d1-4b48-4088-be4a-a0768413e768)

详细见链接：[安装DotNetCore.1.0.1-VS2015Tools.Preview2.0.2出现0x80072f8a未指定的错误](http://www.cnblogs.com/JiaoWoWeiZai/p/5892255.html)


上面都弄好之后，理论上在VS2O15-新建项目里面可以看到ASP.NET CORE的模板了。如下图：
![ASP.NET CORE的模板](http://7xread.com1.z0.glb.clouddn.com/3364e7c9-a41e-47f2-8d08-60345e4efa35)


### 项目迁移

#### 新建空白ASP.NET CORE项目
新建好了之后如下图：

![空白ASP.NET CORE项目](http://7xread.com1.z0.glb.clouddn.com/e047d81d-56f1-4e51-9d72-7989e1fc6225)

#### Nuget获取引用

https://www.nuget.org/packages/AngleSharp/

https://www.nuget.org/packages/Newtonsoft.Json

#### 添加Controllers文件夹
然后把之前项目的Controllers拷贝过来，改掉命名空间，去掉无用代码，添加相应引用。

#### 添加Views文件夹
本项目直接把之前项目的Views拷贝过来是完全没有问题的。

#### 静态文件处理
asp.net core MVC中的文件结构和asp.net mvc的文件结构略有不同。

asp.net core MVC在view中“IMG/Little/PaleGreen.png”对应的文件对应于“项目路径/webroot/IMG/Little/PaleGreen.png”；

而asp.net mvc中，对应路径为“项目/IMG/Little/PaleGreen.png”。

因而，我们的所有静态文件都应该放到：webroot文件夹下。

上面的都做完了之后，项目结构如下：


![项目结构](http://7xread.com1.z0.glb.clouddn.com/2fee156b-2505-4953-bfe9-1d2521f13565)


接下来就是改代码了。



### 代码迁移


#### Startup.cs添加MVC
```
    public class Startup
    {
        // This method gets called by the runtime. Use this method to add services to the container.
        // For more information on how to configure your application, visit http://go.microsoft.com/fwlink/?LinkID=398940
        public void ConfigureServices(IServiceCollection services)
        {
            //添加MVC框架
            services.AddMvc();
            
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IHostingEnvironment env,
        ILoggerFactory loggerFactory)
        {
            loggerFactory.AddConsole();

            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }
            //启用静态文件中间件
            app.UseStaticFiles();
            //启动MVC路由
            app.UseMvcWithDefaultRoute();
            //设置默认页面
            app.UseMvc(routes =>
            {
                routes.MapRoute(
                    name: "default",
                    template: "{controller=House}/{action=Index}/{id?}"); 
            });

          
        }
    }

```


#### 改写GetHTMLByURL方法
之前的方法：

![old GetHTMLByURL](http://7xread.com1.z0.glb.clouddn.com/d005a100-e3e6-423b-b34c-3eae11b2ab63)

.net core重写了HttpWebRequest，变成了WebRequest,所以上面的代码废了。

重写如下：

```
        public static string GetHTMLByURL(string Url, string type = "UTF-8")
        {
            try
            {
                Url = Url.ToLower();

                System.Net.WebRequest wReq = System.Net.WebRequest.Create(Url);
                // Get the response instance.
                System.Net.WebResponse wResp = wReq.GetResponseAsync().Result;
                System.IO.Stream respStream = wResp.GetResponseStream();
                using (System.IO.StreamReader reader = new System.IO.
                StreamReader(respStream, Encoding.GetEncoding(type)))
                {
                    return reader.ReadToEnd();
                }
            }
            catch (System.Exception ex)
            {

                return string.Empty;
            }

        }
```

#### 改写Controller代码
嗯，换了命名空间，别的一句都没改直接拉过来了...略过。



### 发布到ubuntu

[Install for Ubuntu 14.04, 16.04 & Linux Mint 17](https://www.microsoft.com/net/core#ubuntu)

第一步
```
//Ubuntu 14.04 / Linux Mint 17
sudo sh -c 'echo "deb [arch=amd64] https://apt-mo.trafficmanager.net/repos/dotnet-release/ trusty main" > /etc/apt/sources.list.d/dotnetdev.list'
sudo apt-key adv --keyserver apt-mo.trafficmanager.net --recv-keys 417A0893
sudo apt-get update


//Ubuntu 16.04
sudo sh -c 'echo "deb [arch=amd64] https://apt-mo.trafficmanager.net/repos/dotnet-release/ xenial main" > /etc/apt/sources.list.d/dotnetdev.list'
sudo apt-key adv --keyserver apt-mo.trafficmanager.net --recv-keys 417A0893
sudo apt-get update

```

第二步
```
sudo apt-get install dotnet-dev-1.0.0-preview2-003131
```

安装好了之后，输入 dotnet -v 应该能看到版本信息，如下图：

![dotnet -v ](http://7xread.com1.z0.glb.clouddn.com/6ebca12e-2be9-487e-b230-d22562a5aabe)

这样的下，一句完成了ubuntu 运行asp.net core的环境搭建了。

### project.json里面隐藏的坑

#### dependencies

NET Core 1.0.1 - VS 2015 Tooling Preview 2模板的asp.net core 版本和ubuntu 的asp.net core 版本不一致。

根据微软爸给的教程，我们在ubuntu上安装的.NET Core 1.0.0，见上图。

然而我们创建项目的模板是.NET Core 1.0.1，见下图:

![.NET Core 1.0.1](http://7xread.com1.z0.glb.clouddn.com/a3192bdf-f548-4fad-868a-4865632acd29)

怎么办？要不升级ubuntu的asp.net core，要不降级。

由于没找到.NET Core 1.0.1 ubuntu的安装包，所以我选择了降级到.NET Core 1.0.0.

其中需要把Microsoft.NETCore.App version 、Microsoft.AspNetCore.Server.Kestrel、Microsoft.AspNetCore.Mvc 这三个节点都改成“1.0.0”。如下：

```
  "dependencies": {
    "Microsoft.NETCore.App": {
      "version": "1.0.1",
      "type": "platform"
    },
    "Microsoft.AspNetCore.Diagnostics": "1.0.0",
    "Microsoft.AspNetCore.Server.IISIntegration": "1.0.0",
    "Microsoft.AspNetCore.Server.Kestrel": "1.0.1",
    "Microsoft.Extensions.Logging.Console": "1.0.0",
    "Microsoft.AspNetCore.Mvc": "1.0.1",
    "Microsoft.AspNetCore.StaticFiles": "1.0.0",
    "Newtonsoft.Json": "9.0.1",
    "AngleSharp": "0.9.8.1"
  },

```

#### publishOptions
发布输出包括Views文件夹
```
  "publishOptions": {
    "include": [
      "wwwroot",
      "web.config",
      "Views"
    ]
  },
```

#### runtimes
runtimes 配置为模板运行平台。
详细见链接：[https://docs.nuget.org/ndocs/schema/project.json](https://docs.nuget.org/ndocs/schema/project.json)

```
  "runtimes": { "ubuntu.14.04-x64": {} }
```


上面都弄好之后，跑一下看,如下图：

```
dotnet restore

dotnet run
```

来个请求看看：

![请求log](http://7xread.com1.z0.glb.clouddn.com/b1191226-c33b-45c7-98cc-f62bb3ea73b4)


### jexus转发/反向代理

[ASP.NET Core "完整发布,自带运行时" 到jexus](http://www.cnblogs.com/gaobing/p/5663012.html)

















