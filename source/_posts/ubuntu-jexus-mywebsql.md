---
layout: post
title: ubuntu使用Jexus搭建MyWebSQL
category: ubuntu
date: 2016-04-07 00:00:00

---


之前在阿里云上装了一个ubuntu，后来也没怎么用力，就挂这一个mysql数据库。最近在家里用MySQL Workbench 连接阿里云上面的MySQL的时候，连着过了一会就中断了。后来看了一圈回来才发现，目测是家里电信宽带的锅，不断给我动态分配IP地址....后来群里面的小伙伴说，搭个websql了事啦。听起来不错的想法，于是昨天就试了一下。

之前在ubuntu上装过apache，后来为了跑asp.net，把apache停了，换成了jexus。
Jexus是国内.NET 跨平台大牛们写的一个web服务器，使用方便，很稳定，也在不断加入新特性。相关资料直接访问[www.jexus.org](http://www.jexus.org/)。

jexus是以mono为基础的，其实首先应该先配置mono的运行环境。

###第一步 安装mono
相关资料链接：


[在Ubuntu操作系统上安装mono的具体方法](http://www.linuxdot.net/bbsfile-3090)

[Ubuntu 14.04 安装 Mono](http://www.isvee.com/archives/763)



我的ubuntu老早之前就安装好了mono，这个就此瞥过咯。


###第二步 安装jexus

[Jexus web server V5.1 安装配置要点](http://www.linuxdot.net/bbsfile-3084)

[jexus首页](http://www.jexus.org/)


```
A、安装：
cd /tmp
wget linuxdot.net/down/jexus-5.8.1.tar.gz 
tar -zxvf jexus-5.8.1.tar.gz 
cd jexus-5.8.1 
sudo ./install 

B、更新
cd /tmp
sudo /usr/jexus/jws stop
wget linuxdot.net/down/jexus-5.8.1.tar.gz
tar -zxvf jexus-5.8.1.tar.gz
cd jexus-5.8.1
sudo ./upgrade

```

5.8.1差不多是现在最新版本了。

###第三步 jexus 支持PHP

先在ubuntu上安装一下PHP5-CGI.

[用 Jexus ASP.NET WEB服务器搭建 PHP 网站的具体方法](http://www.linuxidc.com/Linux/2012-05/60172.htm)


总结来说就是下面两句：
```
sudo apt-get update

sudo apt-get install php5-cgi

```

接着：

```
1)修改“/etc/php.ini”文件:

找到cgi.force_redirect=1一行，把前边的"#"号去掉，把值从1改为0，如：

cgi.force_redirect=0

2)修改jws.conf。打开jexus文件夹中的jws.conf，作如下配置：

填写PHP-CGI程序路径和工作进程数。如：“php-fcgi.set=/usr/bin/php-cgi,6”。

3)修改网站配置。在需要使用PHP的网站的配置文件中添加:

fastcgi.add=php|socket:/var/run/jexus/phpsvr
```

[Jexus 支持PHP的三种方式-张善友](http://www.cnblogs.com/shanyou/p/3369322.html)


搞完上面这些，理论上你的jexus已经能跑PHP网站了。



###第四步 安装mywebsql

[mywebsql首页](http://mywebsql.net/)

mywebsql跑起来应该是下图的：

![mywebsql效果图](http://7xread.com1.z0.glb.clouddn.com/7d902b94-f132-4041-84fa-78f044f91358)

[下载地址](https://sourceforge.net/projects/mywebsql/files/stable/mywebsql-3.6.zip/download)

```
cd /tmp

wget https://sourceforge.net/projects/mywebsql/files/stable/mywebsql-3.6.zip

cp mywebsql-3.6.zip /var/www 

cd /var/www

tar -zxvf mywebsql-3.6.zip 

```


把mywebsql网站文件弄好之后，就可以去看jexus配置php网站了。

jexus的网站配置文件夹一般路径就是/usr/jexus/siteconf/

```
cd /usr/jexus/siteconf/

vi mywebSQL #创建网站配置文件

cd .. 

./jexus restart

```

上面的mywebSQL里面就写网站配置了，主要是端口号/运行环境之类的配置。

贴一下我的配置：

```
#仅供参考
######################
# Web Site: Default
########################################

port=2016
root=/ /var/www/mywebsql
hosts=*    #OR your.com,*.your.com
usephp =true
fastcgi.add=php|socket:/var/run/jexus/phpsvr

# addr=0.0.0.0
# CheckQuery=false
# NoLog=true
# NoFile=/index.aspx
# Keep_Alive=false
# UseGZIP=true
# UseHttps=true
# DenyFrom=192.168.0.233, 192.168.1.*, 192.168.2.0/24
# AllowFrom=192.168.*.*
# DenyDirs=~/cgi, ~/upfiles
# indexes=myindex.aspx
# rewrite=^/.+?\.(asp|php|cgi|pl|sh)$ /index.aspx

# reproxy=/bbs/ http://192.168.1.112/bbs/

# Jexus php fastcgi address is '/var/run/jexus/phpsvr'
#######################################################
# fastcgi.add=php|socket:/var/run/jexus/phpsvr

# php-fpm listen address is '127.0.0.1:9000'
############################################
# fastcgi.add=php|tcp:127.0.0.1:9000

```


到这里，访问http://你的主机IP:上面配置的端口号 就能看到下面的页面了。

![MywWebSQL登陆页](http://7xread.com1.z0.glb.clouddn.com/df3951c0-9d3d-4085-b577-743df68c1d98)


输入账号密码就能登陆。


###然而....
我登陆的时候显示，系统提示：没有安装客户端库。

###第五步 配置PHP MySQL库


于是又跑去看了一下MyWebSQL的说明，文档上说可以在/install.php上面看配置。



![这是配置好的效果图](http://7xread.com1.z0.glb.clouddn.com/cf8d88d8-fb5d-42f0-81cc-e0fbb566ebe5)

显示：

MySQL Client Library	client library is not installed
MySQL improved functionality	client library is not installed


好吧，PHP MySQL客户端库没有安装....

那就安装咯。
于是找到了下面一个文章：
[ZH奶酪：Ubuntu 14.04安装LAMP(Linux，Apache，MySQL，PHP)](http://www.cnblogs.com/CheeseZH/p/4694135.html)

安装一下基础库
```
sudo apt-get install php5 libapache2-mod-php5 php5-mcrypt php5-curl php5-imagick php5-cli

```

搜索一下还有什么库可以安装。

apt-cache search php5-


![](http://7xread.com1.z0.glb.clouddn.com/7bac43bc-01ea-4ce0-ba4b-37a06a51fe3a)


```
sudo apt-get install php5 php5-mysqlnd 

sudo apt-get install php5 php5-mysqlnd-ms

```

接着重启一下jexus的网站，万事大吉。

