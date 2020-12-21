---
title: "Docker MySQL部署"
date: 2020-10-06T22:43:20+08:00
description: "Article description."
draft: false
toc: true
categories:
  - MySQL
tags:
  - MySQL
  - Docker
---

# 前提
安装docker,mac环境下可直接安装docker Desktop

# 拉取

#拉取5.7版本的mysql镜像
docker push mysql:5.7

# 运行
```
docker run -p 13306:3306 \
--name d-mysql-57 \
-e MYSQL_ROOT_PASSWORD=Mo20100528 \
-v /Users/zyg/softs/docker/mysql57/data:/var/lib/mysql \
-v /Users/zyg/softs/docker/mysql57/logs:/var/log/mysql \
-v /Users/zyg/softs/docker/mysql57/conf/my.cnf:/etc/my.cnf \
-d mysql:5.7 
```

参数说明:
* run　run 是运行一个容器
* -d　 表示后台运行
* -p　 表示容器内部端口和服务器端口映射关联
* --privileged=true　设值MySQL 的root用户权限, 否则外部不能使用root用户登陆
* -v 容器内的路径(如/etc/mysql)挂载到宿主机
* -e MYSQL_ROOT_PASSWORD=xxx 设置MySQL数据库root用户的密码
* --name 设值容器名称为mysql
mysql:5.7 表示从docker镜像mysql:5.7中启动一个容器
* --character-set-server=utf8mb4 
* --collation-server=utf8mb4_general_ci 设值数据库默认编码

停止上面启动的容器,容器名字为"d-mysql-57"
```
docker stop d-mysql-57
```

# 配置账户

##进入容器
```
docker exec -it d-mysql-57 bash
```

##登录MySQL
```
``mysql -uroot -p`
```

##创建用户,名叫test,密码是test123,开启远程访问权限
```
GRANT ALL PRIVILEGES ON *.* TO 'test'@'%' IDENTIFIED BY 'test123' WITH GRANT OPTION;
```

##创建数据库,名叫xxx
```
create database xxx;
```


之后便可以通过该用户执行业务脚本

