---
layout:     post
title:      "圆心APP手记"
date:       2015-05-16 12:00:00
author:     "DongYeo"
header-img: "img/post-bg-02.jpg"
tags: ["PHP"]
---
**注：本篇为老博客平台系统迁移过来的**

# 背景
最近一个多月，外包接了个项目，做一个基于地理位置的社交软件，类型上和微信比较相似，目的就是为了约炮，服务端需要为手机客户端提供以下服务：restful API服务，聊天服务、图片服务、管理后台，验证短信发送等。在此记录开发中的点点滴滴。

---
# 准备
工欲善其事必先利其器，在服务器的选择上，选择了最近一直很火的阿里云。
主要使用了一下服务：

- ESC云服务器
- RDS云数据库
- OSS开放存储服务

---
# 实现
## ESC
### 架构
目前app还处于测试状态，服务器整体架构为一台RESTFul API服务程序服务器、一台聊天服务器、一台管理后台服务器。在未来app正式上线以后，RESTFul 服务可以横向扩展，结合LBS服务，实现服务的可靠性。

---
### 系统
服务器的系统统一选择了Centos 5.8 64位。
配置上为了节省开发前期的开支，全部选择了最低配置。

---
### RESTFul服务程序环境配置
由于RESTFul服务采用的是PHP yii2.0框架开发，所以，api服务器需要配置php 环境。因为yum安装的php版本过于老旧，yii框架无法正常使用。所以，采用编译安装的方式安装php：

```
# yum groupinstall "Development tools"
# yum -y install libxml2-devel gd-devel libmcrypt-devel curl curl-devel openssl-devel
# tar -xvf php-5.5.5.tar.gz
# cd php-5.5.5
# ./configure --prefix=/usr/local/php --with-apxs2=/usr/sbin/apxs  --enable-shared --with-libxml-dir --with-gd --with-openssl --enable-mbstring --with-mcrypt --with-mysqli --with-mysql --enable-opcache --enable-mysqlnd --enable-zip --with-zlib-dir --with-pdo-mysql --with-jpeg-dir --with-freetype-dir --with-curl --without-pdo-sqlite --without-sqlite3 --disable-fileinfo
# make
# make install
# cp php.ini-production /usr/local/php/lib/php.ini
在apache配置文件中添加
LoadModule php5_module modules/libphp5.so
上面那行可能在编译安装 php 的过程中已经由系统自动添加了

<FilesMatch \.php$>
	SetHandler application/x-httpd-php
</FilesMatch>
```

由于有短信验证的需求，短信验证码有有效期的限制，故需要给服务器安装memcache客户端和php memcached扩展：
memache程序直接

```
# yum install memcached

启动memcache的服务端：Memcached -d -m 10 -u root -l 127.0.0.1 -p 11211 -c 512 -P /tmp/memcached.pid

参数说明：
-d选项是启动一个守护进程；
-m是分配给Memcache使用的内存数量，单位是MB，我这里是10MB；
-u是运行Memcache的用户，我这里是root；
-l是监听的服务器IP地址我这里指定了服务器的IP地址127.0.0.1；
-p是设置Memcache监听的端口，我这里设置了11211，最好是1024以上的端口；
-c选项是最大运行的并发连接数，默认是1024，我这里设置了512，按照你服务器的负载量来设定；
-P是设置保存Memcache的pid文件，我这里是保存在 /tmp/memcached.pid；

安装php memcached 扩展：
#tar xvf memcache-3.0.5.tar
#cd memcache-3.0.5
#/usr/local/php/bin/phpize
#./configure --enable-memcache --with-php-config=/usr/local/php/bin/php-config --with-zlib-dir
#make
#make install Installing shared extensions:
/usr/local/php/lib/php/extensions/no-debug-non-zts-20060613/
注意上面这行返回的信息，将下面两行添加到/usr/local/php/lib/php.ini
extension_dir = "/usr/local/php/lib/php/extensions/no-debug-non-zts-20060613/"
extension=memcache.so
```
为方便版本控制和方便代码实时更新上传服务器实际生产环境，还需要配置SVN服务。

