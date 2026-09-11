#  Docker容器

## 关于docker你需要知道的几件事



## 安装docker

### 原生系统安装（推荐）

```
1.删除旧版本

2.yum下载

yum install -y yum-utils

3.设置镜像仓库

yum-config-manager \

 	--add-repo \

​	镜像源地址（阿里等都提供）

4.更新yum软件包索引

yum  makecache fast

5.安装docker相关 docker-ce社区   ee 企业版

yu install docker-ce docker-ce-cli containerd.io

6.启动docker

systemmctl start docker

#确认版本

docker version 

7.hello-world
docker run hello-world

8.查看docker images
root@LAPTOP-VS6NS6RA:/mnt/c/Users/a# docker images
                                                                                                    i Info →   U  In Use
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
hello-world:latest   5dd0d3e6e255       25.9kB         9.49kB    U

卸载docker
#1.卸载依赖
yum  remmove docker-ce docker-ce-cli containerd.io
#2.删除资源
rm -rf /var/lib/docker
具体位置以实际工作目录为准
```

### WSL安装



## docker结构

### docker - repository - image  

> run 的运行流程图

```text
开始
  │
  ▼
Docker 在本机寻找镜像
  │
  ├── 本地有这个镜像 ────────────────────► 使用这个镜像运行
  │
  └── 本地没有这个镜像
        │
        ▼
  去 Docker Hub 上下载
        │
        ├── DockerHub 可以找到 ──► 下载这个镜像到本地 ──► 使用这个镜像运行
        │
        └── DockerHub 找不到 ───────────────► 返回错误，找不到镜像
```





### docker底层原理

#### docker工作流程

docker是一个client-server结构的系统，Docker的守护进程运行在主机，通过Socket从客户端访问

Docker-sever 收到Docker-clent的命令就会执行：

```text
  ┌─────────────────────────────────────────────┐
  │              Linux 服务器 / 宿主机             │
  │  ┌───────────────────────────────────────┐  │
  │  │         Docker 后台守护进程            │  │
  │  │  (dockerd，监听 /var/run/docker.sock)  │  │
  │  └──────────────┬────────────────────────┘  │
  │                 │                           │
  │     ┌───────────┴────────────┐             │
  │     ▼                        ▼             │
  │  ┌──────────────┐      ┌──────────────┐     │
  │  │  docker 容器  │      │  docker 容器  │     │
  │  │localhost:8080│      │localhost:3306│     │
  │  └──────────────┘      └──────────────┘     │
  └─────────────────────────────────────────────┘
           ▲                         ▲
           │                         │
    ┌──────┴──────┐           ┌──────┴──────┐
    │    客户端    │           │    客户端    │
    │  docker CLI  │           │  docker CLI  │
    └─────────────┘           └─────────────┘
```

#### 从底层分析Vm和docker性能差异

Docker有更少的抽象层

```text
        VM (Virtual Machine)                      Docker
 ┌────────────────────────────────┐      ┌────────────────────────────────┐
 │  ┌──────┐      ┌──────┐        │      │  ┌──────┐      ┌──────┐        │
 │  │ App A│      │ App B│        │      │  │ App A│      │ App B│        │
 │  ├──────┤      ├──────┤        │      │  ├──────┤      ├──────┤        │
 │  │ Bins/│      │ Bins/│        │      │  │ Bins/│      │ Bins/│        │
 │  │ Libs │      │ Libs │        │      │  │ Libs │      │ Libs │        │
 │  ├──────┤      ├──────┤        │      │  ├──────┤      ├──────┤        │
 │  │Guest │      │Guest │        │      │  │      │      │      │        │
 │  │ OS   │      │ OS   │        │      │  │      │      │      │        │
 │  └──────┘      └──────┘        │      │  └──────┘      └──────┘        │
 │       Hypervisor               │      │     Docker Engine              │
 ├────────────────────────────────┤      ├────────────────────────────────┤
 │           Host OS              │      │           Host OS              │
 ├────────────────────────────────┤      ├────────────────────────────────┤
 │           Server               │      │           Server               │
 └────────────────────────────────┘      └────────────────────────────────┘
```

> 容器不需要完整的 Guest OS，**直接在 Docker Engine 上运行**，调用宿主机内核。

容器并不需要建立在虚拟层上，而是在docker引擎上的

因此docker直接调用宿主机cpu，可以达到秒级响应速度



## docker命令

### 帮助命令

```
docker version #显示docker的版本信息

docker info #显示docker系统信息，包括镜像和容器数量

docker commmand --help #显示帮助命令
更多可查询docker doc（国内菜鸟教程也提供）
```

### 镜像命令

docker images 查看本机所有镜像

```
root@LAPTOP-VS6NS6RA:/mnt/c/Users/a# docker images
                                                                                                    i Info →   U  In Use
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
hello-world:latest   5dd0d3e6e255       25.9kB         9.49kB    U
#WSL有所不同

#参数解释
REPOSITORY  镜像仓库源
TAG			镜像标签
INAMGE ID	镜像id
CREATED		镜像创建时间
SIZE		镜像的源码大小

 -a, --all             Show all images (default hides intermediate and dangling images)
      --digests         Show digests
  -f, --filter filter   Filter output based on conditions provided
      --format string   Format output using a custom template:
                        'table':            Print output in table format with column headers (default)
                        'table TEMPLATE':   Print output in table format using the given Go template
                        'json':             Print in JSON format
                        'TEMPLATE':         Print output using the given Go template.
                        Refer to https://docs.docker.com/go/formatting/ for more information about formatting
                        output with templates
      --no-trunc        Don't truncate output
  -q, --quiet           Only show image IDs
      --tree            List multi-platform images as a tree (EXPERIMENTAL)
```



docker search  镜像搜索

```
docker  search imagename
Search Docker Hub for images

Options:
  -f, --filter filter   Filter output based on conditions provided
      --format string   Pretty-print search using a Go template
      --limit int       Max number of search results
      --no-trunc        Don't truncate output
```

docker pull

```
docker pull imagename #建议指定版本，不指定默认最新版本
Usage:  docker pull [OPTIONS] NAME[:TAG|@DIGEST]

Download an image from a registry

Aliases:
  docker image pull, docker pull

Options:
  -a, --all-tags          Download all tagged images in the repository
      --platform string   Set platform if server is multi-platform capable
  -q, --quiet             Suppress verbose output
```

```
docker pull tomcat
Using default tag: latest
#不同版本容器可同时存在
latest: Pulling from library/tomcat  #镜像分层下载，相同文件不会重复下载
4f4fb700ef54: Pull complete
0926a8eb0e60: Pull complete
89416f64c8e9: Pull complete
fbdaa7f21d48: Pull complete
b385a54fa58d: Pull complete
297337c1dc13: Pull complete
6bd4a5f61a04: Pull complete
da76df643e43: Download complete
d477108507e1: Download complete
Digest: sha256:b4237a8551b327a88d67c156a943e2db14eb2bcbaa85cbfebd587e0cab9c8885 #签名
Status: Downloaded newer image for tomcat:latest
docker.io/library/tomcat:latest #真实地址
```

docker rmi 删除镜像

```
docker rmi -f dockerid				 #删除指定id容器
docker rmi -f  dockerid dockerid 	 #删除多个容器
docker rmi -f $(docker images -aq)	 #删除全部容器
```



### 容器命令

值得注意的是，只有在有镜像情况下才能创建容器

#### 新建容器|启动容器

```bin/
docker run [可选参数]  image

#常用参数说明

--name="Name"	#容器命名，用于区分容器

-d							 #后台方式运行

-it							 #使用交互式运行，进入容器查看内容

-p							 #指定容器端口  -p8080:8080

 		-p  ip:主机端口:容器端口
		-p  主机端口:容器端口（常用）
		-p  容器端口

-P							#随机指定端口

#测试，进入容器
docker  run -it centos bin/bash
#查看到内部并不完整，很多命令不可用
exit #退出容器
```

#### 列出所有的运行容器

```
docker ps
	 #列出所有的运行容器
-a   #列出所有的运行容器+带出1历史运行过的容器
-n=? #显示最近创建的容器
-q	 #只显示容器编号
```

#### 退出容器

```
exit  #退出容器并终止
Ctrl+P+Q#退出容器，后台运行
```



#### 删除容器

```
docker rm dockerid 				#删除指定非运行容器,rm -f 强制删除
docker rm -f $(docker  ps -q	#删除所有容器
docker -a -q |  xargs docker rm	#删除所有容器
```

#### 启动容器|停止容器

```
docker start  dockerid	#启动容器
docker restart dockerid	#重启容器
docker stop dockerid    #停止当前运行容器
docker kill dockerid	#强制停止容器
```

#### 常用其他命令

```
docker  run -d imagename
#docker ps  会发现挂掉了
#docker后台运行，需要有前台进程，docker未发现前台应用会自动静止
#nginx 容器启动后，发现自己没提供服务会立刻静止，就是没有程序了
```

#### 查看日志

