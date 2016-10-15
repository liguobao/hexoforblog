---
layout: post
title : Microsoft .NET Core 1.0.0 VS 2015 Tooling Preview 2 0x80070003
category: .net core
---

最近安装Microsoft .NET Core 1.0.0 VS 2015 Tooling Preview 2实在有点曲折，忍不住写个文章来讲这货的坑爹之旅了。

一般我们在[.NET Downloads](https://www.microsoft.com/net/download) 下载回来的Microsoft .NET Core 1.0.0 VS 2015 Tooling Preview 2是一个简易安装包。

它在安装过程中会不断去网络请求需要的msi文件，美名曰：在线安装。

然而在我国的国情以及我国网络运营商的衬托下，在线安装这种东西实在不可恭维。

本来网络稳定，在线安装 这种鬼也还算能用，不过最近微软爸爸不知道为嘛了，.net core相关安装包的下载地址全线失效。
如[DotNetCore.1.0.1-SDK.1.0.0.Preview2-003133-x64.exe](https://download.microsoft.com/download/0/A/3/0A372822-205D-4A86-BFA7-084D2CBE9EDF/DotNetCore.1.0.1-SDK.1.0.0.Preview2-003133-x64.exe)，直接出502 Bad Gateway。

这个还能用迅雷或者命令行下载回来，但是DotNetCore.1.0.1-SDK.1.0.0.Preview2这货安装过程中需要的一下msi安装包，就死活下不回来了。

#### 安装过程报错：0x80070003 系统找不到需要的文件。

此处已确认是微软爸爸的bug了。[issue在这里](https://github.com/aspnet/Tooling/issues/655)

我们去看安装失败的log，能看到类似下面的log：

Error 0x80072efd: Failed attempt to download URL: 'https://download.microsoft.com/download/A/3/8/A38489F3-9777-41DD-83F8-2CBDFAB2520C/packages/ancm_iis_express_x64_en_rc2_39.msi' to: 'C:\Users\cneuss\AppData\Local\Temp{C307771D-8D9A-45B5-B514-B6CA69C0C6E2}\ANCM_IISExpress_x64'


很明显这个操作从[https://download.microsoft.com/download/A/3/8/A38489F3-9777-41DD-83F8-2CBDFAB2520C/packages/ancm_iis_express_x64_en_rc2_39.msi](https://download.microsoft.com/download/A/3/8/A38489F3-9777-41DD-83F8-2CBDFAB2520C/packages/ancm_iis_express_x64_en_rc2_39.msi)上面下载ancm_iis_express_x64_en_rc2_39.msi文件。
然而我们点击进去，看到这个同样的问题：502 Bad Gateway。


安装到这里，已经GG了。

不过既然知道是因为下载文件的问题了，那么解决办法也应运而生了。

我们完全手动下载文件（用迅雷或者别的下载工具），发布在本地，改hosts地址让下载请求直接从本地下载文件。

这个方案听起来是没有任何的问题的，我也成功使用迅雷把无法下载的文件下载到本地了。然而在改host地址这里卡住了。

不知道为嘛，无论我把[download.microsoft.com](download.microsoft.com)指向怎么改，也无法把请求重定向到本地。

后来认真看了下安装log，发现所以的下载操作都是把文件下载到 C:\Users\cneuss\AppData\Local\Temp 目录（具体目录看log文件内容）下的临时路径，报错是系统找不到需要的文件。

那么，我们手动把需要的文件放到对应的位置，问题也就解决了。

所以，为了清晰找到临时路径，安装之前先把“C:\Users\cneuss\AppData\Local\Temp”的文件清空，然后点击安装包。

很快可以在temp路径下看到冒出来的{xxxxxxxxxxxxx}文件夹，然后迅速把我们通过迅雷下载回来的文件仍到这个目录底下。

迅速操作的原因是，在线安装包正在下载所需要的文件，如：ancm_iis_express_x64_en_rc2_39.msi，我们要在下载超时之前把文件应该在的位置，这样的话即使下载没拿到文件，安装程序还是可以拿我们的文件去执行安装。

别的就是不断尝试，看缺少那个msi手动下哪个msi的事情了。

#### 最终方案
最后，根据log文件，还发现了一个更简单的方法：

在DotNetCore.1.0.1-SDK.1.0.0.Preview2-003133-x64.exe的同级目录下新建packages文件夹，把上面无法在线下载到的msi仍进去，然后就一路绿灯了。






 
