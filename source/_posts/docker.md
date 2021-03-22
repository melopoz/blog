# Docker

Docker本身是一个容器运行载体（或称为 管理引擎）。我们把应用程序和配置依赖打包成一个可交付的运行环境，这个环境就是image镜像文件。只有通过这个image才能生成Docker容器。

    image文件生成的容器实例，本身也是一个文件。
    一个容器运行一种服务，需要的时候就通过docker客户端创建一个对应的运行实例
    仓库就是存放一堆镜像的地方，可以吧镜像发布到仓库中，需要的时候pull。就像Git
## 三要素
    仓库 镜像image 容器
| docker | java             |
| ------ | ---------------- |
| 仓库   | 包(存放多个镜像) |
| 镜像   | 类               |
| 容器   | 对象             |
## Dockerfile 镜像描述文件

Dockerfile可以解放手工操作，保证环境统一

docker build 命令会根据Dockerfile文件自动构建镜像

> Hello.java    -->  Hello.class
> DockerFile   -->  Docker images

## Dockerfile 基础命令

1. FROM

   ```dockerfile
   # 制作基准镜像
   FROM 镜像
   # 比如我们要发布一个应用到tomcat里，那么的第一步就是FROM tomcat
   FROM tomcat<:tags>
   ```

2. LABEL & MAINTAINER

   ```dockerfile
   # MAINTAINER：维护员，一般写个人id或组织id
   # LABEL 就是注释，纯注释。方便阅读的，纯注释说明。不会对Dockerfile造成任何影响
   MAINTAINER melopoz.com
   LABEL version = "1.0.0"
   LABEL description = "my project"
   ```

3. WORKDIR

   ```dockerfile
   # 类似Linux的cd。不同点是如果没有这个目录，WORKDIR就创建
   WORKDIR /usr/local/mpdir
   ```

4. ADD & COPY

   ```dockerfile
   # COPY 拷贝文件到镜像的某个路径下
   COPY test1.txt /usr/local/mpdir/test
   # 拷贝所有 abc 开头的文件到testdir目录下
   COPY abc* /testdir/
   # ? 是单个字符的占位符，比如匹配文件 abc1.log
   COPY abc?.log /testdir/
   ```

   ```dockerfile
   # 拷贝文件到镜像的某个路径下，如果路径不存在则创建
   ADD test1.txt /usr/local/mpdir/test1
   # 如果ADD的是tar.gz则将解压缩的内容拷贝到镜像的某个路径下
   ADD test1.tar.gz /usr/local/mpdir/test1
   ```

   如果是从远程复制文件，尽量使用curl、wget代替ADD。因为ADD会创建更多镜像层。

5. ENV

   ```dockerfile
   # ENV 设置环境变量，便于下文使用  其他地方都可用${JAVA_HOME}引用
   ENV JAVA_HOME=/usr/local/jdk1.8
   ```

6. RUN & CMD & ENTRYPOINT

   ```dockerfile
   # RUN 在构建镜像时运行，可以修改镜像内部的文件, 有两种命令格式
   RUN ["echo", "image is building..."]
   ```

   > - SHELL 命令格式 生成一个子shell进程去执行脚本，执行完退出子进程，回到父进程
   >
   > ```dockerfile
   > RUN yum -y install vim
   > ```
   >
   > - EXEC 命令格式 用exec进程替换当前进程，保持PID不变，执行完退出，且不会退回原来的进程
   >
   > ```dockerfile
   > RUN ["yum", "-y", "install", "vim"]
   > ```
   >
   > shell会创建子进程执行，EXEC不会创建子进程。**推荐使用EXEC进程**。

   ```dockerfile
   # CMD 在容器启动时执行,如果启动时有其他额外命令，则CMD命令不生效。
   CMD ["echo", "container starting..."]
   ```

   ```dockerfile
   # ENTRYPOINT 在容器创建时执行，且Dockerfile中只有最后一个ENTRYPOINT会被执行
   ENTRYPOINT ["ps", "-ef"]
   ```

   