```
docker logs -f -t --tail #容器，没有日志

-tf				#显示日志
--tail nummmber #显示指定数量日志1

#自己编写一段shell脚本以保证shell存活
"while true;do echo hi;sleep 10;done"
#tomcat为例
docker run -it --rm -p 8080:8080 tomcat

root@LAPTOP-VS6NS6RA:/mnt/c/Users/a# docker run -d --name my-tomcat -p 8080:8080 tomcat
5276d8a8049f104631c8b048fa12b64a88a9f88545e7ee379f91cfec512fa4d8


root@LAPTOP-VS6NS6RA:/mnt/c/Users/a# docker logs -f my-tomcat
09-Sep-2026 06:24:19.848 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server version name:   Apache Tomcat/11.0.25
09-Sep-2026 06:24:19.851 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server built:          Aug 12 2026 12:46:04 UTC
09-Sep-2026 06:24:19.851 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server version number: 11.0.25.0
09-Sep-2026 06:24:19.851 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log OS Name:               Linux
09-Sep-2026 06:24:19.851 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log OS Version:            6.18.33.2-microsoft-standard-WSL2
09-Sep-2026 06:24:19.851 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Architecture:          amd64
09-Sep-2026 06:24:19.851 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Java Home:             /opt/java/openjdk
09-Sep-2026 06:24:19.851 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log JVM Version:           25.0.4+7-LTS
09-Sep-2026 06:24:19.851 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log JVM Vendor:            Eclipse Adoptium
09-Sep-2026 06:24:19.852 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log CATALINA_BASE:         /usr/local/tomcat
09-Sep-2026 06:24:19.852 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log CATALINA_HOME:         /usr/local/tomcat
09-Sep-2026 06:24:19.861 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Djava.util.logging.config.file=/usr/local/tomcat/conf/logging.properties
09-Sep-2026 06:24:19.861 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager
09-Sep-2026 06:24:19.861 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Djdk.tls.ephemeralDHKeySize=2048
09-Sep-2026 06:24:19.861 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Dorg.apache.catalina.security.SecurityListener.UMASK=0027
09-Sep-2026 06:24:19.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: --add-opens=java.base/java.lang=ALL-UNNAMED
09-Sep-2026 06:24:19.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: --add-opens=java.base/java.lang.reflect=ALL-UNNAMED
09-Sep-2026 06:24:19.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: --add-opens=java.base/java.io=ALL-UNNAMED
09-Sep-2026 06:24:19.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: --add-opens=java.base/java.util=ALL-UNNAMED
09-Sep-2026 06:24:19.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: --add-opens=java.base/java.util.concurrent=ALL-UNNAMED
09-Sep-2026 06:24:19.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: --add-opens=java.rmi/sun.rmi.transport=ALL-UNNAMED
09-Sep-2026 06:24:19.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: --enable-native-access=ALL-UNNAMED
09-Sep-2026 06:24:19.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Dcatalina.base=/usr/local/tomcat
09-Sep-2026 06:24:19.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Dcatalina.home=/usr/local/tomcat
09-Sep-2026 06:24:19.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Djava.io.tmpdir=/usr/local/tomcat/temp
09-Sep-2026 06:24:19.872 INFO [main] org.apache.catalina.core.AprLifecycleListener.lifecycleEvent Loaded Apache Tomcat Native library [2.0.15] using APR version [1.7.2].
09-Sep-2026 06:24:19.876 INFO [main] org.apache.catalina.core.AprLifecycleListener.initializeSSL OpenSSL successfully initialized [OpenSSL 3.0.13 30 Jan 2024]
09-Sep-2026 06:24:20.084 INFO [main] org.apache.coyote.AbstractProtocol.init Initializing ProtocolHandler ["http-nio-8080"]
09-Sep-2026 06:24:20.101 INFO [main] org.apache.catalina.startup.Catalina.load Server initialization in [421] milliseconds
09-Sep-2026 06:24:20.132 INFO [main] org.apache.catalina.core.StandardService.startInternal Starting service [Catalina]
09-Sep-2026 06:24:20.133 INFO [main] org.apache.catalina.core.StandardEngine.startInternal Starting Servlet engine: [Apache Tomcat/11.0.25]
09-Sep-2026 06:24:20.140 INFO [main] org.apache.coyote.AbstractProtocol.start Starting ProtocolHandler ["http-nio-8080"]
09-Sep-2026 06:24:20.147 INFO [main] org.apache.catalina.startup.Catalina.start Server startup in [44] milliseconds



```

#### 查看进程信息

```
#命令 docker top dockerid


^Croot@LAPTOP-VS6NS6RA:/mnt/c/Users/a# docker ps
CONTAINER ID   IMAGE     COMMAND             CREATED         STATUS         PORTS                                         NAMES
5276d8a8049f   tomcat    "catalina.sh run"   6 minutes ago   Up 6 minutes   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp   my-tomcat
root@LAPTOP-VS6NS6RA:/mnt/c/Users/a# docker top 5276d8a8049f
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                421                 396                 0                   06:24               ?                   00:00:01            /opt/java/openjdk/bin/java -Djava.util.logging.config.file=/usr/local/tomcat/conf/logging.properties -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djdk.tls.ephemeralDHKeySize=2048 -Dorg.apache.catalina.security.SecurityListener.UMASK=0027 --add-opens=java.base/java.lang=ALL-UNNAMED --add-opens=java.base/java.lang.reflect=ALL-UNNAMED --add-opens=java.base/java.io=ALL-UNNAMED --add-opens=java.base/java.util=ALL-UNNAMED --add-opens=java.base/java.util.concurrent=ALL-UNNAMED --add-opens=java.rmi/sun.rmi.transport=ALL-UNNAMED --enable-native-access=ALL-UNNAMED -classpath /usr/local/tomcat/bin/bootstrap.jar:/usr/local/tomcat/bin/tomcat-juli.jar -Dcatalina.base=/usr/local/tomcat -Dcatalina.home=/usr/local/tomcat -Djava.io.tmpdir=/usr/local/tomcat/temp org.apache.catalina.startup.Bootstrap start
root@LAPTOP-VS6NS6RA:/mnt/c/Users/a#
```

#### 查看镜像元数据

```
docker inspect dockerid
#大量相关数据，可以用于排查，这里不便列出
```

#### 进入当前容器

```
docker exec -it dockerid bashshell
#在容器内新建一个 Shell 进程
docker atttach dockerid
#连接到容器现有的主进程
```

想进容器内部操作，**永远优先使用 `docker exec -it`**，它更安全，不会误停容器。想看实时日志，用 `docker logs -f` 更稳妥。除非特殊需求，尽量少用 `attach`

#### 从容器拷贝文件到本机

```
docker cp ddockerid fileposition aimroute
#拷贝是手动过程，未来使用-v卷可以实现
```

### 小结

> Docker 命令全景图：围绕 Image、Container、Registry、Engine 四大对象，以及 Tar 包 / Dockerfile / Host 文件的中转。

```text
 Images                          Container lifecycle
 ┌─────────┐                     ┌────────────────────────────┐
 │ images  │                     │  create ──► run ──► start  │
 │ rmi     │                     │     │           ▲    │    │
 │ tag     │                     │     ▼           │    ▼    │
 │ history │                     │  ┌─────┐   kill/stop  Stop │
 │ build   │◄──── Dockerfile     │  │Running│◄─────────────┤  │
 │ commit  │────►                │  └──┬──┘   pause/unpause │
 │ load    │◄──── Tar files      │     │           ▼    ▲    │
 │ save    │────►                │  ┌─────┐   Pause ──────┘   │
 └────┬────┘                     └────────────────────────────┘
      │                              │
      │ pull / push                  │ cp / logs / inspect / attach
      │                              │ port / ps / top / rm / exec / wait
      ▼                              ▼
 ┌─────────┐                     ┌─────────────┐
 │Registry │                     │  Host files  │
 │ search  │                     │  / folders   │
 │ login   │                     └─────────────┘
 │ logout  │
 └─────────┘

 Engine 通用：docker version / docker info / docker events
```

>   docker命令相当之多，想要熟练还是需要多用

#### nginx示例

```
 docker search nginx
 docker pull nginx
 ocker images
                                                                                                    i Info →   U  In Use
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
hello-world:latest   5dd0d3e6e255       25.9kB         9.49kB    U
nginx:latest         05b8cb60c354        253MB         69.2MB
tomcat:latest        b4237a8551b3        589MB          158MB    U

 docker run -d --name nginx01 -p 3344:80 nginx
0798b6d1c5175d5ca0d136fdc1a9ed6abe7a5d74b27fa887b8eeb6a502376a8e
root@LAPTOP-VS6NS6RA:/mnt/c/Users/a# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                                         NAMES
0798b6d1c517   nginx     "/docker-entrypoint.…"   11 seconds ago   Up 10 seconds   0.0.0.0:3344->80/tcp, [::]:3344->80/tcp       nginx01
5276d8a8049f   tomcat    "catalina.sh run"        27 minutes ago   Up 27 minutes   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp   my-tomcat
root@LAPTOP-VS6NS6RA:/mnt/c/Users/a# curl localhost:3344
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>


#docker exec -it nginxid bash 修改配置但是否有其他管理方法呢？显然是有的
```



## Docker镜像原理

### 镜像是什么

镜像是一种轻量级、可执行的独立软件包，用来打包软件运行环境和基于运行环境开发的软件，它包含运行某个软件所需的所有内容，包括代码、运行时、库、环境变量和配置文件。

所有的应用，直接打包 docker 镜像，就可以直接跑起来！

如何得到镜像：

- 从远程仓库下载
- 朋友拷贝给你
- 自己制作一个镜像 DockerFile

### Docker 镜像加载原理

#### UnionFS（联合文件系统）

UnionFS（联合文件系统）：Union 文件系统（UnionFS）是一种分层、轻量级并且高性能的文件系统，它支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件系统下 (unite several directories into a single virtual filesystem)。Union 文件系统是 Docker 镜像的基础。镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。

特性：一次同时加载多个文件系统，但从外面看起来，只能看到一个文件系统，联合加载会把各层文件系统叠加起来，这样最终的文件系统会包含所有底层的文件和目录

#### Docker 镜像加载原理

docker 的镜像实际上由一层一层的文件系统组成，这种层级的文件系统 UnionFS。

bootfs (boot file system) 主要包含 bootloader 和 kernel, bootloader 主要是引导加载 kernel, Linux 刚启动时会加载 bootfs 文件系统，在 Docker 镜像的最底层是 bootfs。这一层与我们典型的 Linux/Unix 系统是一样的，包含 boot 加载器和内核。当 boot 加载完成之后整个内核就都在内存中了，此时内存的使用权已由 bootfs 转交给内核，此时系统也会卸载 bootfs。

rootfs (root file system)，在 bootfs 之上。包含的就是典型 Linux 系统中的 /dev, /proc, /bin, /etc 等标准目录和文件。rootfs 就是各种不同的操作系统发行版，比如 Ubuntu，Centos 等等。



### 镜像分层结构

1. **bootfs（引导文件系统）** 镜像最底层，包含`bootloader`引导程序 + Linux 内核`kernel`。

   > 宿主机本身已经有内核，**Docker 容器启动时不会加载 bootfs**，会直接复用宿主机内核，这也是容器比虚拟机轻量的根本原因。bootfs 加载完成后会被卸载。

2. **rootfs（根文件系统）** 在 bootfs 上层，包含 Linux 标准目录：`/bin`、`/etc`、`/dev`、`/proc`等。 就是我们常说的 Ubuntu、CentOS 基础镜像，**只有系统文件，没有内核**。

3. 上层应用层（多层叠加）

   在 rootfs 基础上，每执行一条 Dockerfile 指令（

   ```
   RUN
   ```

   /

   ```
   COPY
   ```

   /

   ```
   ADD
   ```

   等），就会新增

   一个只读层

   。

   - 所有镜像层都是**只读**的；
   - 多个镜像可以**共享底层只读层**，节省磁盘空间。

##### 容器启动：在镜像顶层加一层可读写层

当`docker run`启动容器：

1. Docker 会在镜像所有只读层的最上方，新增一层**容器读写层**；

2. 容器修改文件时，不会改动底层镜像只读层，而是触发

   写时复制（Copy-on-Write）

   ：

   > 需要修改某个文件 → 先把这个文件从下层只读层复制到顶层读写层 → 在读写层修改。底层镜像文件保持原样。

3. 容器删除时，只会删除顶层读写层，底层镜像层依然保留，可以被其他容器复用。

##### 分层的优点

 **资源复用**：多个容器共用同一个基础镜像层，不用重复下载存储。

 **镜像增量更新**：拉取镜像时，只下载本地不存在的层。 

