### 

关于docker的安装，[请参考](docker入门.md)

### docker信息的查看

```shell
#查看docker的版本信息
docker version
#查看docker容器的信息
docker info
#帮助
docker help
```

### docker操作镜像

#### 搜索镜像

```shell
docker search java
```

![](.\pic\ea5631d807e0b996a072b778479d33c.png)

#### 下载镜像

```shell
#下载java 8版本的镜像
docker pull java:8
#下载nginx 1.17版本的镜像
docker pull nginx:1.17.0
```

可以在官网查找镜像支持的版本，[地址](https://hub.docker.com/)

#### 删除镜像

```shell
#列出镜像
docker  images
#指定名称删除镜像
docker rmi java:8
#指定名称删除镜像（强制）
docker rmi -f java:8
#强制删除所有镜像
docker rmi -f $(docker images)
```

### Docker操作容器

#### 新建并启动容器

```shell
docker run -p 80:80 --name nginx -d nginx:1.17.0
```

- -d选项：表示后台运行

- --name选项：指定运行后容器的名字为nginx,之后可以通过名字来操作容器

- -p选项：指定端口映射，格式为：hostPort:containerPort 

#### 列出容器

```shell
#列出运行中的容器
docker ps
#列出所有容器
docker ps -a
```

#### 停止容器

```shell
# $ContainerName及$ContainerId可以用docker ps命令查询出来

docker stop $ContainerName(或者$ContainerId)
#比如如下停止命令

docker stop nginx
#或者
docker stop cc043710bf0b
#强制停止容器
docker kill $ContainerName(或者$ContainerId)
#启动已经停止的容器
docker start $ContainerName(或者$ContainerId)
```

#### 进入容器

```shell
#先查询出容器的pid：
docker inspect --format "{{.State.Pid}}" $ContainerName (或者$ContainerId)

#根据容器的pid进入容器：

nsenter --target "$pid" --mount --uts --ipc --net --pid
```

![9ce78c7fce757d6b93e6db2ea3f0681](.\pic\9ce78c7fce757d6b93e6db2ea3f0681.png)

#### 删除容器

```shell
#删除指定容器
docker rm $ContainerName(或者$ContainerId)
#强制删除所有容器
docker rm -f $(docker ps -a -q)
```

#### 其他操作

```shell
#查看容器日志
docker logs $ContainerName(或者$ContainerId)
#查看容器ip地址
docker inspect --format '{{ .NetworkSettings.IPAddress }}' $ContainerName(或者$ContainerId)
#同步宿主机时间到容器
docker cp /etc/localtime $ContainerName(或者$ContainerId):/etc/
#在宿主机查看docker使用cpu、内存、网络、io情况
#查看指定容器的情况
docker stats $ContainerName(或者$ContainerId)
#查看所有容器的情况
docker stats -a
#进入docker容器内部的bash
docker exec -it $ContainerName /bin/bash
```

![ea5631d807e0b996a072b778479d33c](.\pic\ea5631d807e0b996a072b778479d33c.png)



#### 修改docker镜像存放的位置

```shell
#查看Docker镜像的存放位置
docker info | grep "Docker Root Dir"
#关闭docker服务
systemctl stop docker
#移动目录到目标路径
mv /var/lib/docker /mydata/docker
#建立软连接到目标路径
ln -s /mydata/docker /var/lib/docker
```























































































