## docker 服务相关命令
    sudo systemctl status docker//查看docker状态
    sudo systemctl restart docker// 重启docker
    sudo vim /etc/docker/daemon.json// 修改配置文件 需要reload
    sudo systemctl daemon-reload// 重新加载配置文件
## docker 操作命令
##### 镜像操作
```
    docker images //列出本地主机上的镜像
    docker search 镜像名 // 从hub.docker.com查找镜像
        -s 20 镜像名 //筛选点赞数超过20的镜像
        -no-trunc 镜像名 //显示完整描述信息
        -automated 镜像名 //只列出automated build的镜像
    docker pull 镜像名:9.0 //从镜像加速器(如果有配置，没配就是从hub.docker)拉取镜像到本地;版本号不写默认laster
    docker rmi 唯一镜像名(或id) //删除本地镜像 可直接空格连接多个id全部删除
        -f // 强制删除  --force //如果在使用也删除
    docker run 镜像名 //启动一个容器
    
    docker commit //提交容器副本使之成为一个新的镜像
        -m="描述信息" -a="作者" 容器id 要创建的目标镜像名:[标签名]
```
###### 构建镜像

```
docker build * //使用Dockerfile创建镜像；Dockerfile可以是本地、在线、自定义的。
    docker build [options] PATH | URL | - //语法
        --build-arg=[] :设置镜像创建时的变量；
        --cpu-shares :设置 cpu 使用权重；
        --cpu-period :限制 CPU CFS周期；
        --cpu-quota :限制 CPU CFS配额；
        --cpuset-cpus :指定使用的CPU id；
        --cpuset-mems :指定使用的内存 id；
        --disable-content-trust :忽略校验，默认开启；
        -f :指定要使用的Dockerfile路径
        --force-rm :设置镜像过程中删除中间容器；
        --isolation :使用容器隔离技术；
        --label=[] :设置镜像使用的元数据
        -m :设置内存最大值；
        --memory-swap :设置Swap的最大值为内存+swap，"-1"表示不限swap；
        --no-cache :创建镜像的过程不使用缓存；
        --pull :尝试去更新镜像的新版本；
        -q, --quiet :安静模式，成功后只输出镜像 ID；
        --rm :设置镜像成功后删除中间容器；
        --shm-size :设置/dev/shm的大小，默认值是64M；
        --ulimit :Ulimit配置。
        --t, --tag: 镜像的名字及标签，通常 name:tag 或者 name 格式；可以在一次构建中为一个镜像设置多个标签。
        --network: 默认 default。在构建期间设置RUN指令的网络模式
```

例如

```
当前目录下有Dockerfile和hims-0.1.jar
创建镜像：
    docker build -t hims:0.1 .
查看镜像：
    docker images
    结果：
    hims   0.1   ...id...     5 seconds ago    713MB
运行镜像：
    docker run -d 0
```

##### 启动容器 start

```
    docker start 镜像id(或容器名)
```
##### 新建并启动容器 run
```
docker run [options] IMAGE [command][arg]
    --name="容器新名字"// 为容器指定一个名称
    -d //后台运行容器，返回容器id，即启动守护式容器
    -i //以交互模式运行容器，通常与-t同时使用
    -t //为容器重新分配一个伪输入终端
    -P //随机端口映射
    -p //指定端口映射有四种格式 一般使用 主机端口:docker容器端口
        ip:hostPort:containerPort
        ip::containerPort
        hostPort:containerPort
        containerPort
```
##### 运行时命令
```
    docker ps// 查看正在运行的容器   --no-trunc 不截断输出,显示完整containerID
        -a //列出当前所有正在运行的容器 + 历史上运行过的
        -l //显示最近创建的容器
        -n 3 //显示最近创建的3个容器
        -q //静默模式，只显示容器编号(可用于批量关闭容器)
        --no-trunc // 不截断输出
```
##### 停止容器
```
    docker stop 容器id(或容器名)
```
##### 强制停止容器
```
    docker kill 容器id(或容器名)
```
##### 删除已停止容器
```
    docker rm -f 容器id
    docker rm 容器id(或容器名)
        -f // 关闭并删除
```
##### 删除多个容器
```
    docker rm -f $(docker ps -a -q) 或者 docker ps -a -q | xargs docker rm
```
##### 退出容器
```
    exit // 容器停止退出
    ctrl+P+Q // 不停止容器退出
```
##### 重新进入正在运行的容器
```
    docker attach 容器id  // 直接进入容器启动命令的终端，不会启动新进程
```
##### 在容器内执行命令
```
    docker exec -t 容器id 命令 // 在容器中打开新终端，并且可以启动新进程
        比如sentos容器：docker exec -t centosiID ls -l /tmp  //就会返回该容器的tmp目录
```
##### 查看日志
    docker logs 容器id
        -t // 显示日志打印的时间
        -f // =tail -f 
        -tail 3 // 看三行  可以和-f 一起用