**读写分离**：镜像只读保证镜像不变；容器改动只落在独立读写层，容器销毁改动丢失。

##### 面试一句话总结

Docker 镜像基于 UnionFS 做多层只读堆叠，底层是 bootfs+rootfs；容器启动时在镜像顶部增加一层可读写层，文件修改使用写时复制机制；分层实现镜像层共享，节省存储空间。

### 容器层 vs 镜像层（Docker）

#### 1. 镜像层（Image Layer）

- **属性：全部只读（Read-only）**
- 镜像由**多层只读文件系统**堆叠而成（UnionFS），每一层对应 Dockerfile 的一条指令。
- 层一旦构建完成，**永远不能修改**。如果要改，只能在上面新增一层。
- 作用：保存系统、程序、配置等基础文件，**多个容器可以共享同一套镜像层**，节省磁盘。
- 示例：centos 基础镜像层、安装 jdk 的层、拷贝项目代码的层，全都是只读。

#### 2. 容器层（Container Layer）

- **属性：可读写（Read-write）**

- 执行`docker run`启动容器时，Docker 会在**镜像所有只读层的最顶部，额外新增一层可读写层**，这就是容器层。

- 容器里所有新增、修改、删除文件的操作，**只会发生在这一层**，不会改动底层镜像。

- 核心机制：

  写时复制 Copy-on-Write（CoW）

  - 当容器要修改底层镜像里的某个文件：不会直接改只读层。
  - 先把这个文件从镜像层复制到容器读写层，再在副本上修改。底层镜像文件保持原样。

- 生命周期：**容器删除，容器读写层跟着一起删除；底层镜像层不受影响，依然保留**。容器停止，读写层数据还在；容器 rm，读写层数据丢失。

#### 3. 对比总结表

表格

| 项目     | 镜像层                   | 容器层                           |
| -------- | ------------------------ | -------------------------------- |
| 读写权限 | 只读                     | 可读写                           |
| 数量     | 多层堆叠（多个只读层）   | **只有 1 层**（最顶层）          |
| 数据修改 | 不可修改                 | 所有增删改都在这里               |
| 共享性   | 多个容器共享             | 容器私有，一个容器对应一个容器层 |
| 删除影响 | 删除容器，镜像层不会被删 | 容器被 rm，容器层数据全部销毁    |

#### 4. 小结

Docker 镜像是一堆**只读层**堆叠；容器启动时在镜像之上增加**一层独立可读写容器层**。容器修改文件使用写时复制，修改只作用于容器层，底层镜像不变；容器删除，容器读写层就被清除。

#### 补充

> 容器持久化：容器层的数据随着容器删除会丢失。如果要持久化数据，需要使用**Volume 数据卷**，把数据挂载到宿主机，脱离容器层保存。

### commit镜像

```
docker commit #提交容器成为一个新的副本

#命令原理类似git
docker commit -m="提交的描述信息" -a="作者" dockerId imagesname:[TAG]
```

实战

```
#启动一个默认tomcat
#将webapp.dist 下内容迁移到webapps
#将对应docker commit为一个镜像，这样修改过的docker就成为了属于你的镜像
```

提示：可以随时提交以保存容器，类似于快照哦





## 容器卷技术

数据都在容器，损坏和删除都会丢失，显然有极大风险！数据持久化是非常重要的一步

容器之间有一个数据共享技术，Docker容器产生的数据同步到本地

这就是卷技术，将容器内目录的挂载到linux上，就可以很好的解决这个问题

**总结：容器持久化和同步操作得到解决，还1实现容器间的数据共享** 



### 数据卷的使用

> 方式一：直接使用命令挂载 -v 

```
docker run -it -v 主机目录：容器内目录
-v 主机目录：容器内目录 -v 主机目录：容器内目录 #挂载多个文件
docker inspect 可查看挂载情况
#请不要随意选定目录挂载，权限不足会写入失败
```

优点：修改主机文件即可，会自动同步到容器

#### 具名挂载|匿名挂载 

```
#匿名挂载
-v 容器内路径
docker run  -d -P --name nginx01 -v /etc/nginx nginx

#查看所有volume的情况
docker volume ls
DRIVER    VOLUME NAME
local     387c98881b3bd4a2408c1e2f1d78b017376c799d4ed5b46522f1370305c2cd55
#这里显然没有指明主机目录，-v只有容器内路径。没有容器外的路径，这就是匿名挂载

#具名挂载（推荐）
-v 卷名:容器内路径

docker run  -d -P --name nginx03 -v test-nginx:/etc/nginx nginx
d1b295e45578596567989cd01f173c400571a3664d074750a0ad70ac60aacab4
root@LAPTOP-VS6NS6RA:/mnt/c/Users/a# docker volume ls
DRIVER    VOLUME NAME
local     387c98881b3bd4a2408c1e2f1d78b017376c799d4ed5b46522f1370305c2cd55
local     test-nginx

#查看挂载的卷，可以看到默认的挂载目录
docker volume inspect test-nginx
[
    {
        "CreatedAt": "2026-09-09T11:03:40Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/test-nginx/_data",
        "Name": "test-nginx",
        "Options": null,
        "Scope": "local"
    }
]
```

拓展

```
# 通过 -v 容器内路径: ro  rw 改变读写权限

ro    readonly  # 只读
rw    readwrite # 可读可写

# 一旦这个了设置了容器权限，容器对我们挂载出来的内容就有限定了！

docker run -d -P --name nginx02 -v juming-nginx:/etc/nginx:ro nginx
docker run -d -P --name nginx02 -v juming-nginx:/etc/nginx:rw nginx

# ro 只要看到 ro 就说明这个路径只能通过宿主机来操作，容器内部是无法操作！
```

#### 初识DockerFile

DockerFile就是用来构建docker镜像的构建文件，属于命令脚本

通过脚本生成镜像，一层层脚本生成不同层

>   方式二DockerFile挂载
>

```
# 创建一个dockerfile文件，名字可以随机 建议 Dockerfile
# 文件中的内容 指令(大写)  参数
FROM centos

VOLUME ["volume01","volume02"]

CMD echo "----end----"

CMD /bin/bash


 docker build -f /mnt/c/Users/a/volumetest/dockerfile -t meme/nginx:1.0 .
[+] Building 0.2s (5/5) FINISHED                                                                         docker:default
 => [internal] load build definition from dockerfile                                                               0.0s
 => => transferring dockerfile: 116B                                                                               0.0s
 => WARN: JSONArgsRecommended: JSON arguments recommended for CMD to prevent unintended behavior related to OS si  0.0s
 => WARN: MultipleInstructionsDisallowed: Multiple CMD instructions should not be used in the same stage because   0.0s
 => [internal] load metadata for docker.io/library/nginx:latest                                                    0.0s
 => [internal] load .dockerignore                                                                                  0.0s
 => => transferring context: 2B                                                                                    0.0s
 => CACHED [1/1] FROM docker.io/library/nginx:latest@sha256:05b8cb60c354a44ab824ea6e7dc69b46d50762cdbe728a347a5b6  0.0s
 => => resolve docker.io/library/nginx:latest@sha256:05b8cb60c354a44ab824ea6e7dc69b46d50762cdbe728a347a5b656e6fb3  0.0s
 => exporting to image                                                                                             0.1s
 => => exporting layers                                                                                            0.0s
 => => exporting manifest sha256:919a12daa41481a9ef4febb93f92629cf302e91cc5e067e4099d2944b7314b9e                  0.0s
 => => exporting config sha256:ab9c9eb182c4a4c3eacf30519cf79198e6b3790f4cc9dabbcb3b1d5172e0f13b                    0.0s
 => => exporting attestation manifest sha256:93fd0a5795519e8d492140d6483d0d5c7f53dfa8a7d856c906918fd22e709371      0.0s
 => => exporting manifest list sha256:4544e852d2b52059b5f0be59808d3517480934acbdf6c1095858544a18b3ccd3             0.0s
 => => naming to docker.io/meme/nginx:1.0                                                                          0.0s
 => => unpacking to docker.io/meme/nginx:1.0                                                                       0.0s

 2 warnings found (use docker --debug to expand):
 - JSONArgsRecommended: JSON arguments recommended for CMD to prevent unintended behavior related to OS signals (line 6)
 - MultipleInstructionsDisallowed: Multiple CMD instructions should not be used in the same stage because only the last one will be used (line 6)
 
 
#运行可以确定完成挂载 （推荐使用docker inspect确认）
docker images
                                                                                                    i Info →   U  In Use
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
hello-world:latest   5dd0d3e6e255       25.9kB         9.49kB    U
meme/nginx:1.0       4544e852d2b5        250MB         66.3MB
nginx:latest         05b8cb60c354        253MB         69.2MB    U
tomcat:latest        b4237a8551b3        589MB          158MB    U
root@LAPTOP-VS6NS6RA:/mnt/c/Users/a/volumetest# docker run -it  4544e852d2b5 /bin/bash
root@32f254cfac31:/# ls
bin   dev                  docker-entrypoint.sh  home  lib64  mnt  proc  run   srv  tmp  var      volume2
boot  docker-entrypoint.d  etc                   lib   media  opt  root  sbin  sys  usr  volume1
```

### 数据容器卷

--volumes-from 就可以实现容器间数据共享

```
docker run --it --name  docker01  --volumes-from docker meme/nginx:1.0
docker run --it --name  docker02  --volumes-from docker01 meme/nginx:1.0
docker run --it --name  docker03  --volumes-from docker01 meme/nginx:1.0
#可以删除 docker01，查看一下 docker02 和 docker03 是否还可以访问这个文件，显然不会。所以很好的实现数据的持久化保存。
```

结论：容器之间配置信息的传递，数据卷容器的生命周期一直持续到没有容器使用为止。

但是一旦你持久化到了本地，这个时候，本地的数据是不会删除的！



## DockerFile

DockerFile是用于构建镜像的命令参数脚本；

**构建步骤：**

1. 编写一个 dockerfile 文件
2. `docker build` 构建成为一个镜像
3. `docker run` 运行镜像
4. `docker push` 发布镜像（DockerHub、阿里云镜像仓库）

### DockerFile构建过程

Dockerfile 就是一份**镜像构建说明书**：从上到下逐行执行，每个保留关键字（指令）都是大写字母，`#` 表示注释。

> 核心逻辑：**一行指令 → 一层只读镜像层 → 逐层叠加 → 最终成一个完整镜像**。
> 和容器层不一样：容器层是**可读写**，镜像层一旦生成就是**只读**，想改只能往上再盖一层。

