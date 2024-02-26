---
title: "Ubuntu18.04离线安装java Nginx Mysql"
date: 2024-02-26T19:21:08+08:00
lastmod: 2024-02-26T19:21:08+08:00
draft: false
author: "帕里亚"
description: "在无网络的机器中无法直接使用apt-get进行安装，而如果下载.deb或是.rpm包则需要依次安装其所需的依赖包，相对复杂；本文利用打包好的tar.gz或是源码编译可以实现不需要apt-get的Java、Nginx和MySQL安装，从而满足网页应用的部署。"

tags:        ["网页部署"]
categories:  ["Blog"]

lightgallery: true
---

<!--more-->

## Java1.8的环境配置

官方的下载地址为：[Java SE 8](https://www.oracle.com/cn/java/technologies/javase/javase8u211-later-archive-downloads.html)

选择任意版本的Linux ARM64 Compressed Archive进行下载（需要登陆Oracle账号），如[jdk-8u391-linux-aarch64.tar.gz](https://www.oracle.com/cn/java/technologies/javase/javase8u211-later-archive-downloads.html#license-lightbox)

华为镜像：[jdk1.8.0_202](https://repo.huaweicloud.com/java/jdk/8u202-b08/jdk-8u202-linux-arm64-vfp-hflt.tar.gz)

此处以jdk1.8.0_391为例，可以将.tar.gz文件解压到任意路径下，如/home/paliya/env

```sh
tar -zxvf jdk-8u391-linux-aarch64.tar.gz -C /home/paliya/env
```

从而获得JDK的安装路径/home/paliya/env/jdk1.8.0_391，利用这个地址编辑```/etc/profile```进行环境变量配置，在文件的最后增加以下3行

```sh
JAVA_HOME=/home/paliya/env/jdk1.8.0_391
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```

保存后通过```source /etc/profile```生效配置，随后就可以通过```java --version```查看安装版本从而验证是否配置成功。

## Nginx1.18离线安装

可以通过[https://github.com/nginx/nginx/tags](https://github.com/nginx/nginx/tags)选择想要使用的版本，此处以Nginx1.18为例，下载地址：[Nginx1.18](https://github.com/nginx/nginx/archive/refs/tags/release-1.18.0.tar.gz)

将.tar.gz文件解压到任意路径：

```sh
tar -zxvf nginx-1.18.0.tar.gz -C /home/paliya/env
```

获得nginx-1.18.0文件夹，通过```./configure```进行nginx的编译配置，在配置的最开始会输出各项路径信息，其默认安装位置为/usr/local/nginx，再通过```sudo make && make install```完成编译：

```sh
cd /home/paliya/env/nginx-1.18.0
./configure
sudo make && make install 
//此处也可以直接使用 sudo make install 完成编译
```

我在部署过程中没有遇到缺少pcre、zlib或是openssl的情况，如有此类依赖缺失可以参考[https://blog.csdn.net/u012549626/article/details/126432074](https://blog.csdn.net/u012549626/article/details/126432074)和[https://blog.csdn.net/alfiy/article/details/123206823](https://blog.csdn.net/alfiy/article/details/123206823)。

完成编译后可以通过/usr/local/nginx/sbin/nginx启动nginx

```sh
cd /usr/local/nginx/sbin
sudo ./nginx -c /usr/local/nginx/conf/nginx.conf
```

可以通过本地浏览器访问localhost或是外部连接访问IP地址验证nginx是否配置完成，如果完成配置则会出现如下画面：

![nginx欢迎](https://img-blog.csdnimg.cn/direct/750ef468efba4cfd87a1f2b977ee9b0c.png)

可以通过修改/usr/local/nginx/conf中的nginx.conf来部署自己的网站，随后通过创建/lib/systemd/system/nginx.service来建立服务，可以先通过```./nginx -s stop```终止nginx进程：

```sh
sudo vim /lib/systemd/system/nginx.service
```

```sh
[Unit]
Description=nginx server
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/local/nginx/sbin/nginx -c reload
ExecStop=/usr/local/nginx/sbin/nginx -c stop
[Install]
WantedBy=multi-user.target
```

```sh
sudo systemctl daemon-reload
sudo systemctl enable nginx.service
sudo service nginx start
// 开机自启服务
sudo systemctl enable nginx.service
sudo service nginx status
```

![nginx服务配置](https://img-blog.csdnimg.cn/direct/c0a98ac017c44385b682b6a3c364dd4a.png)

## MySQL8.2.0离线安装

MySQL下载地址：[https://downloads.mysql.com/archives/community/](https://downloads.mysql.com/archives/community/)

![mysql下载地址](https://img-blog.csdnimg.cn/direct/3281a49cd57742e2866b2358206f4e67.png)

此处我下载的是8.2.0的mini版本：[Linux - Generic (glibc 2.17) (ARM, 64-bit), Compressed TAR Archive
Minimal Install](https://downloads.mysql.com/archives/get/p/23/file/mysql-8.2.0-linux-glibc2.17-aarch64-minimal.tar.xz)

将mysql-8.2.0-linux-glibc2.17-aarch64-minimal.tar.xz文件解压到任意路径：

```sh
tar -xvf mysql-8.2.0-linux-glibc2.17-aarch64-minimal.tar.xz -C /home/paliya/env
// 解压完成后文件夹名过长，可以进行重命名
mv /home/paliya/env/mysql-8.2.0-linux-glibc2.17-aarch64-minimal /home/paliya/env/mysql-8.2.0
// 或是直接建立软连接
sudo ln -s /home/paliya/env/mysql-8.2.0-linux-glibc2.17-aarch64-minimal /usr/local/mysql
```

随后创建数据库存储文件夹如```/data/mysql/```，再编写```/etc/my.cnf```配置文件，内容如下:

```sh
[mysqld]
bind-address=0.0.0.0
port=3306
user=mysql
basedir=/usr/local/mysql
datadir=/data/mysql
socket=/tmp/mysql.sock
log-error=/data/mysql/mysql.err
pid-file=/data/mysql/mysql.pid
#character config
character_set_server=utf8mb4
symbolic-links=0
```

随后通过以下指令完成mysql的初始化：

```sh
cd /usr/local/mysql/bin/
./mysqld --defaults-file=/etc/my.cnf --basedir=/usr/local/mysql/ --datadir=/data/mysql/ --user=mysql --initialize
```

通过```cat /data/mysql/mysql.err```来查看初始密码，如图所示：

![初始密码](https://img-blog.csdnimg.cn/direct/4153022a3cce42cf869a344b954bd096.png)

如果初始化时报提示缺少libaio、libmecab2等依赖可以参考：
1. [离线安装MySQL缺少libaio.so.1文件——并离线安装libaio.so.1](https://blog.csdn.net/itwxming/article/details/109221937)；
2. [Ubuntu18.04，离线安装mysql 5.7.39数据库](https://blog.csdn.net/GL666/article/details/129033236)
3. 或是在[https://launchpad.net/ubuntu/bionic/arm64](https://launchpad.net/ubuntu/bionic/arm64/)搜索所需要的依赖进行下载和安装。

最后创建mysql.service，具体命令如下：

```sh
sudo cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql
sudo systemctl daemon-reload
sudo service mysql start
// 开机自启服务
sudo systemctl enable mysql.service
```

可以通过```service mysql status```查看mysql的运行状态，此时需要进入```/usr/local/mysql/bin```目录下连接mysql，如果需要将mysql用于全局调用可以参考博文：[linux如何将应用程序（使用源码安装的软件）全局可用（以coverage为例）](https://blog.csdn.net/lwgkzl/article/details/81058961)

此处我使用的方法是编辑```/etc/profile```或是```~/.bashrc```，在文件末尾增加：

```sh
export PATH=/usr/local/mysql/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/mysql/lib:$LD_LIBRARY_PATH
```

最后可以更改mysql中root用户的密码，如果完成全局环境配置则可以使用```mysql -uroot -p'初始密码'```，由于初始密码可能存在&符号导致命令错误，因此引号是必须的，进入mysql后更改密码的指令如下：

```sql
alter user 'root'@'localhost' identified by '更新密码';
alter user 'root'@'localhost' identified WITH mysql_native_password BY '更新的密码;
flush privileges;
```

至此结束了mysql的环境配置，如果需要使用C++连接数据库，此处建议开发环境和部署环境采用同种mysql部署方式，即均在线环境配置或均离线做环境配置，否则会出现找不到.sock文件的错误，可以参考:

1. [linux：ubuntu安装mysql，记一次mysqld.sock的坑](https://blog.csdn.net/weixin_34184561/article/details/92427486)
2. [mysql.sock作用-解决mysql.sock直接找不到了的问题-重新生成mysql.sock](https://blog.csdn.net/fantaxy025025/article/details/84918795)

等文章对mysql.sock的处理，我是通过创建软连接实现mysql.sock的连接，命令如下：

```sh
sudo mkdir /var/run/mysqld
sudo ln -s /tmp/mysql.sock /var/run/mysqld/mysqld.sock
```

## Reference

> [arm64架构的Linux安装jdk8](https://blog.csdn.net/baidu_33542504/article/details/121248348)
>[Ubuntu18.04 离线安装nginx](https://blog.csdn.net/alfiy/article/details/123206823)
>[【Linux-ARM】安装Nginx](https://blog.csdn.net/u012549626/article/details/126432074)
>[arm版(以uos为例)linux安装mysql8](https://blog.csdn.net/yxl0011/article/details/128844397)
>[Ubuntu（arm架构）安装MySQL8.0以上版本，并使用phpMyAdmin进行管理](https://blog.csdn.net/tuzh6/article/details/130324004)
>[Mysql安装包方式安装，解决mysqld.sock无法连接](https://blog.csdn.net/weixin_43283363/article/details/109662322)