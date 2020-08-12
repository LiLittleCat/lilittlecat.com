---
title: CentOS 7 上安装 MySQL 5.7
excerpt: 本文介绍了在 CentOS 7 上安装 MySQL 5.7 的过程，作为笔记方便以后查看。
tags:
  - MySQL
  - CentOS
categories:
  - - 环境搭建
index_img: 'https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/blog/mysql-logo.svg'
banner_img: 'https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/blog/mysql-logo.svg'
abbrlink: f35f8af2
date: 2020-08-12 18:38:18
---


## 开始安装

卸载自带 mariadb 和已安装的 MySQL，如遇到依赖问题导致卸载出错则加上 `--nodeps` 参数：

```sh
# 停止服务
systemctl stop mysqld mariadb
systemctl disable mysqld mariadb
# 卸载已安装服务
rpm -qa | grep mysql | xargs rpm -e
rpm -qa | grep mariadb | xargs rpm -e --nodeps
# 删除相关文件
rm -rvf /etc/my.cnf /var/lib/mysql /var/log/mysqld.log
# 删除用户，非必须
userdel -rf mysql
```

## 下载安装 MySQL

### 网络畅通推荐使用 YUM

下载 Repository：

```sh
wget -c http://repo.mysql.com/mysql57-community-release-el7-10.noarch.rpm #  -c 断点续传
```

安装 Repository：

```sh
yum -y install mysql57-community-release-el7-10.noarch.rpm
```

安装服务器：

```sh
yum -y install mysql-community-server
```

### 无网络先在有网络的地方下载安装包

下载地址：https://downloads.mysql.com/archives/community/，选择对应版本下载。

> 推荐下载 mysql-5.7.30-1.el7.x86_64.rpm-bundle.tar

解压下载的 tar 包，按如下顺序安装：

```sh
rpm -ivh mysql-community-common-5.7.30-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.30-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-5.7.30-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-5.7.30-1.el7.x86_64.rpm
```

## 启动服务

启动服务：

```sh
systemctl start mysqld
```

设置开机启动：

```sh
systemctl enable mysqld
systemctl daemon-reload
```

## 修改登录密码

查看安装完之后 root 用户的临时密码：

```sh
grep "temporary password" /var/log/mysqld.log | awk -F " " '{print $11}' | awk 'END{print}'
```

登录 MySQL，第一次登录需要修改密码之后才能操作：

```sh
mysql -u root -p
```

输入临时密码登录，然后修改用户密码：

```mysql
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '你的密码';
```

> 注意：
>
> MySQL 内置默认密码策略较严格，如需使用较简单的密码，先修改复杂密码之后方可修改密码策略。
>
> 查看密码策略：
>
> ```mysql
> mysql> SHOW VARIABLES LIKE "%password%";
> ```
>
> 修改密码长度，最短为 4，设置低于 4 仍然为 4：
>
> ```mysql
> mysql> SET GLOBAL validate_password_length=4;
> ```
>
> 修改密码强度：
>
> ```mysql
> mysql> SET GLOBAL validate_password_policy=0;
> ```
>
> 再通过上述 `ALTER USER` 修改较简单的密码。
>
> 此时为临时修改，永久修改需要修改 MySQL 配置文件 /etc/my.conf。
>
> 将以下内容添加到 /etc/my.conf 中：
>
> ```sh
> validate_password_policy=0
> validate_password_length=4
> ```
>
> 重启服务即可：
>
> ```sh
> systemctl restart mysqld 
> ```

刷新权限：

```mysql
mysql> FLUSH PRIVILEGES;
```

允许远程登录：

```mysql
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '你的密码' WITH GRANT OPTION;
```

退出：

```mysql
mysql> exit
```

## 开放 3306 端口或者关闭防火墙

开放 3306 端口：

```sh
firewall-cmd --permanent --add-port=3306/tcp
```

> 查看防火墙规则：
>
> ```sh
> firewall-cmd --list-all
> ```

关闭防火墙：

```sh
systemctl stop firewalld
```

取消防火墙开机自启：

```sh
systemctl disable firewalld
systemctl daemon-reload
```

## 配置 MySQL 默认编码

配置 MySQL 默认编码为 utf8，修改 MySQL 配置文件 /etc/my.conf。

将以下内容添加到 /etc/my.conf 中：

```sh
character_set_server=utf8
init_connect='SET NAMES utf8'
```

重启服务即可：

```sh
systemctl restart mysqld 
```

登录 MySQL 查看是否修改成功：

```mysql
mysql> show variables like '%character%';
```

## 安装完成

完成上述步骤后就完成了 CentOS7 下 MySQL 5.7 Server 的安装，可使用 Navicat 等数据库管理软件连接。