```text
┌─────────────────────────────────────────────────┐
│  Dockerfile (文本脚本)                            │
│  ─────────────────────────────────────────────  │
│  FROM ubuntu:22.04          ← 第 1 层：基础镜像    │
│  RUN apt-get update          ← 第 2 层：系统依赖     │
│  RUN apt-get install -y jdk  ← 第 3 层：运行环境     │
│  COPY app.jar /app/          ← 第 4 层：应用文件     │
│  EXPOSE 8080                 ← 元数据，不产生新层    │
│  ENTRYPOINT ["java","-jar"]  ← 启动入口            │
│  CMD ["/app/app.jar"]        ← 默认参数            │
└─────────────────────────────────────────────────┘
                    │
                    │ docker build -t myapp:1.0 .
                    ▼
        ┌─────────────────────────────────────┐
        │  Image（多层只读 UnionFS）           │
        │  第 5 层：ENTRYPOINT/CMD 配置        │
        │  第 4 层：COPY app.jar               │
        │  第 3 层：RUN install jdk            │
        │  第 2 层：RUN apt-get update         │
        │  第 1 层：FROM ubuntu:22.04          │
        └─────────────────────────────────────┘
```

#### 构建缓存机制

Docker build 会自动缓存每层结果。如果某层的上下文和指令没变，Docker 会直接复用上次生成的层，**不会重新执行**。

```bash
# 第一次 build：会逐层执行
docker build -t myapp:1.0 .

# 第二次 build：只改代码，COPY 之前的层全部命中缓存
docker build -t myapp:1.0 .
# => [build 1/5] FROM ubuntu:22.04 -- CACHED
# => [build 2/5] RUN apt-get update -- CACHED
# => [build 3/5] RUN apt-get install -y jdk -- CACHED
# => [build 4/5] COPY app.jar /app/ -- 重新执行
# => [build 5/5] CMD ["/app/app.jar"] -- 重新执行
```

> **注意**：只要有一层缓存失效，**它之后的所有层都会重新构建**。所以 Dockerfile 里要把**变化频率最低**的指令放前面（比如 FROM、RUN install 依赖），**变化频率最高**的放后面（比如 COPY 业务代码）。别把 `COPY . /app` 放在 `RUN apt-get install` 前面，否则每次改一行代码都要重新装依赖，慢死。

#### 最佳实践速查

1. **每个 RUN 只做一件事**，多命令合并用 `&&`，减少层数：
   ```dockerfile
   RUN apt-get update && apt-get install -y \
       curl \
       vim \
       && rm -rf /var/lib/apt/lists/*
   ```
2. **用 .dockerignore 排除不需要进构建上下文的文件**，比如 `.git`、`target/`、`node_modules/`，减小 build 上下文。
3. **最小化基础镜像**：生产尽量用 alpine、distroless，而不是完整的 Ubuntu。
4. **不要把密码写进 Dockerfile**，用 build args、secret mount 或运行时环境变量。

#### 面试一句话总结

Dockerfile 从上到下逐条执行，每条可写指令生成一个只读镜像层；镜像由这些层叠加而成，容器启动时在最顶层再挂一个可读写层。build 会缓存未变更的层，所以写 Dockerfile 要把不变动的依赖往前放、业务代码往后放。

### DockerFile的指令解析

```ADD
FROM			#基础镜像，一切构建1基础
MAINTAINER		#镜像信息，name+email
RUN				#镜像构建需要运行的命令		
ADD				#步骤，tomcat镜像，这个tomcat压缩包，是添加内容
WORKDIR			#镜像工作目录
VOLUME			#挂载的目录
EXPOSE			#保留的端口配置
CMD				#指定容器启动时执行的命令，只有最后一条执行，会被替代
ENTRYPOINT		#指定容器启动时执行的命令，可以追加命令
ONBUILD			#当构建一个被继承Dockerfile，这时会运行ONBUILD的指令。触发指令
COPY			#类似ADD，帮助拷贝文件到镜像
ENV				#构建的时候设置环境变量
```

#### CMD|ENTRYPOINT区别

- **CMD**：容器启动时的**默认命令**，`docker run` 后面跟的命令会**直接覆盖**它。
- **ENTRYPOINT**：容器启动时的**固定入口程序**，`docker run` 后面跟的参数会**追加**到它后面，不会被覆盖。

------

##### 对比表

表格

|                        | CMD                                    | ENTRYPOINT                                         |
| ---------------------- | -------------------------------------- | -------------------------------------------------- |
| 作用                   | 定义默认启动命令 / 参数                | 定义固定的可执行入口                               |
| `docker run <img> xxx` | xxx **覆盖**整个 CMD                   | xxx 作为参数**追加**到 ENTRYPOINT 后               |
| 用途                   | 提供默认行为，允许用户替换             | 把容器当成 "命令" 来用（如 `nginx -g daemon off`） |
| Dockerfile 中多条      | 只有最后一条生效                       | 同样只有最后一条生效                               |
| 常见搭配               | 单独使用，或为 ENTRYPOINT 提供默认参数 | 与 CMD 配合：ENTRYPOINT 写程序，CMD 写默认参数     |



##### 示例

###### 1. 只有 CMD

```
FROM ubuntu
CMD ["echo", "hello"]
docker run myimg          # 输出: hello
docker run myimg echo hi  # 输出: hi  ← CMD 被覆盖了！
```

###### 2. 只有 ENTRYPOINT

```
FROM ubuntu
ENTRYPOINT ["echo"]
docker run myimg          # 输出: （空）
docker run myimg hi       # 输出: hi  ← "hi" 被追加到 echo 后
```

###### 3. 黄金组合：ENTRYPOINT + CMD（推荐）

```
FROM ubuntu
ENTRYPOINT ["echo"]
CMD ["hello"]
docker run myimg              # 输出: hello（用 CMD 的默认参数）
docker run myimg world        # 输出: world（CMD 被替换成 world，追加给 echo）
```

> **本质**：ENTRYPOINT 决定 "跑什么程序"，CMD 决定 "不传参时默认给什么参数"。



### 实战测试tomcat服务镜像构建

目标：从零构建一个带自己 war 包的 Tomcat 镜像，运行后访问到自定义页面。

#### 1. 准备目录结构

```bash
~/docker-tomcat
├── Dockerfile
└── myapp.war        # 你要部署的 war 包
```

#### 2. Dockerfile

```dockerfile
FROM tomcat:9.0-jdk8-openjdk

# 维护者信息（官方已标记废弃，面试题里还会考）
MAINTAINER yourname@example.com

# 删除 Tomcat 默认的 ROOT 应用，避免冲突
RUN rm -rf /usr/local/tomcat/webapps/ROOT

# 把自己的 war 包放进去
COPY myapp.war /usr/local/tomcat/webapps/ROOT.war

# 暴露端口
EXPOSE 8080

# 启动 Tomcat
CMD ["catalina.sh", "run"]
```

#### 3. 构建并运行

```bash
# 构建
cd ~/docker-tomcat
docker build -t mytomcat:1.0 .

# 运行
docker run -d -p 8080:8080 --name mytomcat mytomcat:1.0

# 验证
curl http://localhost:8080
```

#### 4. 常见坑

- `webapps/ROOT.war` 与 `webapps/ROOT/` 目录不能同时存在，否则 Tomcat 启动会报错。
- 想让 war 自动解压成 `ROOT/`，可以保留原文件名，不用删目录；想精确控制就用 `ROOT.war`。
- 生产环境建议用官方 `tomcat:9.0-jdk8-temurin` 或 `tomcat:10-jdk17`。

### 实战测试 nginx站点镜像构建

目标：用 Dockerfile 构建一个自定义 Nginx 镜像，首页显示自定义 HTML。

#### 1. 准备目录

```bash
~/docker-nginx
├── Dockerfile
└── index.html
```

`index.html`：

```html
<!DOCTYPE html>
<html>
<head><title>我的 Nginx</title></head>
<body><h1>Hello from Dockerfile!</h1></body>
</html>
```

#### 2. Dockerfile

```dockerfile
FROM nginx:1.27-alpine

# 把自定义首页替换默认首页
COPY index.html /usr/share/nginx/html/index.html

# 把本地 nginx.conf 挂进去（可选，没有就用默认配置）
# COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

# nginx 镜像默认 ENTRYPOINT 已经是 nginx，CMD 是 -g daemon off;
# 这里只需覆盖 CMD 或保持不变
CMD ["nginx", "-g", "daemon off;"]
```

#### 3. 构建并运行

```bash
cd ~/docker-nginx
docker build -t mynginx:1.0 .
docker run -d -p 80:80 --name mynginx mynginx:1.0

# 验证
curl http://localhost
# Hello from Dockerfile!
```

#### 4. 进阶：带反向代理配置

```dockerfile
FROM nginx:1.27-alpine

# 先删除默认配置，避免和自定义配置冲突
RUN rm /etc/nginx/conf.d/default.conf

COPY nginx.conf /etc/nginx/nginx.conf
COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

> **注意**：`nginx:alpine` 默认没有 `bash`，进去排错用 `docker exec mynginx sh`。

### 发布自己的镜像

> DockerHub注册，登录

```
docker push #上传镜像，请尽量写版本号，否则版本混乱难以管理

Usage:  docker push [OPTIONS] NAME[:TAG]

Upload an image to a registry

Aliases:
  docker image push, docker push

Options:
  -a, --all-tags          Push all tags of an image to the repository
      --platform string   Push a platform-specific manifest as a single-platform image to the registry.
                          Image index won't be pushed, meaning that other manifests, including attestations won't
                          be preserved.
                          'os[/arch[/variant]]': Explicit platform (eg. linux/amd64)
  -q, --quiet             Suppress verbose output
```

补充：关于阿里docker仓库，可以查看相关文档，这里不进行介绍。

docler save/load 备份，请--help查询



## Docker网络

何谓网络？

### 核心结论

Docker 安装后会在宿主机创建一个虚拟网桥 **docker0**，每个容器通过一对 **veth pair（虚拟网卡对）** 连接到 docker0，所有容器共享这个二层网络，从而实现通信。

------

### 通信流程

预览

查看代码

```
flowchart TB
    subgraph Host["宿主机"]
        subgraph docker0["docker0 网桥 (172.17.0.1)"]
        end
        subgraph C1["容器1 (172.17.0.2)"]
            eth1["eth0"]
        end
        subgraph C2["容器2 (172.17.0.3)"]
            eth2["eth0"]
        end
        eth1 <-->|"veth pair"| docker0
        eth2 <-->|"veth pair"| docker0
    end
    Internet -->|"iptables NAT"| docker0
