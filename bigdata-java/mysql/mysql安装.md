## mysql安装

[参考1](https://blog.csdn.net/qq_37598011/article/details/93489404)

#### 下载命令

```shell
wget http://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.26-linux-glibc2.12-x86_64.tar


wget http://dev.mysql.com/get/Downloads/MySQL-5.6/mysql-5.6.36-linux-glibc2.5-x86_64.tar.gz
```

#### 解压并移动：

```shell
tar -xvf mysql-5.7.26-linux-glibc2.12-x86_64.tar 

mv mysql-5.7.26-linux-glibc2.12-x86_64 /usr/local/mysql
```

## 创建mysql用户组和用户并修改权限

```bash
groupadd mysql
useradd -r -g mysql mysql
```

创建数据目录并赋予权限

```bash
mkdir -p  /data/mysql              #创建目录
chown mysql:mysql -R /data/mysql   #赋予权限
```

配置my.cnf

```bash
vim /etc/my.cnf

内容如下：
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
explicit_defaults_for_timestamp=true
```

初始化数据库

进入mysql的bin目录

```bash
cd /usr/local/mysql/bin/
初始化
./mysqld --defaults-file=/etc/my.cnf --basedir=/usr/local/mysql/ --datadir=/data/mysql/ --user=mysql --initialize

查看密码
cat /data/mysql/mysql.err

```

安装依赖

yum install libncurses*

## 启动mysql，并更改root 密码

**先将mysql.server放置到/etc/init.d/mysql中**

```
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql
```

启动

```bash
service mysql start

ps -ef|grep mysql
```

![a183ccf5cbdc9f5a513e176c373b2ff](.\pic\a183ccf5cbdc9f5a513e176c373b2ff.png)

SET PASSWORD = PASSWORD('123456');
ALTER USER 'root'@'localhost' PASSWORD EXPIRE NEVER;
FLUSH PRIVILEGES;                                 



![d13ac2dfa8aaa824f72fbcf0bd989ad](.\pic\d13ac2dfa8aaa824f72fbcf0bd989ad.png)

use mysql                                            #访问mysql库
update user set host = '%' where user = 'root';      #使root能再任何host访问
FLUSH PRIVILEGES;                                    #刷新



update mysql.user set authentication_string=password('root_$123') where user='root'; 



## Mysql 5.6安装

mysql 5.6配置用户名和密码：

创建用户和密码
CREATE USER 'bestsellercrm'@'%' IDENTIFIED BY 'crm!@1#23';
授予用户数据库的权限
grant all privileges on bestseller_crm.* to 'bestsellercrm'@'%';

对权限进行刷新
flush privileges;