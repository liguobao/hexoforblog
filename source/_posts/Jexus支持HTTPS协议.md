---
layout: post
title: Jexus支持HTTPS协议
category: jexus
---


众所周知，在HTTPS页面请求HTTP资料的时候，现代浏览器会拦截，提示用户是否继续，或者直接拦截，提示都不出来。


最近给自己做了个快速书签工具，点击书签就直接把书签发送到服务器地址，然后保存到我的网站中。

一开始一切都挺正常的，不过遇到了https的网站的时候，就跪掉了。

开始的时候看到HTTPS证书是收费的，想想还是算了，反正凑合能用就是。前几天偶尔看到有一个免费申请HTTPS的开源软件，喵了一下看起来还不错，这几天有空了立马开干。下面有一个教程，我申请证书差不多就是按照这个来处理的。

[用Let’s Encrypt获取免费证书](https://www.paulyang.cn/blog/archives/39?spm=5176.blog2666.yqblogcon1.12.Nu0TgL)


关于这个Let's Encrypt，维基百科是这样介绍的：

> Let's Encrypt 是一个将于2015年末推出的数字证书认证机构，将通过旨在消除当前手动创建和安装证书的复杂过程的自动化流程，为安全网站提供免费的SSL/TLS证书。  Let's Encrypt 是由互联网安全研究小组（ISRG，一个公益组织）提供的服务。主要赞助商包括电子前哨基金会，Mozilla基金会，Akamai以及思科。2015年4月9日，ISRG与Linux基金会宣布合作。用以实现这一新的数字证书认证机构的协议被称为自动证书管理环境（ACME）。  GitHub上有这一规范的草案，且提案的一个版本已作为一个Internet草案发布。Let's Encrypt 宣称这一过程将十分简单、自动化并且免费。  2015年8月7日，该服务更新其推出计划，预计将在2015年9月7日当周某时发布首个证书，随后向列入白名单的域名发行少量证书并逐渐扩大发行。若一切按计划进行，该服务预计将在2015年11月16日当周某时全面开始提供.


整个项目在Github有代码，主要是通过客户端来为我们的网站生成https证书。
首先我们先下载客户端，如下：
```shell
git clone https://github.com/letsencrypt/letsencrypt.git

```
接着进入这个仓库内，执行下面代码：
```shell
./letsencrypt-auto certonly -a 
webroot\ --webroot-path 网站所在路径(如：/var/www/web/) \ 
-d 你的域名(如：test.online) -d www.你的域名(如ww.test.online)

```
这里需要注意的事，我这里为了排版，给上面的命令加了换行，运行这个命令的时候记得把换行符去掉。
换行符在webroot、-d 前面各有一个。

一切顺利的话，我们在`/etc/letsencrypt/live/域名/`这个目录下能看到四个文件，分别是：

1. 域名证书文件
2. 签发域名证书的证书链文件
3. 域名证书+证书链文件
4. 私钥文件

如下图：
![letsencrypt文件](http://7xread.com1.z0.glb.clouddn.com/60e4f29a-6da5-40e1-ae32-453a3bbf2455)

接着就是为网站设置证书了。


Jexus设置HTTPS要更改jws.conf文档以及网站的配置文档。

操作步骤如下：

1. 修改jws.conf
进入Jexus文件夹中，打开 “jws.conf”，添加下面两句：

```shell
	CertificateFile    = /etc/letsencrypt/live/域名/fullchain.pem
	CertificateKeyFile = /etc/letsencrypt/live/域名/privkey.pem
```

修改之后效果图如下：
![图片描述](http://7xread.com1.z0.glb.clouddn.com/d306d9c5-6391-421d-86fc-053b97d1b489)


2. 开启网站的HTTPS功能

进入siteconf/文件夹，找到对应的网站conf文件，

把网站服务端口改为443：
port=443

启用https：
UseHttps=true

修改之后的效果图如下：
![图片描述](http://7xread.com1.z0.glb.clouddn.com/0800dc87-2500-42d2-a3c5-a75a2c819330)

然后重启jexus即可。

完了之后，通过HTTPS即可访问。

最后上一个HTTPS证书的图证明一下这个是可行的。
![图片描述](http://7xread.com1.z0.glb.clouddn.com/24842774-311e-4e55-a6b5-b88a89edc754)


撒花，下次再见。