```

豆包

你的 AI 助手，助力每日工作学习

------

### 详细步骤

**1. 网络命名空间隔离** 每个容器有自己的独立网络命名空间（netns），拥有独立的网卡、路由表、iptables 规则，互不干扰。

**2. veth pair 连接容器与网桥**

- Docker 为每个容器创建一对虚拟网卡 `vethxxx`
- 一端放进容器内部（命名为 `eth0`）
- 另一端挂到 docker0 网桥上
- 相当于一根 "网线"，一端在容器里，一端插在 docker0 交换机上

**3. 容器间通信（同一宿主机）**

```
容器1 (172.17.0.2) → eth0 → veth → docker0 → 查路由表 → veth → eth0 → 容器2 (172.17.0.3)
```

docker0 像一个二层交换机，自动转发同一网段内的数据包。

**4. 容器访问外网** Docker 通过 iptables 的 **MASQUERADE（NAT）** 规则，将容器源 IP（172.17.x.x）替换为宿主机 IP 后发出：

```
容器 → docker0 → iptables NAT（改源IP为宿主机IP）→ 物理网卡 → 互联网
```

**5. 外网访问容器（端口映射）** 用 `-p 8080:80` 时，Docker 在 iptables 加一条 **DNAT** 规则，把宿主机 8080 端口的流量转发到容器 80 端口。

------

#### veth pair 原理

veth pair（虚拟以太网配对）是一对**虚拟网卡**，从一端发送的数据会原封不动地从另一端出来 —— 就像一根 "虚拟网线"，用来连接两个网络命名空间。

------

##### 本质

```
flowchart LR
    A["netns A 中的 eth0"] <==|"veth pair<br/>一根虚拟网线"| B["netns B 中的 vethxxx"]
    
    style A fill:#85c1e9,stroke:#2980b9,color:#fff
    style B fill:#f5b041,stroke:#d68910,color:#fff
```

- veth pair 总是**成对出现**，一端叫 `eth0`，另一端叫 `vethxxx`
- 数据包从一端进入，**直接从另一端穿出**，不经过任何物理设备
- 它不是设备，而是一条**管道 / 线缆**，连接两个网络命名空间

------

##### 工作原理

**1. 创建一对 veth**

```
ip link add veth0 type veth peer name veth1
```

此时 `veth0` 和 `veth1` 在同一个命名空间（宿主机）里。

**2. 把一端移动到容器命名空间**

```
ip link set veth1 netns <容器PID>
```

- `veth0` 留在宿主机，挂到 docker0 网桥上
- `veth1` 被移进容器，改名为 `eth0`

```
flowchart TB
    subgraph Host["宿主机 netns"]
        docker0["docker0 网桥"]
        v0["veth0 (宿主机端)"]
    end
    subgraph Container["容器 netns"]
        v1["eth0 (原 veth1)"]
    end
    v0 <==> v1
    docker0 --- v0
```

**3. 数据流向**

```
容器内 ping 172.17.0.1
  → 容器 eth0 发出包
  → veth1（容器端）收到
  → 从 veth0（宿主机端）穿出
  → 到达 docker0 网桥
  → docker0 转发给目标
```

反向同理，数据沿原路返回。

------

##### 为什么需要 veth pair？

因为**网络命名空间是隔离的**。普通的虚拟网卡（如 bridge、tap）只能存在于一个命名空间内。要在两个隔离的命名空间之间传数据，就需要 veth pair 这种 "两端分属不同命名空间" 的特殊设备。

> 类比：veth pair 就像一根网线，插头 A 插在容器里，插头 B 插在 docker0 交换机上，数据沿着网线直达。

------

##### 关键特性

表格

| 特性       | 说明                               |
| ---------- | ---------------------------------- |
| 成对存在   | 创建一对，删除一个另一个也自动消失 |
| 双向管道   | 一端进，另一端出，无缓冲           |
| 跨命名空间 | 两端可在不同 netns 中              |
| 无物理对应 | 纯软件模拟，不依赖硬件             |
| 状态联动   | 一端 down，另一端也跟着 down       |

------

##### 实际查看

```
# 查看宿主机上的 veth 设备
ip link show | grep veth

# 查看 veth 两端的对应关系（通过 ifindex）
ip -d link show veth0

# 进入容器查看自己的网卡
docker exec <容器> ip addr show
```

> **总结**：veth pair 是 Docker 网络的 "plumbing（管道工）"，本身不做任何处理，只是把两个网络命名空间的接口连起来，真正的转发和路由由 docker0 网桥和内核协议栈完成。

### 关键命令

```
# 查看 docker0 网桥
ip addr show docker0

# 查看容器 IP
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <容器名>

# 查看 veth pair 对应关系
ip link show

# 容器间互 ping 测试
docker exec 容器1 ping 172.17.0.3
```

------

### 默认桥接网络的局限

表格

| 问题                 | 说明                               |
| -------------------- | ---------------------------------- |
| 容器间只能用 IP 互访 | 默认 docker0 不支持容器名 DNS 解析 |
| 容器重建 IP 会变     | 无法固定 IP，不适合动态场景        |
| 隔离性弱             | 所有默认桥接容器默认互通           |

> **生产环境推荐**：用 `docker network create mynet` 创建**用户自定义桥接网络**，它自带 DNS 解析，容器之间可以直接用**容器名**互相通信，比默认 docker0 更好用。

```
docker network create mynet
docker run -d --name app1 --network mynet nginx
docker run -d --name app2 --network mynet nginx
docker exec app2 ping app1   # 直接用容器名通信！
```

### --link 的使用

`--link` 是 Docker 早期用来让容器之间**通过容器名互相访问**的参数，现在**已废弃**，推荐用用户自定义网桥网络替代。

------

#### 基本用法

```
# 先启动被连接的容器（数据库）
docker run -d --name mysql mysql:5.7

# 再启动应用容器，通过 --link 连接到 mysql
docker run -d --name web --link mysql:mysql nginx
```

格式：`--link <容器名或ID>:<别名>`

- 前半部分：要连接的容器名
- 冒号后：在当前容器中可以用什么名字访问它

------

##### 写入 hosts 文件

1. ：在 web 容器的 

   ```
   /etc/hosts
   ```

    中自动加一条记录

   ```
   172.17.0.3  mysql
   ```

2. 注入环境变量

   ：把被链接容器的 IP、端口等信息以环境变量形式传入

   ```
   MYSQL_PORT_3306_TCP=tcp://172.17.0.3:3306
   MYSQL_PORT_3306_TCP_ADDR=172.17.0.3
   ```

这样 web 容器里就可以直接用 `mysql` 这个主机名访问数据库，不用写死 IP。

------

#### 实际操作示例

```
# 启动数据库容器
docker run -d --name mysql -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7

# 启动应用容器，link 到 mysql
docker run -it --rm --name web --link mysql:mysql centos:7 bash

# 在 web 容器内测试
ping mysql          # 可以 ping 通，自动解析到 mysql 容器 IP
curl mysql:3306     # 可以直接访问
cat /etc/hosts      # 能看到自动添加的 mysql 记录
env | grep MYSQL     # 能看到注入的环境变量
```

### 自定义网络

> docker network ls #显示所有的网络

#### 网络模式

表格

| 模式          | 说明                                              |
| ------------- | ------------------------------------------------- |
| **bridge**    | 桥接 docker（默认，自己的容器也使用 bridge 模式） |
| **none**      | 不配置网络                                        |
| **host**      | 和宿主机共享网络                                  |
| **container** | 容器网络连通！（用的少！局限很大）                |

#### 测试

```
# 我们直接启动的命令 --net bridge，而这个就是我们的 docker0
docker run -d -P --name tomcat01 tomcat
docker run -d -P --name tomcat01 --net bridge tomcat
```

> **docker0 特点**：默认，域名不能访问，`--link` 可以打通连接！

```
docker network create --help #创建网络
Usage:  docker network create [OPTIONS] NETWORK

Create a network

Options:
      --attachable            Enable manual container attachment
      --aux-address map       Auxiliary IPv4 or IPv6 addresses used by Network driver (default map[])
      --config-from string    The network from which to copy the configuration
      --config-only           Create a configuration only network
  -d, --driver string         Driver to manage the Network (default "bridge")
      --gateway ipSlice       IPv4 or IPv6 Gateway for the master subnet (default [])
      --ingress               Create swarm routing-mesh network
      --internal              Restrict external access to the network
      --ip-range ipNetSlice   Allocate container ip from a sub-range (default [])
      --ipam-driver string    IP Address Management Driver (default "default")
      --ipam-opt map          Set IPAM driver specific options (default map[])
      --ipv4                  Enable or disable IPv4 address assignment (default true)
      --ipv6                  Enable or disable IPv6 address assignment
      --label list            Set metadata on a network
  -o, --opt map               Set driver specific options (default map[])
      --scope string          Control the network's scope
      --subnet strings        Subnet in CIDR format that represents a network segment
      
      
# 我们可以自定义一个网络！
# --driver bridge
# --subnet 192.168.0.0/16
# --gateway 192.168.0.1
docker network create --driver bridge --subnet 192.168.0.0/16 --gateway 192.168.0.1 mynet

#不同集群推荐使用不同网络这就涉及到网络联通
      
```

### 网络联通

核心命令

```bash
docker network connect <网络名> <容器名>
```

把已有容器接入到另一个网络，无需重启容器。

#### 完整示例

```bash
# 1. 在 mynet 中启动容器
docker run -d -P --name tomcat-net-01 --net mynet tomcat
docker run -d -P --name tomcat-net-02 --net mynet tomcat

# 2. 在默认 bridge 网络中启动容器
docker run -d -P --name tomcat1 tomcat
docker run -d -P --name tomcat2 tomcat

# 3. 把 tomcat1 接入 mynet，打通两个网络
docker network connect mynet tomcat1
```

#### 验证连通性

```bash
# tomcat1 可以通过容器名访问 mynet 中的容器
docker exec -it tomcat1 ping tomcat-net-01

# mynet 中的容器也能访问 tomcat1
docker exec -it tomcat-net-01 ping tomcat1
```

## Docker Compose

前面我们都是用 `docker run` 一个个启动容器，参数又长又难记。一个真实项目至少得跑 应用 + 数据库 + 缓存 + 反向代理，四五个容器，每个都带一堆 `-v -e -p --net`，手敲一遍不仅累，换台机器还复现不出来。

Docker Compose 就是来干这个的：**把一堆 `docker run` 写进一个 YAML 文件，一条命令拉起整个项目。**

> 定位要清楚：Compose 是**单机**多容器编排工具，管的是"一台机器上的一个项目"。跨机器调度是 Swarm / Kubernetes 的事。

### 核心结论

- 一个 `compose.yaml` 描述 **一个项目（project）**
- 项目里的每个 **服务（service）** 对应一组容器
- Compose 会**自动创建专用网络**，同一项目的服务之间**直接用服务名当域名互访**
- 一条 `docker compose up -d` 干完：建网络 → 建卷 → 拉/构建镜像 → 按依赖顺序启动

也就是说，前面"自定义网络 + `--link`"那一套手工活，Compose 全自动替你做了 —— 这就是为什么网络那节最后要往 Compose 走。

------

### 三个概念

表格

| 概念              | 说明                                       | 例子                        |
| ----------------- | ------------------------------------------ | --------------------------- |
| **项目 project**  | 一个 compose 文件就是一个项目，默认名 = 目录名 | 目录 `myblog` → 项目 `myblog` |
| **服务 service**  | 一类容器，写在 `services:` 下面            | `web`、`db`、`redis`        |
| **容器 container** | 服务实际跑起来的实例                       | `myblog-web-1`、`myblog-db-1` |

> 命名规则：容器叫 `<项目名>-<服务名>-<副本序号>`，网络叫 `<项目名>_default`，命名卷叫 `<项目名>_<卷名>`。`docker ps` 里那些带前缀的名字就是这么来的。

------

### 安装

Docker Desktop 和新版 Docker Engine 一般都自带，**先确认**：

```bash
docker compose version
# Docker Compose version v2.29.1
```

没有就单独装插件：

```bash
# Debian / Ubuntu
apt install docker-compose-plugin

