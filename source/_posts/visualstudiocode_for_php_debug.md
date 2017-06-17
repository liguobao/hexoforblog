---
layout: post
title: 用Visual Studio Code Debug世界上最好的语言
category: Visual Studio
date: 2017-03-17 00:00:00
tags:
- VisualStudio
- PHP
- Debug
---

#### 前言
这阵子因缘巧合接手了一个辣鸡项目，是用世界上最好的拍黄片写的，项目基本是另一个小伙伴在撸码，我就兼职打杂和发布做点运维的工作。

然后昨天项目上了测试版之后，一用起来Error满天飞了。让小伙伴查了很久都没有头绪，实在尴尬，只好自己动手了...

作为一个后端狗，虽然知道PHP大体原理和框架，看着项目的业务逻辑也大体知道个所以然，在此之前还是没撸过代码的。

看代码基本是Visual Studio Code或者HBuilder工具，本地跑代码很白痴的在用phpStudy。

Error出来了，第一反应就是debug咯...然后问了下小伙伴他以前怎么玩的，答曰：echo。

一口老血都...

查了下谷歌发现，Visual Studio Code + 插件是完全可以用来调试PHP的，所以就撸起了。


#### Visual Studio Code + php-debug插件 + phpStudy + xdebug

#### 安装Visual Studio Code

首先肯定是先下载[Visual Studio Code](http://code.visualstudio.com/) 咯。

安装好之后，随便在一个文件夹内鼠标“右键”，都能看到Open with code，打开之后如下图：

![图片描述](http://7xrayk.com1.z0.glb.clouddn.com/77bd5dbc-b738-4445-8598-2783ff876aca)

#### 安装Visual Studio Code php-debug插件
装好VS Code之后，接下来是安装一下PHP-Debug插件了。我们在插件商城搜索一下php，排名第二的PHP Debug就是我们要的插件了。
如下图：
![图片描述](http://7xrayk.com1.z0.glb.clouddn.com/02f2cba5-5872-4e8f-8d83-89d0e82c6540)

装好了之后重启一下vs code即可。

#### phpStudy
对于我这种懒人来说，去配置什么PHP运行环境肯定是不愿意的，那么类似的集成环境有么？

小伙伴和我说，你下个[phpStudy](http://www.phpstudy.net/)撸就算了，别去倒腾什么版本了。

然后...

![图片描述](http://7xrayk.com1.z0.glb.clouddn.com/23baf3f9-0fa7-446e-8f58-4d63c0306700)

下载好了安装完了，打开程序如下图：

![图片描述](http://7xrayk.com1.z0.glb.clouddn.com/b3836805-41f0-4d4d-8bbc-4a14e9557052)

看了下功能，其实这个软件就是集成了各种版本的PHP，可以方便切换PHP版本；同时自带一个Apache和MySQL，各种配置管理起来也挺方便的。
（感觉dalao们应该不怎么会用这么白痴的东西，233...

装好之后，启动一下服务，点击一下phpMyAdmin，看看它打开的网站是否能登录到本地的MySQL数据库。

如果可以，说明PHP环境应该是正常的了；如果有问题，请自行谷歌了...

接着切换PHP版本到意向版本，点击一下运行模式旁边的“切换版本”就可以选择版本了。

#### xdebug设置
[xdebug](https://xdebug.org/)是什么呢？
```
Xdebug作为PHP调试工具，提供了丰富的调试函数，
也可将Xdebug安装配置为zend studio、editplus调试PHP的第三方插件，

通过开启自动跟踪(auto_trace)和分析器功能，可以直观的看到PHP源代码的性能数据，
以便优化PHP代码。

引用自：[PHP调试工具Xdebug安装配置教程]
(http://www.cnblogs.com/qiantuwuliang/archive/2011/01/23/1942382.html)

```
我们可以在[xdebug.org](https://xdebug.org/)（自备梯子）上面下载到PHP各个版本的xdebug dll使用。

不过当我打开phpStudy的php-ini打算手动开启debug的时候，非常高兴得发现已经phpStudy已自带了对应版本的xdebug，而且路径都配好了。

phpStudy的php.ini在“其他选项-打开配置文件-php-ini”，如下图：

![图片描述](http://7xrayk.com1.z0.glb.clouddn.com/d53fd113-e81b-4de2-bab5-00e7d168c5cb)

把文档拉到最后，看得到xdebug的配置如下：

![图片描述](http://7xrayk.com1.z0.glb.clouddn.com/51f5f593-81cc-4c7c-b9ee-aa2c464e445b)

phpStudy已经帮我们配置好xdebug dll的路径了，我们只需要手动在zend_extension上面添加远程调试和自动启动配置即可，代码如下：

```

xdebug.remote_enable = 1
xdebug.remote_autostart= 1

```


完整配置如下：
```
[XDebug]
;xdebug.profiler_output_dir="C:\phpStudy\tmp\xdebug"
;xdebug.trace_output_dir="C:\phpStudy\tmp\xdebug"
xdebug.remote_enable = 1
xdebug.remote_autostart= 1
;你的PHP版本的php_xdebug.dll，phpStudy自动设置的
zend_extension="C:\phpStudy\php\php-5.5.38\ext\php_xdebug.dll"
```


保存文件，重启一下phpStudy服务。


#### Visual Studio Code 设置用户配置和调试配置

这个时候，我们随便在PHP文件夹中打开vs code，vs code会自动提示我们：Cannot validate since no PHP executable is set. Use the setting 'php.validate.executablePath' to configure the PHP executable.

嗯，没有设置PHP执行文件，可以通过设置php.validate.executablePath属性来配置它。

这个在哪配置呢？在“文件-首选项-设置”，打开之后如下图：


![php.validate.executablePath](http://7xrayk.com1.z0.glb.clouddn.com/d4dddab0-334c-45da-b89b-2ee3ed1517a4)

这个php.validate.executablePath对应就是当前phpStudy中运行的php.exe的路径，可以在phpStudy-其他选项菜单-打开文件位置-PHP中找到此路径。

保存好了之后，回到Visual Studio Code界面，转到Debug，选择添加配置，之后选择PHP，生成如下图的launch.json：


![Listen for XDebug](http://7xrayk.com1.z0.glb.clouddn.com/4a6d2f63-33eb-440f-97ec-e13d30ae6027)



不用改任何东西，直接开撸...

#### 开启Debug

确保phpStudy启动了，网站也正常运行起来了,然后在Visual Studio Code中启动调试，打上要的断点，接着启动调试。

如下图：

![图片描述](http://7xrayk.com1.z0.glb.clouddn.com/f327e135-9ec6-41a1-878c-0cdca4ca9c36)

接着访问你要调试的页面对应的PHP代码，打上你的断点，华丽丽的Debug出来了...

![图片描述](http://7xrayk.com1.z0.glb.clouddn.com/dba6c24d-dda4-4cc9-9682-a3e68f13d5a7)


F10单步调试，F11跳入函数，F5直接运行之类的快捷键自己玩吧。


#### 其他运行环境下的配置

基本没什么区别，配置php.ini，下载到对应版本的xdebug.dll，php.validate.executablePath配置正确就完事。

其他参考链接：

1. [ 如何使用XDebug调试php](http://blog.csdn.net/ruihanchen/article/details/7705842)

2. [XAMPP环境下用phpStorm+XDebug进行断点调试的配置](http://www.chenxuanyi.cn/xampp-phpstorm-xdebug.html)

3. [PHPStorm下XDebug配置](http://blog.csdn.net/dc_726/article/details/9905517)


#### PS:果然是世界上最好的语言...(逃




































