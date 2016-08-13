---
layout: post
title: CodeSmith 连接MySQL数据库报“can't find .net framework data provider”
category: CodeSmith
---

1、下载 mysql-connector-net 安装

[mysql-connector-net](https://dev.mysql.com/downloads/connector/net/6.9.html)

2、mysql-connector-net 安装完毕之后，到对应的安装目录下，把对应的MySQL .NET dll拷贝到 CodeSmith的bin目录和SchemaProviders目录。

一般DLL所在目录是：

C:\\Program Files (x86)\\MySQL\\MySQL Connector Net 6.9.8\\Assemblies\\v4.0


3、重启CodeSmith生效


<br>
<br>

其余解决方案：
<br>
[codesmith无法连接Mysql的解决方法](http://blog.csdn.net/joke01/article/details/9469515)

[codesmith6.5连接Mysql提示“找不到请求的 .Net Framework Data Provider。可能没有安装。”解决方法](http://www.cnblogs.com/tim190/archive/2013/01/18/2866161.html)