# CentOS / RHEL
yum install docker-compose-plugin

# 手动装（通用，注意 CPU 架构）
DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
mkdir -p $DOCKER_CONFIG/cli-plugins
curl -SL https://github.com/docker/compose/releases/download/v2.29.1/docker-compose-linux-x86_64 \
  -o $DOCKER_CONFIG/cli-plugins/docker-compose
chmod +x $DOCKER_CONFIG/cli-plugins/docker-compose
```

#### v1 |v2 

表格

|      | v1                    | v2                          |
| ---- | --------------------- | --------------------------- |
| 命令 | `docker-compose`（短横线） | `docker compose`（空格，插件形式） |
| 实现 | Python                | Go，作为 docker CLI 插件     |
| 状态 | 2023-07 起已 EOL      | 现在唯一在维护的            |
| 速度 | 慢                    | 快很多                      |

> 网上大量老教程还在写 `docker-compose`，照抄会报 `command not found`。**一律用 `docker compose`（中间是空格）。**

------

### 快速体验

最小可跑的例子，`compose.yaml`：

```yaml
services:
  web:
    image: nginx:1.27-alpine
    ports:
      - "8080:80"
    depends_on:
      - redis
  redis:
    image: redis:7-alpine
```

```bash
# 后台拉起（第一次会自动建网络、拉镜像）
docker compose up -d

# 看状态
docker compose ps

# 验证服务名可以直接当域名用
docker compose exec web nslookup redis

# 收工
docker compose down
```

> 注意 `up` 和 `down` 都必须在 **compose 文件所在目录**执行，或者用 `-f` 指定路径。

------

### compose 文件的三层结构

```yaml
# 第 1 层：服务（唯一必写的顶层字段）
services:
  web:
    image: nginx
  db:
    image: mysql:8.0

# 第 2 层：网络（不写也会自动建一个 <项目名>_default）
networks:
  frontend:
  backend:
    internal: true      # 禁止该网络访问外网

# 第 3 层：数据卷（供 services 引用）
volumes:
  db_data:
  redis_data:
```

> **卷和网络必须在顶层先声明，服务里才能引用。** 新手最常见的报错就是卷忘了在顶层声明。

#### 默认文件名与查找顺序

表格

| 优先级 | 文件名               |
| ------ | -------------------- |
| 1      | `compose.yaml`       |
| 2      | `compose.yml`        |
| 3      | `docker-compose.yaml` |
| 4      | `docker-compose.yml` |

推荐统一用 `compose.yaml`（Compose v2 的官方写法）。

------

### yaml 编写规则（踩坑高发区）

**1. 缩进只能用空格，绝对不能按 Tab** —— 报错信息还特别难懂。

**2. 端口号必须加引号**

```yaml
ports:
  - "8080:80"     # 正确
  - 8080:80       # 危险！YAML 会把 xx:xx 当六十进制解析，60:60 这类直接算错
```

**3. 冒号后面必须有空格**

`key: value` 是对的，`key:value` 会被当成一个整体字符串。

**4. 容器里要用的 `$VAR` 得写成 `$$VAR`**

`$` 是 Compose 的变量插值符，不转义会先被 Compose 自己吃掉：

```yaml
environment:
  - JAVA_OPTS=-Xms$${XMS}    # 容器里拿到的才是 ${XMS}
```

**5. `version: '3'` 已经废弃** —— 那是 v1 时代的产物，Compose v2 会提示 `the attribute version is obsolete`，直接删掉不写。

**6. 相对路径是相对 compose 文件的位置**，不是你敲命令的位置。

------

### 常用配置字段

表格

| 字段              | 作用                          | 例子                                     |
| ----------------- | ----------------------------- | ---------------------------------------- |
| `image`           | 指定镜像                      | `nginx:1.27-alpine`                      |
| `build`           | 现场构建镜像                  | `context: ./app` + `dockerfile: Dockerfile` |
| `container_name`  | 固定容器名（写了就不能扩容）  | `site-app`                               |
| `ports`           | 端口映射 `宿主:容器`          | `"8080:80"`、`"127.0.0.1:8080:80"`（只本机可访问） |
| `expose`          | 只声明端口，不映射到宿主      | `["8080"]`                               |
| `environment`     | 环境变量                      | `- MYSQL_ROOT_PASSWORD=123`              |
| `env_file`        | 从文件读环境变量              | `.env`                                   |
| `volumes`         | 挂载卷/目录                   | `./html:/usr/share/nginx/html:ro`        |
| `networks`        | 加入哪些网络                  | `[frontend, backend]`                    |
| `depends_on`      | 启动顺序依赖                  | 见下一节                                 |
| `command`         | 覆盖镜像的 CMD                | `redis-server --appendonly yes`          |
| `entrypoint`      | 覆盖 ENTRYPOINT               | `["/bin/sh","-c"]`                       |
| `restart`         | 重启策略                      | `unless-stopped`                         |
| `healthcheck`     | 健康检查                      | 见下一节                                 |
| `logging`         | 日志驱动与滚动                | `max-size: "20m"`                        |
| `init`            | 用 tini 做 1 号进程，回收僵尸进程 | `true`                                |
| `user`            | 以哪个用户运行                | `"1000:1000"`                            |
| `extra_hosts`     | 写 hosts                      | `"host.docker.internal:host-gateway"`    |
| `shm_size`        | /dev/shm 大小（Chrome、PG 常用） | `256m`                                |
| `profiles`        | 按需启动（dev/prod 分组）     | `[dev]`                                  |
| `deploy`          | 资源限制等                    | 见"资源限制"一节                         |

#### 资源限制：两种写法

```yaml
services:
  db:
    image: mysql:8.0
    # 写法一：服务级简写（老格式沿用）
    mem_limit: 1g
    cpus: 1.5

    # 写法二：Compose 规范写法（推荐）
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 1g
        reservations:
          memory: 256m
```

> **重要纠偏**：很多人以为 `deploy` 只有 Swarm 才生效。实际上 Compose v2 跑 `docker compose up` **是会应用 `deploy.resources.limits` 和 `reservations` 的**，不需要 Swarm。真正只有 Swarm 才认的是 `mode`、`placement`、`update_config`、`endpoint_mode` 这几个。
>
> 一个项目里选一种写法就行，别两个都写。验证是否生效：
>
> ```bash
> docker inspect --format '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}' <容器名>
> # 1073741824 1500000000   （单位分别是字节和纳核，0 表示没设上）
> ```

#### 日志一定要设滚动

不加限制，json-file 日志能几天把磁盘写满：

```yaml
logging:
  driver: json-file
  options:
    max-size: "20m"
    max-file: "5"
```

------

### 最大的坑：depends_on ≠ 就绪

`depends_on` **只保证启动顺序，不保证服务可用**。mysql 容器起来了但还在初始化，app 就去连，照样连不上然后崩掉。

```yaml
services:
  app:
    depends_on:
      mysql:
        condition: service_healthy      # 关键：等它健康了再启动我
  mysql:
    image: mysql:8.0
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 40s                 # 给初始化留的宽限期，这段时间失败不算数
```

#### condition 三种取值

表格

| 取值                            | 含义                       |
| ------------------------------- | -------------------------- |
| `service_started`（默认）       | 对方容器启动了就算数       |
| `service_healthy`               | 对方**健康检查通过**才算数（必须配 `healthcheck`） |
| `service_completed_successfully` | 对方**成功退出**才算数（跑迁移、初始化脚本用） |

#### healthcheck 写法

```yaml
healthcheck:
  test: ["CMD-SHELL", "curl -fs http://localhost/health || exit 1"]
  # 也可以用 CMD（不经过 shell）：["CMD", "curl", "-f", "http://localhost/health"]
  interval: 15s       # 多久查一次
  timeout: 5s         # 单次超时
  retries: 3          # 连续失败几次判为 unhealthy
  start_period: 30s   # 启动宽限期，期间失败不计入 retries
```

> **总结**：`depends_on` + `healthcheck` + `condition: service_healthy` 三件套，才是解决"启动顺序"的正确姿势。应用侧最好再加一层**连接重试**，别把宝全押在编排上。

------

### 常用命令

表格

| 命令                                  | 作用                                 |
| ------------------------------------- | ------------------------------------ |
| `docker compose up -d`                | 后台创建并启动全部服务               |
| `docker compose up -d --build`        | 先重新构建镜像再启动                 |
| `docker compose up -d --force-recreate` | 强制重建容器（配置没变也想重建时用） |
| `docker compose up -d --remove-orphans` | 清掉文件里已删除的残留服务         |
| `docker compose up -d --wait`         | 等到所有服务 healthy / running 才返回 |
| `docker compose up -d --scale app=3`  | 扩容到 3 个副本（无状态服务才行）    |
| `docker compose down`                 | 停止并删除容器 + 网络                |
| `docker compose down -v`              | **连命名卷一起删（数据没了！）**     |
| `docker compose down --rmi all`       | 顺便删镜像                           |
| `docker compose start/stop/restart`   | 启动/停止/重启（**不读配置变更**）   |
| `docker compose ps -a`                | 查看服务状态（含已停止）             |
| `docker compose logs -f svc`          | 跟踪某个服务日志                     |
| `docker compose logs --tail=100 svc`  | 看最后 100 行                        |
| `docker compose exec svc sh`          | 进入**已有**容器                     |
| `docker compose run --rm svc cmd`     | **新建一个**临时容器跑一次性命令     |
| `docker compose top`                  | 看各容器内的进程                     |
| `docker compose stats`               | 实时资源占用                         |
| `docker compose build --no-cache`     | 构建镜像，不用缓存                   |
| `docker compose pull`                 | 拉取所有镜像                         |
| `docker compose config`               | 校验并渲染出最终配置（排错首选）     |
| `docker compose port svc 80`          | 看某个服务端口映射到宿主哪个口       |
| `docker compose events`               | 实时事件流                           |

> **改了 compose 文件要生效，必须 `docker compose up -d`**，光 `restart` 不会重新读取配置，这是个高频误区。

#### 项目名与多文件

```bash
# 指定项目名（默认是目录名）
docker compose -p myblog up -d

