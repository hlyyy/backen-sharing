# Docker介绍
## 容器的出现
 我们都知道，一个产品的出现要经过开发、测试、部署、上线等等步骤。而如今，开发人员通常使用多种服务构建和主装应用，而且这些应用很可能会部署到不同的环境，比如虚拟服务器、公有云和私有云。

这个时候就产生了一些问题。首先，应用包含的多种服务有自己所依赖的库和软件包；另一方面存在多种部署的环境，服务在运行时可能需要迁移到不同的环境中。那如何让每种服务在所有的部署环境中顺利运行呢？

这个时候容器技术就出现了：我们把所有的环境、依赖、代码打包好，生成一个镜像。只需要一次配置好环境，换到别的机子上就可以一键部署好，大大简化了操作。

再来看看 docker 的图标：

![](https://upload-images.jianshu.io/upload_images/21203385-015c997cc7c72878.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Docker 将集装箱的思想运用到软件打包上，为代码提供了一个基于容器的标准化运输系统。Docker 可以将任何应用及其依赖打包成一个轻量级、可移植、自包含的容器。容器可以运行在几乎所有操作系统上。

我们现在知道 docker 是如何出现的了，总结成一句话：“**一次封装，到处运行**”。

## Docker 架构
先来看看 docker 架构图：
![docker架构图](https://upload-images.jianshu.io/upload_images/21203385-9526f26ee8fc6ce7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Docker 采用的是 Client/Server 架构。客户端向服务端发送请求，服务器负责构建、运行和分发容器。客户端和服务器可以在同一个 Host 上，客户端也可以通过 socket 或 REST API 与远程服务器通信。

架构图中的几个核心组件：
- docker 客户端：Client
最常用的 docker 客户端是 docker 命令。通过 docker 我们可以方便地在Host上构建和运行容器。
- docker 服务器：Docker daemon 
Docker daemon 运行在 Docker host 上，负责创建、运行、监控容器，构建、储存镜像。默认配置下，Docker daemon 只能相应来自本地 Host 的客户端请求。
- docker 镜像：Images
可以将 docker 的镜像看成**只读模板**。镜像可以用来创建容器。一个镜像可以创建很多容器。镜像和容器的关系类似于面向对象编程中的类与对象。
- docker 容器：Container
Docker 容器就是镜像的运行实例。它可以被启动、开始、停止、删除。每个容器都是相互隔离的、保证安全的平台。可以这么认为，镜像就是软件的构建和打包阶段，而容器则是启动和运行阶段。
- docker 仓库：Registry
Registry 是存放 Docker 镜像的仓库，Registry 分私有和公有两种。
Docker Hub (https://hub.docker.com/) 是默认的 Registry ，由 Docker 公司维护，上面有数以万计的镜像可以下载和使用。当然，我们也可以创建自己私有的 Registry。
``docker pull``命令可以从 Registry 下载镜像。
``docker run``命令是先下载镜像 (如果本地没有)，然后再启动容器。

### 一个完整的例子:快速认识docker
```
huanglingyun$ docker run -d httpd    ①    
Unable to find image 'httpd:latest' locally     ②
latest: Pulling from library/httpd  
5b54d594fba7: Pull complete 
4b53bced9ee8: Pull complete     ③
33abd7401e3d: Pull complete 
8fb831f2b4d7: Pull complete 
f26db3b1c783: Pull complete 
Digest: sha256:f1d23356c95762858854958be00b66c92f7bf35a88b902575a2afd808e1ab29e
Status: Downloaded newer image for httpd:latest    ④
eef2406da215e77b1c44f70ff849d1c8452b2bbe98023a1348e25a55a6f736cd    ⑤
huanglingyun$
```
①在客户端执行``docker run``命令
②daemon 发现本地没有 httpd 的镜像
③daemon从Docker hub下载镜像
④下载完成，镜像 httpd 被保存到本地
⑤启动容器，这一长串是容器的ID

再看一下现在的镜像列表和正在运行的容器：
```
huanglingyun$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
httpd               latest              a8a9cbaadb0c        10 hours ago        166MB
huanglingyun$ docker ps
CONTAINER ID        IMAGE               COMMAND              CREATED             STATUS              PORTS               NAMES
eef2406da215        httpd               "httpd-foreground"   8 minutes ago       Up 8 minutes        80/tcp              affectionate_keldysh
huanglingyun$ 
```
## Docker镜像内部结构
我们已经知道了，镜像是一种轻量级、可执行的独立软件包，用来打包软件运行环境和基于运行环境开发的软件，它包含运行某个软件所需的所有内容，包括代码、运行时、库、环境变量和配置文件。

docker 的镜像实际上是由一层一层的文件系统构成，这种层级的文件系统就是 UnionFS。

UnionFs（联合文件系统）：union文件系统是一种分层、轻量级并且高性能的文件系统，它支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件下。

Union 文件系统是 Docker 镜像的基础，镜像可以通过分层来进行继承，基于基础镜像，可以制作各种具体的应用镜像。


特性：一次同时加载多个文件系统，但从外面看来，只能看到一个文件系统，联合加载会把各层文件系统叠加起来，这样最终的文件系统会包含所有底层的文件和目录。
#### base镜像
base镜像有两层含义：①不依赖其他镜像，从0(scratch)构建 ②其他镜像可以以之为基础进行拓展。
通常能称为base镜像的都是各种 Linux 发行的 docker 镜像，比如 Ubuntu、CentOS、Debian 等等。

以 CentOS 为例：
```
huanglingyun$ docker images centos
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              latest              470671670cac        3 months ago        237MB
```
我们发现 CentOS 的镜像只有200MB，比我们安装虚拟机的几个GB少很多。

![](https://upload-images.jianshu.io/upload_images/21203385-0446c6b172dce420.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这是因为：Linux操作系统由内核空间和用户空间组成。Linux 刚启动时会加载 bootfs 文件系统，之后 bootfs 会被删除掉。用户空间的文件系统是 rootfs。**对于 base 镜像来说，底层直接用 Host 的 kernel，自己只需要提供 rootfs 就行了**。

而对于精简版的 OS 来说，rootfs 可以很小，只需要包括最基本的命令、工具和程序库就可以了。所以 base 镜像占用的空间都不大。
#### 镜像的分层结构
Docker 支持通过扩展现有的镜像，创建新的镜像。
实际上，Docker Hub 中绝大部分镜像都是通过在 base 镜像中安装和配置需要的软件构建出来的。如图：

![](https://upload-images.jianshu.io/upload_images/21203385-1df313e7c5edc032.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，新镜像是从 base 镜像一层一层叠加生成的。每安装一个软件，就在现有镜像的基础上增加一层。

分层结构的优势：**共享资源**。比如多个镜像由相同的base镜像构建而来，那么 Docker Host 只需在磁盘上保存一份 base 镜像；同时内存中也只需加载一份 base 镜像，就可以为所有容器服务了。而且除了 base 镜像以外，镜像的每一层都可以被共享。
#### 容器的可写层
当我们用镜像启动容器时，一个新的可写层被加载到镜像的顶部。这一层通常被称作**容器层**，容器层下面的是**镜像层**。所有对容器的改动 - 无论添加、删除、还是修改文件都只会发生在容器层中。

![](https://upload-images.jianshu.io/upload_images/21203385-f7dbf4e9f51db306.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当我们对容器进行一些操作：
①添加文件 ：在容器中创建文件时，新文件被添加到容器层中。
②读取文件 ：在容器中读取某个文件时，Docker 会从上往下依次在各镜像层中查找此文件。一旦找到，打开并读入内存。
③修改文件 ：在容器中修改已存在的文件时，Docker 会从上往下依次在各镜像层中查找此文件。一旦找到，立即将其复制到容器层，然后修改之。
④删除文件 ：在容器中删除文件时，Docker 也是从上往下依次在镜像层中查找此文件。找到后，会在容器层中记录下此删除操作。

可见，容器层保存的是镜像变化的部分，不会对镜像本身进行任何修改。且只有当需要修改时才复制一份数据，这种特性被称作 Copy-on-Write。

总结：容器层 - 可写
          镜像层 - 可共享 - 不能被修改 - 只读

### 构建镜像
Docker 提供了两种构建镜像的方法：① docker commit 命令 ② 通过Dockerfile 来构建
#### docker commit
``docker commit``命令是创建新镜像最直观的方法，包含三个步骤：
- 运行容器
- 进入容器并修改容器
- 将容器保存为新的镜像
这是一种手工创建镜像的方式，容易出错且效率低。而且使用者并不知道镜像是如何创建出来的，安全性不知。所以 docker 并不推荐用这种方式构建镜像。
这个很好理解就不多讲了，重点是如何通过 Dockerfile 构建镜像。
#### Dockerfile
Dockerfile 是一个文本文件，记录了镜像构建的所有步骤。Dockerfile的内容包括执行代码或者是文件、环境变量、依赖包、服务进程和内核进程等等。通过``docker build``这个文件来创建镜像。
以 CentOS镜像的 Dockerfile 为例：
```
#Dockerfile
1 FROM scratch
2 ADD centos-7-docker.tar.xz /
3 CMD ["/bin/bash"]
```
``FROM``指定了基础镜像，当前镜像是基于哪个镜像的，centos 本身就是 base 镜像，所以它是从 scratch (0)开始构建的。``ADD``将 Host 的文件拷贝进镜像并且会自动处理 URL 和解压 tar 包，centos 的 tar 包会自动解压到`` / ``目录下。``CMD``指定一个容器启动时要运行的命令。

Dockerfile 的常用命令除了上面的还有很多，例如：``RUN``容器构建时需要运行的命令。``ENV``设置环境变量。``WORKDIR``设置默认的工作目录。``COPY``类似于``ADD``。``VOLUME``容器的数据卷，用于数据的保存和持久化工作。``EXPOSE``指定容器要打开的端口。具体的语法和命令差别不做过多的介绍了。

Docker 执行 Dockerfile 的大致流程：
①docker 从基础镜像运行一个容器
②执行每一条指令并对容器进行修改
③执行类似``docker commit``操作提交一个新的镜像层
④docker 再基于刚提交的镜像运行一个新的容器
⑤执行 Dockerfile 中的下一条指令直到所有指令都执行完成
由此可见，使用 Dockerfile 构建镜像，底层也是``docker commit``一层一层实现的。

#### docker  build
用 miniProject 作为示例，先看看 Dockerfile 文件：
```
  #Dockerfile
  1 FROM golang
  2 ENV GO111MODULE "on"
  3 ENV GOPROXY "https://goproxy.cn"
  4 ADD . $GOPATH/src/github.com/2020-LonelyPlanet-backend/miniProject
  5 WORKDIR $GOPATH/src/github.com/2020-LonelyPlanet-backend/miniProject
  6 COPY etc/localtime /etc/localtime
  7 RUN make
  8 EXPOSE 9090
  9 CMD ["./miniProject"]
```
然后用``docker build``创建镜像
```
huangliyundeair:miniProject huanglingyun$ docker build -t miniproject:v1  .    
Sending build context to Docker daemon  152.5MB
Step 1/9 : FROM golang
latest: Pulling from library/golang
90fe46dd8199: Pull complete 
35a4f1977689: Pull complete 
···
Digest: sha256:e36e74d9770fa9fbf06cda4494ae1e6442637e53d71fbd77560ac3eb976a1417
Status: Downloaded newer image for golang:latest
 ---> 2421885b04da
Step 2/9 : ENV GO111MODULE "on"
 ---> Running in 5ad2740ff60e
Removing intermediate container 5ad2740ff60e
 ---> e7075b535fbd
Step 3/9 : ENV GOPROXY "https://goproxy.cn"
 ---> Running in 4c98dd5049c3
Removing intermediate container 4c98dd5049c3
 ---> 316589a734eb
Step 4/9 : ADD . $GOPATH/src/github.com/2020-LonelyPlanet-backend/miniProject
 ---> 926ecaa94440
Step 5/9 : WORKDIR $GOPATH/src/github.com/2020-LonelyPlanet-backend/miniProject
 ---> Running in f26cf0e66359
Removing intermediate container f26cf0e66359
 ---> 3b77b943c038
Step 6/9 : COPY etc/localtime /etc/localtime
 ---> 7cf54ecc7738
Step 7/9 : RUN make
 ---> Running in 4d49bf1e0a84
gofmt -w .
go mod tidy
···
Removing intermediate container 4d49bf1e0a84
 ---> 23795a5b24cb
Step 8/9 : EXPOSE 9090
 ---> Running in 2221d5b05298
Removing intermediate container 2221d5b05298
 ---> ec6d55e4dd84
Step 9/9 : CMD ["./miniProject"]
 ---> Running in 118daaec1d2e
Removing intermediate container 118daaec1d2e
 ---> 254c7b660fdc
Successfully built 254c7b660fdc
Successfully tagged miniproject:v1
```
首先运行``docker build``命令，``-t``将新镜像命名为 miniproject ，命名末尾的``.``指的是docker 默认从当前目录下查找 Dockerfile 文件。也可以通过``-f``参数指定 Dockerfile 位置。
执行过程从上到下，分别是 Step1 到 Step9。

再查看一下本地镜像，看到了这个新镜像 miniproject 且标签为 v1。ID 与构建时输出一致。
```
huanglingyun$ docker images miniproject
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
miniproject         v1                  254c7b660fdc        About an hour ago   1.33GB
```
### docker仓库
类似于git，用户可以从 Docker hub 上 pull 镜像下来使用，同样用户可以将自己的镜像保存到 Docker hub 免费的 repository中。
方法很简单，终端登录上docker hub，再 push 就可以了。
注意：镜像的 registry 要包含完整的用户名，用``docker tag``改一下名字就行了。
```
huanglingyun$ docker tag miniproject:v1 hlyyy/miniproject:v1
huanglingyun$ docker push hlyyy/miniproject:v1
```
除了使用 Docker hub，国内也有免费的镜像仓库，比如阿里云。还可以在本地搭建镜像仓库，具体方法就不多说了。
## Docker容器
#### 容器与虚拟机的区别
虚拟机：需要安装整个操作系统。缺点：占用内存多，步骤多，启动慢。

容器：直接运行于 Host 的内核，没有硬件，仅包含运行时所需的 runtime 环境。优点：启动速度快，资源利用率高，性能开销小。

下图表示了容器各种状态之间的转换：

![](https://upload-images.jianshu.io/upload_images/21203385-3e0f017698a22e9e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

包括了容器的创建，运行，暂停，停止，删除。
## 总结
![](https://upload-images.jianshu.io/upload_images/21203385-8e778ab24c870c44.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面的内容可以总结为这一张图，包含了各个部分的关系。

以上是关于 docker 的一些基础知识，深入学习还有网络通信、数据管理和容器平台技术等等。
