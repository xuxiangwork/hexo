---
title: centos7 安装 kong 详解
date: 2017-08-11 06:49:09
tags:
---
# 依赖  
1. gcc  
2. pcre  
3. zlib  
4. openssl  
5. postgresql9.4+  

## gcc 安装  
安装 gcc 编译环境：  
```C
sudo yum install -y gcc gcc-c++
```  

## pcre 安装  
pcre(Perl Compatible Regular Expressions) 是一个 Perl 库，包括 perl 兼容的正则表达式，nginx 的 http 库使用 pcre 解析正则表达式。
```C
sudo yum install -y pcre pcre-devel
```  

## zlib 安装  
zlib 库提供多种压缩和加压缩的方式。
```C
sudo yum install -y zlib zlib-devel
```  

## openssl 安装  
openssl 是一个请打的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及 SSL 协议。  
```C
sudo yum install -y openssl openssl-devel
```  

## postgresql 安装  
PostgreSQL是完全由社区驱动的开源项目，由全世界超过1000名贡献者所维护。它提供了单个完整功能的版本。可靠性是PostgreSQL的最高优先级。Kong 默认使用 postgresql 作为数据库。  
```C
// 添加 rpm
sudo yum install -y https://download.postgresql.org/pub/repos/yum/9.5/redhat/rhel-7-x86_64/pgdg-centos95-9.5-2.noarch.rpm
// 安装 postgresql 9.5
sudo  yum install -y postgresql95-server postgresql95-contrib
// 初始化数据库
sudo /usr/pgsql-9.5/bin/postgresql95-setup initdb
```  
![initdb](https://github.com/xuxiangwork/Sharing/blob/master/picture/Kong%20Install/initdb.png)  
```C
// 设置成 centos7 开机启动服务
sudo systemctl enable postgresql-9.5.service
// 启动 postgresql 服务
sudo systemctl start postgresql-9.5.service
// 查看 postgresql 状态
suso systemctl status postgresql-9.5.service
```  
![postgresql-status](https://github.com/xuxiangwork/Sharing/blob/master/picture/Kong%20Install/postgresql-status.png)  

## 配置 Postgresql  
执行完初始化任务之后，postgresql 会自动创建和生成两个用户和一个数据库： 
```C 
> linux 系统用户 postgres：管理数据库的系统用户；  
> postgresql 用户 postgres：数据库超级管理员；  
> 数据库 postgres：用户 postgres 的默认数据库。  
```  
密码由于是默认生成的，需要在系统中修改一下。  
```C
sudo passwd postgres
```  
![update-passwd](https://github.com/xuxiangwork/Sharing/blob/master/picture/Kong%20Install/update-passwd.png)  
为了安全以及满足 Kong 初始化的需求，需要在建立一个 postgre 用户 kong 和对应的 linux 用户 kong，并新建数据库 kong。  

```PHP
// 新建 linux kong 用户 
sudo adduser kong

// 使用管理员账号登录 psql 创建用户和数据库
// 切换 postgres 用户
// 切换 postgres 用户后，提示符变成 `-bash-4.2$` 
su postgres

// 进入 psql 控制台
psql

// 此时会进入到控制台（系统提示符变为'postgres=#'）
// 先为管理员用户postgres修改密码
\password postgres

// 建立新的数据库用户（和之前建立的系统用户要重名）
create user kong with password '123456';

// 为新用户建立数据库
create database kong owner kong;

// 把新建的数据库权限赋予 kong
grant all privileges on database kong to kong;

// 退出控制台
\q
```  
**在 psql 控制台下执行命令，一定记得在命令后添加分号**。  
![create-psql-user](https://github.com/xuxiangwork/Sharing/blob/master/picture/Kong%20Install/create-psql-user.png)  
登录命令为：  
`psql -U kong -d kong -h 127.0.0.1 -p 5432`  
在 work 或者 root 账户下登录 postgresql 数据库会提示权限问题。

认证权限配置文件为 `/var/lib/pgsql/9.5/data/pg_hba.conf`  
常见的四种身份验证为：  
> **trust**：凡是连接到服务器的，都是可信任的。只需要提供psql用户名，可以没有对应的操作系统同名用户；  
> **password** 和 **md5**：对于外部访问，需要提供 psql 用户名和密码。对于本地连接，提供 psql 用户名密码之外，还需要有操作系统访问权。（用操作系统同名用户验证）password 和 md5 的区别就是外部访问时传输的密码是否用 md5 加密；  
> **ident**：对于外部访问，从 ident 服务器获得客户端操作系统用户名，然后把操作系统作为数据库用户名进行登录对于本地连接，实际上使用了peer；  
> **peer**：通过客户端操作系统内核来获取当前系统登录的用户名，并作为psql用户名进行登录。  

psql 用户必须有同名的操作系统用户名。并且必须以与 psql 同名用户登录 linux 才可以登录 psql 。想用其他用户（例如 root ）登录 psql，修改本地认证方式为 trust 或者 password 即可。  
```C
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
host    all             all             0.0.0.0/0               trust
```  

pgsql 默认只能通过本地访问，需要开启远程访问。  
修改配置文件 `var/lib/pgsql/9.5/data/postgresql.conf` ，将 `listen_address` 设置为 '*'。  
```C
#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# - Connection Settings -

listen_addresses = '*' 			# what IP address(es) to listen on;
```  
执行 `sudo systemctl restart postgresql-9.5.service` 重启 postgresql。  

# kong 安装  
在 kong 安装这一块，费了很大神，参照官方的安装方法[https://getkong.org/install/centos/](kong install)最初始终不成功，提示 `Error unpacking rpm package kong-0.10.3-1.noarch`。最后不得已在 kong 发布的版本库[kong releases](https://github.com/Mashape/kong/releases)中选择了 kong-0.10.2.el7.noarch.rpm 进行安装。  
但后来更换机器的情况下得以正常安装官网步骤安装。  
```C
sudo yum install epel-release
sudo yum install kong-0.10.3.*.noarch.rpm --nogpgcheck
```  
如果出现如下信息表示安装成功：  
![kong-install](https://github.com/xuxiangwork/Sharing/blob/master/picture/Kong%20Install/kong-install.png)  

下面修改 kong 的配置文件，默认配置文件位于 `/etc/kong/kong.conf.default` 
```C
sudo cp /etc/kong/kong.conf.default /etc/kong/kong.conf
```  
将之前安装配置好的 postgresql 信息填入 kong 配置文件中：  
`sudo vi /etc/kong/kong.conf`  
```C
#------------------------------------------------------------------------------
# DATASTORE
#------------------------------------------------------------------------------

# Kong will store all of its data (such as APIs, consumers and plugins) in
# either Cassandra or PostgreSQL.
#
# All Kong nodes belonging to the same cluster must connect themselves to the
# same database.

database = postgres    	         # Determines which of PostgreSQL or Cassandra
                                 # this node will use as its datastore.
                                 # Accepted values are `postgres` and
                                 # `cassandra`.

pg_host = 127.0.0.1             # The PostgreSQL host to connect to.
pg_port = 5432                  # The port to connect to.
pg_user = kong                  # The username to authenticate if required.
pg_password = 123456            # The password to authenticate if required.
pg_database = kong              # The database name to connect to.

ssl = off                       # 如果不希望开放 8443 的 ssl 访问可关闭
```  
默认情况下，kong 并没有添加到 $PATH 环境变量中，所以直接 kong start 并不能生效，利用 `whereis kong` 查看 kong 的命令路径：
![whereis-kong](https://github.com/xuxiangwork/Sharing/blob/master/picture/Kong%20Install/which-kong.png)  
执行 `/usr/local/bin/kong start` 启动空，此时报如下问题：  
![start-fail](https://github.com/xuxiangwork/Sharing/blob/master/picture/Kong%20Install/start-fail.png)  
通过命令 `export KONG_SERF_PATH="/usr/local/bin/serf"` 经 serf 暴露给 kong 以顺利启动。  
![export-serf](https://github.com/xuxiangwork/Sharing/blob/master/picture/Kong%20Install/export-serf.png)  
至此，再次启动 kong  
![kong-start](https://github.com/xuxiangwork/Sharing/blob/master/picture/Kong%20Install/kong-start.png)  
如果启动过程中，碰到提示 **FATAL: Ident authentication failed for user "kong"** 的问题，请确认在配置 postgresql 的认证文件时，采用了 password 或者 trust 连接方式。  

---
转载整理自  
[1]: [centos 6.5 安装kong](http://www.infocool.net/kb/WWW/201707/388221.html)  
[2]: [psql 认证失败](http://blog.csdn.net/sanbingyutuoniao123/article/details/52209653)  
[3]: [kong 安装配置](http://blog.100dos.com/2016/07/25/the-installation-and-configuration-of-kong/)  
[4]: [postgresql 9.5 安装](http://www.jianshu.com/p/24207d55a122)  
[5]: [kong rpm 安装错误](https://github.com/Mashape/kong/issues/1893)  
[6]: [kong 启动错误](https://github.com/Mashape/kong/issues/2217)