# 指定文件；多个 -f 时，后面的覆盖前面的
docker compose -f compose.yaml -f compose.prod.yaml up -d

# 只操作某个服务
docker compose up -d --no-deps app     # 只重建 app，不动它的依赖
```

------

### 变量、多套环境与 profiles

#### .env 文件

compose 文件同级放一个 `.env`，Compose 会自动加载：

```bash
# .env
MYSQL_ROOT_PASSWORD=Str0ngPass
TAG=1.4.2
COMPOSE_PROJECT_NAME=site
```

```yaml
services:
  app:
    image: myapp:${TAG:-latest}          # :- 表示默认值
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
```

> 插值语法：`${VAR}` 必填、`${VAR:-默认}` 为空取默认、`${VAR:?报错提示}` 缺失直接报错。`docker compose config` 可以看到插值后的真实内容。

#### profiles 按需启动

开发才用的调试容器，别跟着生产一起起来：

```yaml
services:
  app:
    image: myapp:1.0
  adminer:
    image: adminer
    profiles: ["dev"]
```

```bash
docker compose up -d                 # 只起 app
docker compose --profile dev up -d   # 额外起 adminer
```

没写 `profiles` 的服务**永远启动**，写了的只在 `--profile` 激活时才起。

#### 多套环境：基础文件 + 覆盖文件

```bash
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

`compose.prod.yaml` 里只写差异项（改端口、加资源限制、去掉调试挂载），相同字段覆盖、不同字段合并。比复制两份完整文件好维护得多。

------

### 实战：Nginx + SpringBoot + MySQL + Redis

一个完整的站点编排，`compose.yaml`：

```yaml
services:
  nginx:
    image: nginx:1.27-alpine
    container_name: site-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro      # 配置只读挂载
      - ./dist:/usr/share/nginx/html:ro          # 前端构建产物
      - ./nginx/logs:/var/log/nginx
    depends_on:
      app:
        condition: service_healthy
    networks:
      - frontend
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "20m"
        max-file: "3"

  app:
    build:                                        # 现场用 Dockerfile 构建
      context: ./app
      dockerfile: Dockerfile
    container_name: site-app
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/site?useUnicode=true&characterEncoding=utf8
      - SPRING_REDIS_HOST=redis
      - SPRING_REDIS_PASSWORD=${REDIS_PASSWORD:-redis123}
    env_file:
      - .env
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:8080/actuator/health || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 60s                           # SpringBoot 启动慢，给足宽限
    init: true                                    # 正确转发信号、回收僵尸进程
    networks:
      - frontend
      - backend
    restart: unless-stopped

  mysql:
    image: mysql:8.0
    container_name: site-mysql
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-root123}
      MYSQL_DATABASE: site
      TZ: Asia/Shanghai
    volumes:
      - mysql_data:/var/lib/mysql                  # 命名卷，数据不随容器消失
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql:ro   # 首次启动自动执行
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-uroot", "-p${MYSQL_ROOT_PASSWORD:-root123}"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 40s
    networks:
      - backend
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: site-redis
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD:-redis123}
    volumes:
      - redis_data:/data
    networks:
      - backend
    restart: unless-stopped

networks:
  frontend:                 # 对外（nginx ↔ app）
  backend:
    internal: true          # 内网，数据库和缓存不直接暴露到外网

volumes:
  mysql_data:
  redis_data:
```

配套操作：

```bash
# 1. 准备环境变量
cp .env.example .env && vim .env

# 2. 校验配置（先看渲染结果，别直接 up）
docker compose config

# 3. 构建并后台启动
docker compose up -d --build

# 4. 跟日志确认是否真的起来了
docker compose logs -f app

# 5. 看健康状态
docker compose ps

# 6. 更新：改完代码重新构建，只重建 app
docker compose up -d --build --no-deps app

# 7. 停机（不清数据）
docker compose down
```

> 这个例子里几个关键点：`internal: true` 把数据库关在内网；`healthcheck` + `condition: service_healthy` 保证启动顺序；命名卷保证 `down` 之后数据还在；日志全部加了滚动。

------

### 避坑清单

1. **`down -v` 会删数据** —— 命名卷一起没了。生产环境慎用，或者把关键卷声明成 `external: true` 让它不受 compose 生命周期管理。
2. **`restart` 不等于重载配置** —— 改了 yaml 必须 `up -d`。
3. **生产别用 `latest` 标签** —— 拉到什么版本全看运气，锁定具体版本。
4. **镜像用具体版本**：`mysql:8.0` 可以，`mysql:latest` 不行。
5. **写了 `container_name` 就不能 `--scale` 扩容** —— 名字冲突。
6. **别把配置挂成可写** —— 不确定就加 `:ro`，容器被攻破时少一个写入点。
7. **SELinux 机器（CentOS/RHEL）挂载可能 Permission denied** —— 在挂载后加 `:z` 或 `:Z` 重打标签。
8. **容器内访问宿主机**用 `host.docker.internal`，Linux 上需要在 `extra_hosts` 写 `host.docker.internal:host-gateway`。
9. **网络隔离要用多个 network**，而不是把所有服务塞进默认网络 —— 数据库没必要和 nginx 互通。
10. **健康检查别写太激进**，`interval` 太短会给应用额外压力，`start_period` 该给就给。
11. **退出码 137 不一定是 OOM** —— `128+9` 是 SIGKILL 的指纹。`docker inspect --format '{{.State.OOMKilled}}' <容器>` 返回 `true` 才是真 OOM，`false` 通常是因为没响应 SIGTERM、被 stop 超时强杀。
12. **`up` 报端口占用**先查 `ss -lntp | grep 80`，很多时候是宿主已经有 nginx 了。

------

###小结

> **一句话**：Compose = 把 `docker run` 命令 YAML 化 + 自动建网络/卷 + 一条命令管一整个项目。
>
> 三件事记住就不会出大问题：
>
> 1. **依赖顺序** → `depends_on` + `healthcheck` + `condition: service_healthy`
> 2. **数据不丢** → 用顶层声明的**命名卷**，别手贱 `down -v`
> 3. **能长期跑** → `restart: unless-stopped` + 日志滚动 + 资源限制
>
> 它是**单机**方案。要多机器、要自动扩缩容、要滚动更新不中断，就得往下看 Swarm 和 K8s 了。









## Docker Swarm

Compose 解决的是"一台机器上一个项目"的问题。Swarm 解决的是"很多台机器一起跑同一个项目"的问题 —— **把多台 Docker 主机捏成一个虚拟的 Docker**，用一条命令把容器调度到成百上千台机器上。

> **定位一句话**：Swarm = Docker 自带的、多主机的容器编排工具。
> 跨机器调度 + 服务自愈 + 滚动更新 + 4 层负载均衡，**全包了，不用额外装组件**。
> 缺点也直说：生态被 Kubernetes 碾压，新项目基本直接上 K8s。但 Swarm 内置、零依赖，小团队 / 内部项目拿来就用，依然是性价比很高的方案。

### 节点

集群里就两类节点：

| 节点 | 角色 | 干什么的 |
|------|------|----------|
| **Manager（管理节点）** | 调度 + 决策 | 接收命令、维护集群状态、调度服务、把任务派给 worker。**集群的大脑**，可以有多个（奇数个最好，靠 Raft 共识选主） |
| **Worker（工作节点）** | 执行 | 只负责跑容器（task），不参与调度，状态汇报给 manager |

```
Manager(主)  ←──Raft 共识──→  Manager(副)  ←──→  Manager(副)
       │                                              │
       ▼                                              ▼
   Worker 1     Worker 2     Worker 3     ...     Worker N
  (跑容器)     (跑容器)     (跑容器)             (跑容器)
```

- 显然 manager 也能跑容器（默认 `--availability active` 同时调度任务），不是"管不管人"的角色。
- **生产推荐**：至少 **3 个 manager** + N 个 worker。1 个 manager 是单点，2 个 manager 会脑裂（Raft 必须过半才能选举）。
- 节点之间通信靠这几个端口，部署前防火墙必须放行：

| 端口 | 协议 | 用途 |
|------|------|------|
| 2377 | TCP  | 集群管理通信 |
| 7946 | TCP+UDP | 节点发现 |
| 4789 | UDP  | Overlay 网络数据 |

### 几个必须先搞清的概念

| 概念 | 是什么 | 一句话总结 |
|------|--------|-----------|
| **Service（服务）** | 服务的"期望状态" | "我要 nginx 跑 3 个副本" 这种声明 |
| **Task（任务）** | Service 的一个具体实例 | **一个 task = 一个容器**，task 是 Swarm 调度的最小单位 |
| **Stack（应用栈）** | 一组关联 Service 的集合 | 一份 compose 文件 = 一个 stack |
| **Replicated Service** | 指定副本数（默认） | 精确控制跑几个 |
| **Global Service** | 每个节点跑一个 | 日志收集、监控 agent 这类节点级服务 |

```
Service（声明：我要 3 个 nginx）
   ├── Task 1 ──→ Container（在 worker1 上跑）
   ├── Task 2 ──→ Container（在 worker2 上跑）
   └── Task 3 ──→ Container（在 worker3 上跑）
```

> **Task 挂了 Swarm 会自动在别的节点重建一个新容器**，这就是 Swarm 的"自愈"机制。Task 才是 Swarm 调度的最小单位，不是 Container。

### 集群搭建步骤

#### 1. 初始化 Manager（想做主节点的那台机器上）

```bash
# 多网卡机器必须指定，不然 Swarm 不知道用哪个 IP 互相通信
docker swarm init --advertise-addr 192.168.1.10
```

成功后会返回一长串 `docker swarm join ...` 命令，**复制下来**，worker 加入要用：

```
Swarm initialized: current node (xxxxxxxxxxxx) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-xxxxxxxxxxxx \
    192.168.1.10:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

忘了复制也没事，manager 上随时能查：

```bash
docker swarm join-token worker      # 查 worker 加入令牌
docker swarm join-token manager     # 查 manager 加入令牌
```

#### 2. 加入 Worker（其他机器）

把上面复制下来的 join 命令贴到 worker 机器执行：

```bash
docker swarm join --token SWMTKN-1-xxxxxxxxxxxx 192.168.1.10:2377
This node joined a swarm as a worker.
```

#### 3. 验证集群（在 manager 上）

```bash
docker node ls
# ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
# xxxxxxxxxxxx *               manager1   Ready     Active         Leader           27.3.1
# yyyyyyyyyyyy                 worker1    Ready     Active                          27.3.1
# zzzzzzzzzzzz                 worker2    Ready     Active                          27.3.1
```

- 带 `*` 的是当前操作的节点
- `MANAGER STATUS` 列显示 `Leader` / `Reachable` 说明 manager 集群已经起来了

#### 4. 退集群 / 清理

```bash
# worker 主动离开（在 worker 上执行）
docker swarm leave

