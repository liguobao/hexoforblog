---
layout: post
title: ubuntu相关软件配置备忘录
category: ubuntu
date: 2017-03-25 00:00:00
tags:
- linux
- memorandum
- shell
---


#### 端口占用相关

```
//查找端口占用情况
$ netstat -anp | grep "5000"
 
//干掉某个进程
$ kill -9 2553
 
//后台运行 
$ nohup command & 

```


#### 安装SS
```
$ apt-get update
//安装ss
$ apt-get install python-pip 
$ pip install shadowsocks
//运行ss服务端
$ nohup ssserver -s ip地址 -k 密码 &
```

#### 安装MySQL5.7
```
http://tecadmin.net/install-mysql-5-on-ubuntu/

$ sudo apt-get install software-properties-common
$ sudo add-apt-repository -y ppa:ondrej/mysql-5.7
$ sudo apt-get update
$ sudo apt-get install mysql-server

//编辑此处，允许远程登录
/etc/mysql/my.cnf

bind-address		= 0.0.0.0

//重启MySQL
$ /etc/init.d/mysql restart

```
#### MySQL添加用户
```
//新增用户
grant all privileges  on *.* to root@'%' identified by "root";
//立即生效
flush privileges;


grant select,insert,update,delete,create,drop on  to joe@10.163.225.87 identified by ‘123′;


grant all  on xxxDB.* to xxx@'%';

```


#### MySQL提示“Checking for tables which need an upgrade, are corrupt or were not closed cleanly”

操作步骤：

```
$ sudo service mysql stop 
$ sudo /etc/init.d/apparmor reload
$ sudo service mysql start


或者：

down vote
This error occurs due to multiple installations of mysql. Run the command:

$ ps -A|grep mysql
Kill the process by using:

$ sudo pkill mysql
and then run command:

$ ps -A|grep mysqld
Also Kill this process by running:

$ sudo pkill mysqld
Now you are fully set just run the following commands:

$ service mysql restart
mysql -u root -p

```




#### 7z相关操作
```
//压缩
$ 7z a -t7z -r manager.7z /home/manager/*
//解压
$ 7z X xx.zip

```




#### 腾讯云修改密码,允许root登录

```
$ sudo passwd root

$ sudo vi /etc/ssh/sshd_config

$ sudo service ssh  restart
```

#### 开机启动相关
```
//查看开机启动相关程序
$ ls /etc/rc*
//安装sysv-rc-conf
$ sudo apt-get install sysv-rc-conf
$ sudo sysv-rc-conf 

```





