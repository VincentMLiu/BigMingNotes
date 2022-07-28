[toc]

# 什么是Docker
[Docker教程](https://vuepress.mirror.docker-practice.com/image/build/#run-%E6%89%A7%E8%A1%8C%E5%91%BD%E4%BB%A4)


## Docker相对虚拟机优势
* 高效利用系统资源 ---- 不需要硬件虚拟，直接运行在系统内核
* 启动速度快 ---- 通过宿主内核启动
* 一致的运行环境 ---- 可保持开发、测试、生产环境一致
* 持续交付和部署 ---- 一次创建或配置，任意地方运行

## 基本概念

![[基本概念.png]]
### 镜像 Image
Linux**内核**启动后，会挂在`root` 文件系统为其提供 **用户空间** 。 而 **Image** 就相当于一个 `root` 文件系统。
**Image** 是一个特殊的文件系统，提供`container` 运行时所需的程序、库、资源、配置和参数（如匿名卷、环境变量、用户等）**不包含**任何动态数据，其内容构建后也不会改变。

**分层存储**


### 容器 Container
`Image` 的实例就是 `Container` 。 
容器的实质是**进程**
容器可以被创建、启动、停止、删除、暂停
**容器存储层** 为容器运行时读写而准备的存储层，以镜像为基础层。容器消亡时，容器存储层也随之消亡。因此，任何保存于容器存储层的信息都会随容器删除而丢失。

PS：，容器不应该向其存储层内写入任何数据，容器存储层要保持无状态化。所有的文件写入操作，都应该使用 [数据卷（Volume）](https://vuepress.mirror.docker-practice.com/data_management/volume.html)、或者 [绑定宿主目录](https://vuepress.mirror.docker-practice.com/data_management/bind-mounts.html)，


### Docker Registry
集中的存储、分发镜像的服务
一个 **Docker Registry** 中可以包含多个 **仓库**（`Repository`）；每个仓库可以包含多个 **标签**（`Tag`）；每个标签对应一个镜像。
Tag相当于软件的各个版本
通过 `<仓库名>:<标签>` 的格式来指定具体是这个软件哪个版本的镜像。如果不给出标签，将以 `latest` 作为默认标签。

 **Registry 公开服务**
 官方的 [Docker Hub](https://hub.docker.com/) 这也是默认的 Registry；
 还有 Red Hat 的 [Quay.io](https://quay.io/repository/) 
 Google 的 [Google Container Registry](https://cloud.google.com/container-registry/)，[Kubernetes](https://kubernetes.io/)  的镜像使用的就是这个服务；
 代码托管平台 [GitHub](https://github.com) 推出的 [ghcr.io](https://docs.github.com/cn/packages/working-with-a-github-packages-registry/working-with-the-container-registry) 。

**私有Docker Registry**
除公开服务外，可以搭建离线的私有仓库。


# 使用Image
## 获取镜像
命令 ： `docker pull`
```
docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```

-   Docker 镜像仓库地址：地址的格式一般是 `<域名/IP>[:端口号]`。默认地址是 Docker Hub(`docker.io`)。
-   仓库名：如之前所说，这里的仓库名是两段式名称，即 `<用户名>/<软件名>`。对于 Docker Hub，如果不给出用户名，则默认为 `library`，也就是官方镜像。

_如果从 Docker Hub 下载镜像非常缓慢，可以参照 [镜像加速器](https://vuepress.mirror.docker-practice.com/install/mirror.html) 一节配置加速器。_


### 运行
`docker run` 就是运行容器的命令，具体参考[[#操作容器]]


## 列出镜像
```
docker image ls
```
列表包含了 `仓库名`、`标签`、`镜像 ID`、`创建时间` 以及 `所占用的空间`。


`docker system df` 命令来便捷的查看镜像、容器、数据卷所占用的空间。
**虚悬镜像(dangling image)**
由于新旧镜像同名，旧镜像名称被取消，从而出现仓库名、标签均为 `<none>` 的镜像。
查看
```
docker image ls -f dangling=true
```
删除
```
docker image prune
```

**中间层镜像**
为了加速镜像构建、重复利用资源，Docker 会利用 **中间层镜像**。

### 列出部分镜像
```
docker image ls <repository_name>
```

`docker image ls` 还支持强大的过滤器参数 `--filter`，或者简写 `-f`

例： 查看`mongo:3.2` 之后建立的镜像，可以用下面的命令：

```
docker image ls -f since=mongo:3.2
```
`since` 也可以换成 `before` 。

如果镜像构建时，定义了 `LABEL`，还可以通过 `LABEL` 来过滤。
```
docker image ls -f label=com.example.version=0.1
```

### 以特定格式显示

只展示镜像的ID列
```
docker image ls -q
```

自定义展示列可以采用[Go 的模板语法](https://gohugo.io/templates/introduction/) 。
例：
只包含镜像ID和仓库名：
```
docker image ls --format "{{.ID}}: {{.Repository}}"
```
以表格等距显示，并且有标题行，和默认一样，不过自己定义列：
```
docker image ls --format "table {{.ID}}\t{{.Repository}}\t{{.Tag}}"
```

## 删除本地镜像
```
docker image rm [选项] <镜像1> [<镜像2> ...]
```
`<镜像>` 可以是 `镜像短 ID`、`镜像长 ID`、`镜像名` 或者 `镜像摘要`。
### 用 docker image ls 命令来配合
例
删除所有仓库名为 `redis` 的镜像：
```
docker image rm $(docker image ls -q redis)
```
删除所有在 `mongo:3.2` 之前的镜像：
```
docker image rm $(docker image ls -q -f before=mongo:3.2)
```


## 构建镜像
### commit命令
**构建镜像**
```
docker commit [选项] <容器ID或容器名> [<仓库名>[:<标签>]]
```
例：
将容器保存为镜像
```
docker commit \
    --author "Tao Wang <twang2218@gmail.com>" \
    --message "修改了默认网页" \
    webserver \
    nginx:v2

```

**查看镜像内的具体历史记录**

```
docker history <仓库名>[:<标签>]
```

查看commit后容器的历史改动
`docker diff <容器名>`

#### 慎用 `docker commit`
使用 `docker commit` 命令虽然可以比较直观的帮助理解镜像分层存储的概念，但是实际环境中并不会这样使用。

1. 仅仅是最简单的操作，就会有大量的无关内容被添加进来，将会导致镜像极为臃肿。
2. `docker commit`的所有操作都是黑箱操作，生成的镜像也成为**黑箱镜像**
3. 任何修改都不会影响到上一层，每次还得重新执行



### Dockerfile定制镜像
镜像的定制实际上就是定制每一层所添加的配置、文件。
Dockerfile 含了一条条的 **指令(Instruction)**，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

#### FROM 指定基础镜像
一个 `Dockerfile` 中 `FROM` 是必备的指令，并且必须是第一条指令。
常用的服务镜像：
[`nginx`](https://hub.docker.com/_/nginx/) 、[`redis`](https://hub.docker.com/_/redis/) 、[`mongo`](https://hub.docker.com/_/mongo/) 、[`mysql`](https://hub.docker.com/_/mysql/) 、[`httpd`](https://hub.docker.com/_/httpd/) 、[`php`](https://hub.docker.com/_/php/) 、[`tomcat`](https://hub.docker.com/_/tomcat/)[](https://hub.docker.com/_/tomcat/)

语言镜像：
[`node`](https://hub.docker.com/_/node) 、[`openjdk`](https://hub.docker.com/_/openjdk/) 、[`python`](https://hub.docker.com/_/python/) 、[`ruby`](https://hub.docker.com/_/ruby/)、[`golang`](https://hub.docker.com/_/golang/) 

操作系统镜像：
[`ubuntu`](https://hub.docker.com/_/ubuntu/)、[`debian`](https://hub.docker.com/_/debian/) 、[`centos`](https://hub.docker.com/_/centos/) 、[`fedora`](https://hub.docker.com/_/fedora/) 、[`alpine`](https://hub.docker.com/_/alpine/)

特殊镜像，空白镜像：
```
FROM scratch
```

#### RUN 执行命令
`RUN` 指令格式有两种
_shell_ 格式：`RUN <命令>`
```
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

_exec_ 格式：`RUN ["可执行文件", "参数1", "参数2"]`

**注意**：_每执行一次`RUN` 命令就会创建一层镜像，需要执行多个命令的时候，要用`&&`符号将多条命令连接起来_
例：
```
FROM debian:stretch

RUN set -x; buildDeps='gcc libc6-dev make wget' \
    && apt-get update \
    && apt-get install -y $buildDeps \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz" \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    && rm -rf /var/lib/apt/lists/* \
    && rm redis.tar.gz \
    && rm -r /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
```


Dockerfile 支持 Shell 类的行尾添加 `\` 的命令换行方式，以及行首 `#` 进行注释的格式
这一组命令的最后添加了清理工作的命令，删除了为了编译构建所需要的软件，清理了所有下载、展开的文件，并且还清理了 `apt` 缓存文件。这是很重要的一步，我们之前说过，镜像是多层存储，每一层的东西并不会在下一层被删除，会一直跟随着镜像。因此镜像构建时，一定要确保每一层只添加真正需要添加的东西，任何无关的东西都应该清理掉。

```
docker build [选项] <上下文路径/URL/->
```

#### 镜像构建上下文（Context）
Docker 在运行时分为 **Docker 引擎**（也就是服务端守护进程）和**客户端**工具。Docker 的引擎提供了一组 REST API，被称为 [Docker Remote API](https://docs.docker.com/develop/sdk/) ，而如 `docker` 命令这样的客户端工具，则是通过这组 API 与 Docker 引擎交互，从而完成各种功能。

Docker引擎的操作目录全部都基于 **上下文（context）** 目录， `COPY` 这类指令中的源文件的路径都是_相对路径_

#### 其它 `docker build` 的用法
##### 直接用 Git repo 进行构建
```
docker build -t hello-world https://github.com/docker-library/hello-world.git#master:amd64/hello-world
```

##### 用给定的 tar 压缩包构建
```
docker build http://server/context.tar.gz
```

##### 从标准输入中读取 Dockerfile 进行构建
```
docker build - < Dockerfile
```

##### 从标准输入中读取上下文压缩包进行构建
```
docker build - < context.tar.gz
```

# Dockerfile 指令详解
## COPY 复制文件
-   `COPY [--chown=<user>:<group>] <源路径>... <目标路径>`
-   `COPY [--chown=<user>:<group>] ["<源路径1>",... "<目标路径>"]`

`COPY` 指令将从构建上下文目录中 `<源路径>` 的文件/目录复制到新的一层的镜像内的 `<目标路径>` 位置。




# 操作容器
## 启动容器
### 新建并启动
`docker run <image> [-options]`
例
启动一个 bash 终端，允许用户进行交互。
```
docker run -t -i ubuntu:18.04 /bin/bash
root@af8bae53bdd3:/#
```
`-t` 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入
`-i` 则让容器的标准输入保持打开。

**创建容器后台流程：**
-   检查本地是否存在指定的镜像，不存在就从 [registry](https://vuepress.mirror.docker-practice.com/repository/) 下载
-   利用镜像创建并启动一个容器
-   分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
-   从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
-   从地址池配置一个 ip 地址给容器
-   执行用户指定的应用程序
-   执行完毕后容器被终止

### 启动已终止容器
`docker container start` 命令，直接将一个已经终止（`exited`）的容器启动运行

## 后台运行
`-d` 参数指定容器后台运行
**注：** 容器是否会长久运行，是和 `docker run` 指定的命令有关，和 `-d` 参数无关。
使用 `-d` 参数启动后会返回一个唯一的 id

获取容器的输出信息
```
docker container logs [container ID or NAMES]
```

## 终止容器
`docker container stop`
交互型容器可以通过 `exit` 命令或 `Ctrl+d`

终止状态的容器可以用`docker container ls -a` 命令看到

`docker container restart` 命令会将一个运行态的容器终止，然后再重新启动它。


## 进入容器
`-d` 参数时，容器启动后会进入后台。

### `attach` 命令
```
docker run -dit ubuntu
243c32535da7d142fb0e6df616a3c3ada0b8ab417937c853a9e1c251f499f550

$ docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
243c32535da7        ubuntu:latest       "/bin/bash"         18 seconds ago      Up 17 seconds                           nostalgic_hypatia

$ docker attach 243c
root@243c32535da7:/#
```

_注意：_ 如果从这个 stdin 中 exit，会导致容器的停止。

### `exec` 命令
`docker exec` 后边可以跟多个参数
`-i` 参数没有分配伪终端, 但命令仍旧可以执行
_注意：_ 如果从这个 stdin 中 exit，不会导致容器的停止。

## 导出和导入容器
**导出**容器快照到本地文件。
```
docker export 7691a814370e > ubuntu.tar
```

`docker import` 从容器快照文件中再导入为**镜像**，例
```
cat ubuntu.tar | docker import - test/ubuntu:v1.0
```

也可以通过指定 URL 或者某个目录来导入，例

```
docker import http://example.com/exampleimage.tgz example/imagerepo
```

_注：用户既可以使用 `docker load` 来导入镜像存储文件到本地镜像库，也可以使用 `docker import` 来导入一个容器快照到本地镜像库。
这两者的区别在于
`docker import`容器快照文件将丢弃所有的历史记录和元数据信息（即仅保存容器当时的快照状态），从容器快照文件导入时可以重新指定标签等元数据信息。_
`docker load` _镜像存储文件将保存完整记录，体积也要大。_


## 删除容器

`docker container rm` 来删除一个处于终止状态的容器。例
```
docker container rm trusting_newton trusting_newton
```
删除一个运行中的容器，可以添加 `-f` 参数。Docker 会发送 `SIGKILL` 信号给容器。

**清理所有处于终止状态的容器**

```
docker container prune
```


# Docker 仓库





# Docker数据管理
![[types-of-mounts.cd09b2d7.png]]


### 数据卷volume 

`数据卷` 是一个可供一个或多个容器使用的特殊目录，它绕过 UFS （user file system），可以提供很多有用的特性：
-   `数据卷` 可以在容器之间共享和重用
-   对 `数据卷` 的修改会立马生效
-   对 `数据卷` 的更新，不会影响镜像
-   `数据卷` 默认会一直存在，即使容器被删除


#### 创建数据卷
```
docker volume create my-vol
```

list数据卷
```
docker volume ls
```

查看数据卷具体信息
```
docker volume inspect my-vol
```

#### 启动一个挂载数据卷的容器
`docker run` 命令的时候，使用 `--mount` 标记来将 `数据卷` 挂载到容器里。在一次 `docker run` 中可以挂载多个 `数据卷`。
例：
创建一个名为 `web` 的容器，并加载一个 `数据卷` 到容器的 `/usr/share/nginx/html` 目录。

```
docker run -d -P \
    --name web \
    # -v my-vol:/usr/share/nginx/html \
    --mount source=my-vol,target=/usr/share/nginx/html \
    nginx:alpine
```

#### 查看数据卷的具体信息
查看 `web` 容器的信息

```
 docker inspect web
```

`数据卷` 信息在 "Mounts" Key 下面
例
```
"Mounts": [
    {
        "Type": "volume",
        "Name": "my-vol",
        "Source": "/var/lib/docker/volumes/my-vol/_data",
        "Destination": "/usr/share/nginx/html",
        "Driver": "local",
        "Mode": "",
        "RW": true,
        "Propagation": ""
    }
],
```

#### 删除数据卷
```
docker volume rm my-vol
```
Docker 不会在容器被删除后自动删除 `数据卷`

如果需要在删除容器的同时移除数据卷。可以在删除容器的时候使用 `docker rm -v` 这个命令。

无主的数据卷可能会占据很多空间，要清理请使用以下命令
```
docker volume prune
```


### 挂载一个主机目录作为数据卷
`--mount` 标记可以指定挂载一个本地主机的目录到容器中去， 也可以从主机挂载单个文件到容器中

例
加载主机的 `/src/webapp` 目录到容器的 `/usr/share/nginx/html`目录

```
docker run -d -P \
    --name web \
    # -v /src/webapp:/usr/share/nginx/html \
    --mount type=bind,source=/src/webapp,target=/usr/share/nginx/html \
    nginx:alpine
```

Docker 挂载主机目录的默认权限是 `读写`，用户也可以通过增加 `readonly` 指定为 `只读`。
```
docker run -d -P \
    --name web \
    # -v /src/webapp:/usr/share/nginx/html:ro \
    --mount type=bind,source=/src/webapp,target=/usr/share/nginx/html,readonly \
    nginx:alpine
```


# 使用网络
`-P` 标记时，Docker 会随机映射一个端口到内部容器开放的网络端口
`-p` 则可以指定要映射的端口，支持的格式有：
`ip:hostPort:containerPort | ip::containerPort | hostPort:containerPort`

### 外部宿主机访问容器

**映射所有接口地址**
`hostPort:containerPort` 格式本地的 80 端口映射到容器的 80 端口
```
docker run -d -p 80:80 nginx:alpine
```

**映射到指定地址的指定端口**
使用 `ip:hostPort:containerPort` 格式指定映射使用一个特定地址
```
docker run -d -p 127.0.0.1:80:80 nginx:alpine
```

**映射到指定地址的任意端口**

`ip::containerPort` 绑定 localhost 的任意端口到容器的 80 端口，本地主机会自动分配一个端口。
```
docker run -d -p 127.0.0.1::80 nginx:alpine
```

还可以使用 `udp` 标记来指定 `udp` 端口
```
docker run -d -p 127.0.0.1:80:80/udp nginx:alpine
```

**查看映射端口配置**
`docker port` 来查看当前映射的端口配置，也可以查看到绑定的地址

-   容器有自己的内部网络和 ip 地址（使用 `docker inspect` 查看，Docker 还可以有一个可变的网络配置。）
-   `-p` 标记可以多次使用来绑定多个端口
```
docker run -d \
    -p 80:80 \
    -p 443:443 \
    nginx:alpine
```

### 容器互联
#### 新建网络
创建一个新的 Docker 网络
```
docker network create -d bridge my-net
```
`-d` 参数指定 Docker 网络类型，有 `bridge` `overlay`。其中 `overlay` 网络类型用于 [Swarm mode](https://vuepress.mirror.docker-practice.com/swarm_mode/)

#### 连接容器
运行一个容器并连接到新建的 `my-net` 网络
```
docker run -it --rm --name busybox1 --network my-net busybox sh
```

打开新的终端，再运行一个容器并加入到 `my-net` 网络
```
docker run -it --rm --name busybox2 --network my-net busybox sh
```

**查看容器信息**
```
docker container ls
```

#### Docker Compose

如果你有多个容器之间需要互相连接，推荐使用 [Docker Compose](https://vuepress.mirror.docker-practice.com/network/compose)。
笔记
[[Dockers Compose]]