# manager 主动离开（要加 --force，不然不让走）
docker swarm leave --force

# manager 上把某个节点踢出去
docker node rm <节点ID>
```

> **踩坑**：新机器刚加入集群时，**manager 不会立刻把任务调度过去** —— 新节点往往还没准备好（镜像要拉、卷要挂）。建议用 `docker node update --availability drain <node>` 先把它"排空"，等就绪了再切回 `active`。

### docker Service

Service 是 Swarm 调度的核心，`docker service` 命令专门管它。

#### 创建 / 查看 / 扩缩

```bash
# 创建
docker service create \
  --name nginx \
  --replicas 3 \
  --publish 80:80 \
  nginx:1.27-alpine

# 不指定副本默认 1 个
docker service create --name nginx -p 80:80 nginx:1.27-alpine

# 查看
docker service ls                 # 所有服务
docker service ps nginx           # 看 nginx 这条服务的 task 分布
docker service inspect nginx      # 完整元数据

# 扩缩容
docker service scale nginx=5             # 扩到 5 个
docker service scale nginx=2 redis=3      # 一次改多个
```

`docker service ps` 输出的 `NODE` 列就是 task 真实跑在哪台机器上，可以快速看出调度是否均衡。

#### 滚动更新

```bash
# 创建时就指定滚动策略
docker service create \
  --name nginx \
  --replicas 10 \
  --update-delay 10s           # 每批更新间隔 10 秒
  --update-parallelism 2       # 每次并行更新 2 个
  --update-failure-action rollback \
  nginx:1.27-alpine

# 已有服务升级镜像
docker service update --image nginx:1.27.2 nginx

# 回滚到上一个版本
docker service update --rollback nginx
```

> **注意**：滚动更新的默认参数是 `--update-delay 0s --update-parallelism 1`，**一刀切不缓冲**，生产必须显式给值，不然一次全切容易炸。

#### Global 服务（每个节点一个）

```bash
docker service create \
  --name node-exporter \
  --mode global \
  prom/node-exporter
```

Global 模式**每个节点都会跑一个**，常用于日志收集、监控 agent 这种"节点级别"的服务。

#### 删除

```bash
docker service rm nginx                       # 删指定服务
docker service rm $(docker service ls -q)     # 全删，暴力慎用
```

### Overlay 网络 & Routing Mesh

Swarm 跨机器通信靠 **Overlay 网络**，是 Swarm 自带的"跨主机容器网络"，和 Compose 默认的 bridge 完全不同。

```bash
# Swarm overlay 网络
docker network create --driver overlay mynet

# 普通容器也想 attach 上来，加 --attachable
docker network create --driver overlay --attachable mynet
```

服务跑在 overlay 网络上后，**任何节点上的 task 都通过服务名直接互访**（和 Compose 的服务名 DNS 一个道理），IP 怎么变都不影响。

**Routing Mesh**（路由网格）是 Swarm 的"黑科技"：

```
                ┌─────────────────────────────────────┐
                │  Routing Mesh（集群级 4 层 LB）      │
                │  任何节点的 80 端口都能接到流量       │
                └─────────────────────────────────────┘
                       ↓             ↓             ↓
                 Worker1:80    Worker2:80    Worker3:80
                 (跑 task1)    (跑 task2)    (跑 task3)
```

只要服务声明了 `--publish 80:80`，**访问任何一个节点的 80 端口，都会被路由到真正跑容器的那个节点**。内置 4 层负载均衡，**不用 nginx 不用 haproxy**。

> **注意**：Routing Mesh 是**集群入口的负载均衡**，不是 service 之间的负载均衡。Service 之间互访还是走 overlay 网络的 DNS 轮询。

### docker Stack

一个真实项目不可能只有一个 service。Stack 就是"把一份 compose 文件部署到 Swarm"。

`compose.yaml`（Swarm 模式）：

```yaml
version: "3.8"   # Swarm 必须写 version，3.8 起步
services:
  web:
    image: nginx:1.27-alpine
    ports:
      - "80:80"
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "0.5"
          memory: 128M
    networks:
      - frontend

  api:
    image: myapi:1.0
    deploy:
      replicas: 2
    networks:
      - frontend
      - backend

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root123
    volumes:
      - db_data:/var/lib/mysql
    deploy:
      placement:
        constraints:
          - node.role == worker    # 数据库只跑在 worker 上

networks:
  frontend:
  backend:
    driver: overlay               # Swarm 必须用 overlay

volumes:
  db_data:
```

部署 + 管理：

```bash
# 部署
docker stack deploy -c compose.yaml myblog
# Ignoring unsupported options: build   # 提示：stack 不支持 build，得先 build 好

# 管理
docker stack ls                          # 列所有 stack
docker stack services myblog             # 看 stack 里的服务
docker stack ps myblog                   # 看 stack 里的 task（含分布到哪些节点）
docker stack rm myblog                   # 删整个 stack（卷默认保留，不会丢数据）
```

> **compose 文件差异**：Swarm 模式多了 `deploy:` 顶层字段，**`build:` 在 stack 里不生效**，必须先 `docker build` 出镜像再部署。`container_name` 在 Swarm 里**也无效**（Swarm 自动起名）。这两个坑踩过一次就记住了。

#### stack vs compose up 的区别

表格

| 项目 | `docker compose up` | `docker stack deploy` |
|------|---------------------|------------------------|
| 跑在 | **单机** Docker Engine | Swarm 集群 |
| 调度 | 不调度，全跑本机 | Swarm manager 跨节点调度 |
| 副本 | `deploy.replicas` 单机会照拉 | **真正生效**，按调度规则分布 |
| 高可用 | task 挂了不会自动重建 | task 挂了**自动在别的节点重建** |
| 网络 | bridge | overlay |
| 文件 | compose v2 规范 | compose v3 规范（要带 `version: "3.x"`） |

### docker Secret

Swarm 里专门的 secret 管理，用来安全地分发**密码、证书、API key** 这类敏感数据。

#### 创建 / 使用

```bash
# 从标准输入创建（推荐，密码不会落盘）
echo "root123" | docker secret create mysql_password -

# 从文件创建
docker secret create mysql_password ./password.txt
```

`compose.yaml` 里使用：

```yaml
services:
  db:
    image: mysql:8.0
    secrets:
      - mysql_password
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_password   # MySQL 支持 _FILE 形式

secrets:
  mysql_password:
    external: true   # 表示这个 secret 是 swarm 里已经存在的，不在 stack 文件里创建
```

容器内访问：secret 会被挂载到 `/run/secrets/<secret名>`，**默认 tmpfs，容器删除就没了**。

```bash
docker secret ls
docker secret inspect mysql_password
docker secret rm mysql_password
```

> **注意**：Swarm secret 是 **Raft 日志加密存储** + 节点间 TLS 传输，已经比 compose 里明文写 `environment` 安全多了。但 secret 一旦创建**不能更新内容**，只能 rm 重新建，这是个设计缺陷，改密码比较烦。

### docker Config

Config 和 Secret 几乎一样，区别是 **Config 存非敏感的配置**（nginx.conf、application.yml 这类），可以**滚动更新**。

```bash
# 从文件创建
docker config create nginx_conf ./nginx.conf

# 更新（先 rm 再 create；service 不会自动 reload，要让容器"看到"变更得挂到 volume）
docker config rm nginx_conf
docker config create nginx_conf ./nginx.conf
```

`compose.yaml` 里使用：

```yaml
services:
  web:
    image: nginx:1.27-alpine
    configs:
      - source: nginx_conf
        target: /etc/nginx/nginx.conf
```

### 常用命令一览

表格

| 命令 | 作用 |
|------|------|
| `docker swarm init` | 初始化集群（manager） |
| `docker swarm join --token xxx` | 加入集群 |
| `docker swarm leave / leave --force` | 离开集群（worker / manager） |
| `docker swarm join-token worker/manager` | 查看加入令牌 |
| `docker node ls / inspect / rm` | 列节点 / 详情 / 删除 |
| `docker node update --availability drain/active/pause <node>` | 改节点状态 |
| `docker service create / ls / ps / inspect / rm` | 管 service |
| `docker service scale <svc>=N` | 扩缩容 |
| `docker service update --image xxx <svc>` | 更新镜像 |
| `docker service update --rollback <svc>` | 回滚 |
| `docker service logs -f <svc>` | 看服务日志（所有 task 一起） |
| `docker stack deploy -c xxx.yaml <name>` | 部署 stack |
| `docker stack ls / services / ps / rm` | 管 stack |
| `docker secret ls / create / rm` | 管 secret |
| `docker config ls / create / rm` | 管 config |
| `docker network create --driver overlay <name>` | 建 overlay 网络 |

### 避坑清单

1. **端口必须放行**：2377/tcp、7946/tcp+udp、4789/udp，少一个集群就起不来或调度失败。
2. **奇数个 manager**：1 个单点，2 个脑裂（Raft 必须过半才能选举），3 / 5 / 7 是安全数。
3. **stack 文件不支持 `build`**：先本地 build 出镜像 push 上去，再 deploy。
4. **`container_name` 在 Swarm 里无效**：Swarm 必须自己起名，否则 task 重启会冲突。
5. **Rolling update 默认参数太激进**：`--update-delay 0s --update-parallelism 1` 一刀切不缓冲，生产必须显式给值。
6. **Secret 不能更新内容**：要改密码？rm 重新 create。
7. **Routing Mesh ≠ Service Mesh**：它是 4 层 LB，HTTPS 终结还是要靠 nginx / traefik。
8. **manager 默认也能跑 task**：想让它纯管理，把新 manager 加进来后用 `docker node update --role manager` 改角色，或者创建时 `--availability drain`。
9. **节点 hostname 必须唯一**：否则 `docker node ls` 看着乱、调度也会出问题。
10. **数据卷默认 local driver**，**跨节点不会迁移**。数据库类有状态服务要用 NFS / 云盘 / Rex-Ray 之类的共享存储，否则节点一挂数据就丢。

### 小结

> **一句话**：Swarm = Docker 内置的多机编排，门槛低、零依赖，适合中小团队和内部项目。
>
> 三个核心记住不出大问题：
>
> 1. **调度单位是 Task**（不是 Container）→ task 挂了 Swarm 自动在别的节点重建
> 2. **Stack = compose 文件部署到 Swarm** → `deploy:` 字段管副本/资源/约束
> 3. **Routing Mesh 自带 4 层 LB** → 任何节点入口都能转发到 task 真实所在节点
>
> 缺点也直说：**生态比 K8s 差远了**，文档少、二次开发难、生产级特性（CRD、Operator、自动扩缩 HPA）基本缺失。
> 项目要长期演进、对生态有要求 → **直接上 K8s**。
> 短期跑业务、内部工具、小团队 → Swarm 性价比无敌。
