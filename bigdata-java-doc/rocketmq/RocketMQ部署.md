RocketMQ部署

rocketmq安装需要jre运行环境，所以需要在linux系统中安装，安装步骤不再重复，网上安装教程一大堆。本rocketmq安装的版本为4.9.2，[下载地址](https://rocketmq.apache.org/dowloading/releases/)

## 单Master模式部署

这种方式风险较大，一旦Broker重启或者宕机时，会导致整个服务不可用。

1、在linux opt目录下创建RocketMq文件夹，并将rocketmq-all-4.9.2-bin-release.zip上传到该文件夹下，然后执行下面命令进行解压

![1644462285](.\pic\1644462285.png)

```shell
unzip rocketmq-all-4.9.2-bin-release.zip
```

2、启动NameServer和Broker

进入到rocketmq-4.9.2执行如下命令启动NameServer

```shell
nohup sh bin/mqnamesrv &
```

运行会出现如下错误，这是因为内存不足导致

```shell
[root@VM-0-11-centos rocketmq-4.9.2]# nohup sh bin/mqnamesrv &
[1] 3805513
[root@VM-0-11-centos rocketmq-4.9.2]# nohup: ignoring input and appending output to 'nohup.out'
^C
[1]+  Exit 1                  nohup sh bin/mqnamesrv
[root@VM-0-11-centos rocketmq-4.9.2]# jps
3805567 Jps
[root@VM-0-11-centos rocketmq-4.9.2]# ll
total 68
drwxr-xr-x 2 root root  4096 Oct 22 13:56 benchmark
drwxr-xr-x 3 root root  4096 Oct 22 13:41 bin
drwxr-xr-x 6 root root  4096 Oct 22 13:41 conf
-rw-r--r-- 1 root root 15126 Feb 10 11:06 hs_err_pid3805532.log
drwxr-xr-x 2 root root  4096 Oct 22 13:56 lib
-rw-r--r-- 1 root root 17327 Oct 22 13:41 LICENSE
-rw------- 1 root root   734 Feb 10 11:06 nohup.out
-rw-r--r-- 1 root root  1338 Oct 22 13:41 NOTICE
-rw-r--r-- 1 root root  5342 Oct 22 13:41 README.md
[root@VM-0-11-centos rocketmq-4.9.2]# cat nohup.out 
Java HotSpot(TM) 64-Bit Server VM warning: Using the DefNew young collector with the CMS collector is deprecated and will likely be removed in a future release
Java HotSpot(TM) 64-Bit Server VM warning: UseCMSCompactAtFullCollection is deprecated and will likely be removed in a future release.
Java HotSpot(TM) 64-Bit Server VM warning: INFO: os::commit_memory(0x00000006c0000000, 2147483648, 0) failed; error='Cannot allocate memory' (errno=12)
#
# There is insufficient memory for the Java Runtime Environment to continue.
# Native memory allocation (mmap) failed to map 2147483648 bytes for committing reserved memory.
# An error report file with more information is saved as:
# /opt/RocketMq/rocketmq-4.9.2/hs_err_pid3805532.log
[root@VM-0-11-centos rocketmq-4.9.2]# top
```

需要更改参数，调整bin/ruunserver.sh如下

即将4g更改为256m,2g更改为128m

```shell
    #JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
     JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```

重新执行启动命令，即可启动成功，如下图

![1644463648](.\pic\1644463648.png)

同理修改bin/runbroker.sh中的8g更改为256m

```shell
#JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g"
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m"
```

启动broker

```shell
nohup sh bin/mqbroker -n localhost:9876 &
```

如下图表示启动成功

![1644464123](.\pic\1644464123.png)

3、发送与接受消息

执行如下命令进行发送和接受消息

```shell
 export NAMESRV_ADDR=localhost:9876
 sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
 sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
```

4、终止NameServer和Broker

```shell
sh bin/mqshutdown broker
sh bin/mqshutdown namesrv
```