##### 查看容器信息 json格式
    docker inspect 容器id
##### 复制容器内文件
    (比如centos容器)
    docker cp centosID:/容器内文件路径 /目的路径
## Docker数据卷
### 容器内添加
##### 1. 直接命令添加  -v
```
    docker run -it -v /宿主机绝对路径:/容器内路径 镜像名 //双方都可读写并同步
    // 可以通过【docker inspect 容器id】查看到VolumesRW.containerDir为true
```
```
    docker run -it -v /hostDir:/containerDir:ro 镜像名 // 只有host可执行写操作，容器只能读
    // 可以通过【docker inspect 容器id】查看到VolumesRW.containerDir为false
```
##### 2. DockerFile添加
```
    docker的dockerfile就像linux的脚本文件。
    dockerfile不能创建host的本地目录来实现数据卷，只能创建一个/多个容器内数据卷。
```
创建dockerfile（touch /mydocker/Dockerfile）
```
FROM centos
VOLUME ["/dataVolumeContainer1","/dataVolumeContainer2"]
CMD echo "finished,--------success1"
CMD /bin/bash
```
创建dockerfile   单独例2
```
FORM java:8 //依赖java8
EXPOSE 8080 //暴露8080端口

VOLUME /tmp //

ENV TZ=Asia/Shanghai
RUN ln -sf /usr/share/zoneinfo/{TZ} /etc/localtime && echo "{TZ}" > /etc/timezone

ADD hims-0.0.1-SNAPSHOT.jar /app.jar //将项目jar包添加到容器并设置别名为app.jar
RUN bash -c 'touch /app.jar'
ENTRYPOINT ["java","-jar","/app.jar"] // 启动命令为 java -jar app.jar
```
构建镜像
```
docker build -f /mydocker/Dockerfile -t melopoz/centos
```
创建容器并运行
```
docker run -it melopoz/centos /bin/bash （/bin/bash可以不写）
```
虽然没有指定宿主机(运行docker的服务器)上与数据卷绑定的路径，但是可以通过```docker inspect 容器id```查看默认生成的宿主机数据卷路径

Volumes：{dataVolumeContainer1:/var/lib/docker/..., dataVolumeContainer2:/var/...}
##### 3. 备注

如果Docker挂载主机目录Dokcer访问出现cannot open directory.:Permission denied

就在挂载目录后加参数```--privileged=true```

即```docker run -it -v /myDir:/containerDir --privileged=true 镜像名```



## 镜像分层 概念

以tomcat镜像为例

```dockerfile
FROM tomcat:latest
MAINTAINER melopoz.com
WORKDIR /usr/local/tomcat/webapps
ADD helloworld ./helloworld
```

Dockerfile内容4行，执行过程也是4行：

```
Sending build context to Docker daemon 3.584kB
Step 1/4 : FROM tomcat:latest
 ---> aaaaaaa
Step 2/4 : MAINTAINER melopoz.com
 ---> Running in bbbbbbb
Removing intermediate container bbbbbbb
Step 3/4 : WORKDIR /usr/local/tomcat/webapps
 ---> Running in ccccccc
Step 4/4 : ADD helloworld ./helloworld
 ---> ddddddd
Successfully build ddddddd
Successfully tagged melopoz.com/test-helloworld:1.0.0
```

将Dockerfile中的内容改为

