---
layout: posts
title: 我所用到的一些Docker常用的命令
date: 2018-08-13 00:38:36
categories: Docker
tags: [Docker,Nginx]
top: true
---
以前没有服务器的时候，在Windows上边跑一个虚拟机，里边装一个CentOS7感觉还挺开心的，虽然我不是专门干运维的，但是有哪一个程序员不懂Linux呢？特别是我这种搞Java的，需要数据库、缓存等等等了，有些在Windows上边不是不可以跑，但是真的费劲啊。正常生产上的环境都是用的Linux服务器，我第一次接触Docker大概是在什么什么时候忘了，好像是看了尚硅谷的教学视频，然后就知道Docker特别好用，一键部署？反正挺好的，官网上镜像有的是，Pull下来跑就完事了。等我有了自己的Linux服务器，我才知道原来Docker是这么好用啊，什么这个源码安装了，那个源码安装了，都靠边吧（特别是TM的Nginx），好了，我写下我经常用的Docker命令吧，到时候用也方便查。

<!--more--> 

下面我就用Tomcat为例，写下命令吧。 Docker镜像库：https://hub.docker.com/

##### 一、搜索Docker镜像

```shell
docker search tomcat
```

一般都用官网的，用的毕竟是官方的嘛。

##### 二、拉取镜像

```shell
# 默认拉取官方最新版
docker pull tomcat

#指定某一个版本拉取
docker pull tomcat:8.5.35
```

##### 三、查看镜像

```shell
docker images
```

##### 四、删除镜像

```shell
docker rmi tomcat
```

##### 五、运行镜像

```shell
docker run --name tomcat-guns -d -p 8080:8080 tomcat
```

>  –name 自定义容器名
>
>  -d 后台运行
>
>  -p 端口映射，将容器的8080端口映射到宿主机的8080端口

##### 六、查看运行中的镜像

```shell
#运行中的镜像
docker ps 
#查看运行中的镜像，包括已经退出的
docker ps -a
```

##### 七、容器的停止/重启

```shell
#停止容器
docker stop tomcat-guns
#重启容器
docker restart tomcat-guns
#启动停止的容器
docker start tomcat-guns
#或者也可以使用镜像的ID都成
```

##### 八、删除容器

```shell
#容器必须在停止状态才能删除
docker rm tomcat-guns
```

##### 九、查看容器的日志

```shell
docker logs tomcat-guns
```

##### 十、进入容器内

```shell
docker exec -it tomcat-guns /bin/bash
```

##### 十一、插件容器的IP地址

```shell
#查看容器IP
docker inspect mysql | grep Address
```

##### 十二、一些软件的部署

###### （一）MySQL服务

```shell
#MySQL 可以指定密码,我用的是5.7.24版本
docker run -p 3306:3306 --name mysql -d -e MYSQL_ROOT_PASSWORD=123456 \
-v /opt/docker/mysql/conf:/etc/mysql/conf.d \
-v /opt/docker/mysql/logs:/var/log/mysql \
-v /opt/docker/mysql/data:/var/lib/mysql mysql:5.7.24 \
--character-set-server=utf8mb4 --collation-server=utf8mb4_general_ci
```

执行完之后要在 `/opt/docker/mysql/conf/my.cnf` 文件中加入，要不然差8个小时。然后在重启下。

```ini
[mysqld]
default-time-zone=+8:00
-- 显示当前时间
select now()

-- 查看数据库字符集 显示上边配置的就完事了
SHOW VARIABLES LIKE '%character%';
SHOW VARIABLES LIKE 'collation%';
```

###### （二）Redis服务

```shell
#redis缓存 指定密码并且做本地持久化
docker run -p 6379:6379 -v /opt/docker/redis/data:/data -d redis \
redis-server --appendonly yes --requirepass "123456"
```

###### （三）Nginx服务

```shell
#nginx服务
docker run -d --name nginx -p 80:80 \
-v /opt/docker/nginx/logs:/var/log/nginx \
-v /opt/docker/nginx/html:/usr/share/nginx/html \
-v /opt/docker/nginx/conf/nginx.conf:/etc/nginx/nginx.conf \
-v /opt/docker/nginx/conf.d:/etc/nginx/conf.d nginx
```

###### （四）FastDFS服务

因为要做图片上传的操作，还是自己用，就整一个图片服务器好了。上传用Guns管理起来，很方便。

```shell
#FastDFS 要先拉取 qbanxiaoli/fastdfs 这个服务
docker run -d --restart=always --privileged=true --net=host --name=FastDFS -e 
IP=<Your IP Address> -e WEB_PORT=<Your Port> \
-v /opt/docker/FastDFS:/var/local/fdfs qbanxiaoli/fastdfs \
```

###### （五）一些注意的地方

有时候想要在容器里边进行文本的编辑操作，但是`vi` `vim` 都用不了，这个时候

```shell
#同步 /etc/apt/sources.list和 /etc/apt/sources.list.d 中列出的源的索引，这样才能获取到最新的软件包。
apt-get update
#安装
apt-get install vim
```

##### 十三、部署war包

新建一个Dockerfile，把war包和Dockerfile文件放在同级目录下

```shell
FROM tomcat
MAINTAINER "wangao<15804021586@163.com>"
ADD book.war /usr/local/tomcat/webapps/
CMD ["catalina.sh", "run"]
```

执行

```shell
docker build -t book/tomcat .
```

> 相当于把这个war包放进了tomcat的webapp下
>
> -t 后边是构建后镜像的名称
>
> . 后边这个点很重要，代表在当前目录

当然，以后肯定还会有许多用到的命令，用一个在补充一个。。。。