```
#yum install subversion
#mkdir ..../SVN 创建根目录 （这个随意）
创建repo库
#svnadmin create ..../SVN/repo
#cd  ..../repo/conf
#vi passwd //add user like this : user=password
#vi authz
[groups]            #组
admin = hello,www   #创建一个admin组，将用户加入到组
[/]                 #根目录权限设置（就是“svn”这个文件夹）
aaa = rw            #aaa对svn下的所有版本库有读写权限
[repo:/]            #repo:/,表示对repo版本库下的所有资源设置权限
@admin = rw         #admin组的用户对repo版本库有读写权限

#vim svnserve.conf
[general]
#匿名访问的权限，可以是read,write,none,默认为read
anon-access = none
#使授权用户有写权限
auth-access = write
#密码数据库的路径
password-db = passwd
#访问控制文件
authz-db = authz
#认证命名空间，subversion会在认证提示里显示，并且作为凭证缓存的关键字
realm = /opt/svn/repo

[root@localhost conf]#vim /etc/sysconfig/iptables
添加以下内容：
-A INPUT -m state --state NEW -m tcp -p tcp --dport 3690 -j ACCEPT
保存后重启防火墙
[root@localhost conf]#service iptables restart

#svnserve -d -r /opt/svn/repo

[root@localhost password]# killall svnserve //停止
[root@localhost password]# svnserve -d -r /opt/svn/repo // 启动
设置WEB服务器根目录为/var/www/html

checkout一份SVN
#svn co svn://localhost /var/www/html/yuanxin
修改权限为WEB用户
chown -R apache:apache /var/www/html/yuanxin

建立同步脚本

#cd /www/svndata/oplinux/hooks/

#cp post-commit.tmpl post-commit

编辑post-commit,在文件最后添加以下内容

export LANG=en_US.UTF-8
SVN=/usr/bin/svn
WEB=/www/webroot/
$SVN update \$WEB –username rsync –password rsync
chown -R apache:apache $WEB

增加脚本执行权限
#chmod a+x post-commit
```

这样就可以在本地使用SVN客户端实时将本地的代码commit到服务器的仓库，并同步到web应用下。

---
### 聊天服务程序环境配置
由于聊天服务采用的是php 的workerman 框架开发。Workerman是一款纯PHP开发的开源的高性能的PHP socket 服务器框架。

WorkerMan是以PHP命令行的模式运行的，所以需要安装PHP-CLI。
关于WorkerMan依赖的扩展：

1. pcntl扩展
pcntl扩展是PHP在Linux环境下进程控制的重要扩展，WorkerMan用到了其进程创建、信号控制、定时器、进程状态监控等特性。此扩展win平台不支持。
2. posix扩展
posix扩展使得PHP在Linux环境可以调用系统通过POSIX标准提供的接口。WorkerMan主要使用了其相关
的接口实现了守护进程化、用户组控制等功能。此扩展win平台不支持。
3. libevent扩展
libevent扩展使得PHP可以使用系统Epoll、Kqueue等高级事件处理机制，能够显著提高WorkerMan在高并发连接时CPU利用率。在高并发长连接相关应用中非常重要。libevent扩展不是必须的，如果没安装，则默认使用PHP原生Select事件处理机制。

安装：

1. 命令行运行（此步骤包含了安装php-cli主程序以及pcntl、posix、libevent扩展及github程序）
	yum install php-cli php-process git gcc php-devel php-pear libevent-devel
2. 命令行运行（此步骤是通过pecl安装libevent扩展，如果失败请尝试按照 4.1 环境要求 一节中使用源码
phpize的方式安装）
	pecl install channel://pecl.php.net/libevent-0.1.0
3. 命令行运行（此步骤是配置libevent的ini配置）
	echo extension=libevent.so > /etc/php.d/libevent.ini

具体的文档可以[点这里](http://www.workerman.net/)

---