```dockerfile
FROM tomcat:latest
MAINTAINER melopoz.com
WORKDIR /usr/local/tomcat/webapps
ADD helloworld ./helloworld
ADD helloworld ./helloworld2 #多部署一份helloworld到容器的helloworld2目录
```

再次build之后输出为：

```
Step 1/4 : FROM tomcat:latest
 ---> aaaaaaa
Step 2/4 : MAINTAINER melopoz.com
 ---> Using cache
 ---> Running in bbbbbbb
Step 3/4 : WORKDIR /usr/local/tomcat/webapps
 ---> Using cache
 ---> Running in ccccccc
Step 4/4 : ADD helloworld ./helloworld
 ---> Using cache
 ---> ddddddd
Step 4/4 : ADD helloworld ./helloworld
 ---> eeeeeee
Successfully build eeeeeee
Successfully tagged melopoz.com/test-helloworld:1.0.0
```

前四步没有重复创建容器而是使用了Cache，step1没有UsingCache是由于从本地仓库直接拉取了tomcat:latest镜像作为基础镜像。



# demo

## Docker   mysql

### 查找 拉取镜像
```docker pull mysql:5.7（版本自定义或者不写默认laster)```
### 启动容器
```
docker run -p 3306:3306 --name:mysql_docker
-v .../container_cfgfiles/mysql/conf:/etc/mysql/conf.d
-v .../container_cfgfiles/mysql/logs:/logs
-v .../container_cfgfiles/mysql/conf:/var/lib/mysql
-e MYSQL_ROOT_PASSWORD=123456
-d mysql:5.7
// 指定配置文件 容器名 后台运行 。。
```
### 进入容器
```
docker exec -it 容器id /bin/bash
mysql -uroot -p
输入密码
```
### 数据备份
在docker宿主机终端
```
docker exec mysql容器id sh -C '
exec mysqldump --all-databases -uroot -p"123456"
' > /data/docker_mysql_data/all-databases.sql
```


## Docker   redis
### 启动 这是是在创建容器的时候就启动了redis-server，也可以之后使用exec启动
```
docker run -p 6379:6379
-v .../container_cfgfiles/redis/data:/data
-v .../container_cfgfiles/redis/redis.conf:/usr/local/etc/redis/redis.conf
-d redis:版本号 redis-server /usr/local/etc/redis/redis.conf
--appendonly yes
```
开启了redis AOF
###### 连接redis-cli
```
docker exec -it redis容器id redis-cli
```


# Docker image 提交到阿里云仓库
##### 用实例容器生成最新镜像
```
// docker commit -a 作者 -m "提交信息" 正在运行的容器id 镜像名:版本号
docker commit -a melopoz -m "centos1.3 to 1.4 with vim,ifconfig" xxxx mycentos:1.4
```
##### 查看生成的镜像

```docker images mycentos```

##### 推送到云仓库

###### 阿里云-容器镜像服务
1. 创建镜像仓库(需要命名空间)
2. 阿里云给了推送命令 复制即可
```
//将镜像推送到Registry
sudo docker login --username=xxx registry.cn-beijing.aliyuncs.com
// 输入密码
sudo docker tag [ImageId] registry.cn-beijing.aliyuncs.com/命名空间/仓库名:[镜像版本号]
sudo docker push registry.cn-beijing.aliyuncs.com/xx/xx:[镜像版本号]
//根据实际镜像信息替换示例中的[ImageId]和[镜像版本号]参数。
```
3. 从阿里云拉取镜像
```
sudo docker pull registry.cn-beijing.aliyuncs.com/命名空间/仓库名:[镜像版本号]
```

# 踩坑

## 多个仓库引用一个镜像时 要删除镜像

先使用镜像名删除



---
---
---
---
---
---


## 记第一次启动tomcat，启动成功，访问404

启动成功没有页面可能是tomcat下没有页面，需要进入tomcat下的webapps
```
docker exec -it 容器id /bin/bash
root@f37547416a87:/usr/local/tomcat# ls -l // 查看目录 
root@f37547416a87:/usr/local/tomcat# cd webapps // 为空。。而webapps.dist不为空
    把dist复制到webapps，再访问即可
webapps.dist下的内容：ROOT  docs  examples  host-manager  manager
```