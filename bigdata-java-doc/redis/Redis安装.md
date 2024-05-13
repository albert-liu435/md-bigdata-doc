## Redis安装

1. 源码下载

   ```shell
   #下载地址
   #https://redis.io/download
   #下载
   wget http://download.redis.io/releases/redis-6.2.1.tar.gz
   #解压
   tar -xzvf redis-6.2.1.tar.gz 
   ```

   

2. 安装gcc环境与编译

   由于redis是由C语言编写的，它的运行需要C环境，因此我们需要先安装gcc。安装命令如下：

   ```shell
   [root@VM-0-4-centos redis-6.2.1]# yum install gcc-c++
   #编译与安装
   [root@VM-0-4-centos redis-6.2.1]# make
   
   [root@VM-0-4-centos redis-6.2.1]# cd ./src/
   
   [root@VM-0-4-centos src]# make install
   ```


   为了方便管理，将Redis文件中的conf配置文件和常用命令移动到统一文件中

   ```shell
   [root@VM-0-4-centos redis-6.2.1]# mkdir bin 
   [root@VM-0-4-centos redis-6.2.1]# mkdir etc
   #将redis-6.2.1目录下的 redis.conf 移动到 redis-6.2.1目录下的etc文件夹下
   [root@VM-0-4-centos redis-6.2.1]# mv redis.conf ./etc/
   #将mkreleasehdr.sh、redis-benchmark、redis-check-aof、redis-cli、redis-server 移动到   /usr/local/redis-6.2.1/bin/ 目录下
   [root@VM-0-4-centos redis-6.2.1]# cd ./src/
   [root@VM-0-4-centos src]# mv mkreleasehdr.sh redis-benchmark redis-check-aof redis-cli redis-server ../bin/
   ```

3. 编辑 redis.conf配置文件，设置后台启动redis服务

   ```shell
   #把文件中的daemonize属性改为yes（表明需要在后台运行）
   # By default Redis does not run as a daemon. Use 'yes' if you need it.
   # Note that Redis will write a pid file in /var/run/redis.pid when daemonized.
   # When Redis is supervised by upstart or systemd, this parameter has no impact.
   daemonize yes
   
   #把 redis.conf配置文件中的 bind 127.0.0.1 这一行给注释掉，这里的bind指的是只有指定的网段才能远程访问这个redis，注释掉后，就没有这个限制了。
   # IF YOU ARE SURE YOU WANT YOUR INSTANCE TO LISTEN TO ALL THE INTERFACES
   # JUST COMMENT OUT THE FOLLOWING LINE.
   # ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
   #bind 127.0.0.1 -::1
   #把 redis.conf配置文件中的 protected-mode 设置成no（默认是设置成yes的， 防止了远程访问，在redis3.2.3版本后）
   # By default protected mode is enabled. You should disable it only if
   # you are sure you want clients from other hosts to connect to Redis
   # even if no authentication is configured, nor a specific set of interfaces
   # are explicitly listed using the "bind" directive.
   protected-mode no
   #
   # The requirepass is not compatable with aclfile option and the ACL LOAD
   # command, these will cause requirepass to be ignored.
   #
   # requirepass foobared
   requirepass 123456
   
   #设置Redis开机启动
   [root@localhost etc]# vi /etc/rc.d/rc.local
   #添加如下代码到 /etc/rc.d/rc.local 中：
   /usr/local/redis-6.2.1/bin/redis-server  /usr/local/redis-6.2.1/etc/redis.conf
   #切换到 /usr/local/redis-6.2.1/bin/ 目录下执行 redis-server 命令，使用 /usr/local/redis-6.2.1/etc/redis.conf配置文件来启动redis服务
   #启动Redis服务
   ./redis-server /usr/local/redis-6.2.1/etc/redis.conf   
   ```

   

4. redis下面的如下文件

   redis-server	Redis服务器
   redis-cli	Redis命令行客户端
   redis-benchmark	Redis性能测试工具
   redis-check-aof	AOF文件修复工具
   redis-check-rdb	RDB文件检查工具
   redis-sentinel	Sentinel服务器（哨兵模式）

## Redis卸载

### 查看redis是否已经启动

```shell
ps aux | grep redis
```

停止Redis服务

正常停止redis-server　服务，使用reids 客户端命令: redis-cli shutdown

如果停止不了,则采取杀死进程的方式：kill -9 PID

删除/usr/local/bin　目录下与redis 相关的命令：

ls /usr/local/bin/redis-*

rm -rf /usr/local/bin



删除redis 解压后的目录 redis-* 即可，如：目录 redis-3.2.1 

redis windows安装
https://github.com/MicrosoftArchive/redis/releases