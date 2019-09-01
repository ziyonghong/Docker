

##   一、Docker思想

集装箱

标准化：运输方式、存储方式、API接口

隔离



  

构建build 镜像  运行run 容器  把镜像传送ship到仓库 

## 二、Docker核心组件

### 1.Docker 客户端 - Client

Docker 客户端是 `docker` 命令。通过 `docker` 我们可以方便地在 Host 上构建和运行容器。

### 2.Docker 服务器 - Docker daemon

Docker daemon 是服务器组件，以 Linux 后台服务的方式运行。

Docker daemon 运行在 Docker host 上，负责创建、运行、监控容器，构建、存储镜像。



### 3.镜像-image

镜像：一系列文件、保存在本地   

联合文件系统   实现文件分层



#### 镜像的内部结构

###### base镜像： 

base 镜像有两层含义：

1. 不依赖其他镜像，从 scratch （0）构建。
2. 其他镜像可以之为基础进行扩展。

Linux 操作系统由内核空间和用户空间组成：

![](assert\LinuxOS.jpg)

内核空间是 kernel，Linux 刚启动时会加载 bootfs 文件系统，之后 bootfs 会被卸载掉。

用户空间的文件系统是 rootfs，包含我们熟悉的 /dev, /proc, /bin 等目录。

对于 base 镜像来说，底层直接用 Host 的 kernel，自己只需要提供 rootfs 就行了。





Docker 可以同时支持多种 Linux 镜像，模拟出多种操作系统环境。

![](assert\DockerHost_kernel.jpg)

Debian 和 BusyBox（一种嵌入式 Linux）上层提供各自的 rootfs，底层共用 Docker Host 的 kernel。



#### 可写的容器层

**只有容器层是可写的，容器层下面的所有镜像层都是只读的**。

所以**镜像可以被多个容器共享**。  分层好处：共享资源

![](assert\image.png)



如果多个容器共享一份基础镜像，当某个容器修改了基础镜像的内容，比如 /etc 下的文件，这时其他容器的 /etc 是否也会被修改？

不会！
修改会被限制在单个容器内。

只有当需要修改时才复制一份数据，这种特性被称作 Copy-on-Write。可见，容器层保存的是镜像变化的部分，不会对镜像本身进行任何修改。



##### 构建镜像 

##### （1）.docker commint

​    步骤：1.运行容器  

  eg：【在 ubuntu base 镜像中安装 vi 并保存为新镜像】

```shell
root@ziyonghong:~# docker run -it ubuntu  #-it 参数的作用是以交互模式进入容器，并打开终端。f31a2b11da9f是容器的内部 ID。
```

​		2.修改容器

```shell
root@f31a2b11da9f:/# vim
bash: vim: command not found
root@f31a2b11da9f:/# apt-get install -y vim #出现Unable to locate package,update一下就好啦
Reading package lists... Done
Building dependency tree
Reading state information... Done
E: Unable to locate package vim
root@f31a2b11da9f:/# apt-get update
```

​		3.将容器保存为新的镜像

docker ps查看当前运行的容器

NAMES下是Docker为我们的容器随机分配的名字

docker commit 将容器保存为镜像

docker  images 查看新镜像属性

![](assert\ubuntu-with-vim.png)





##### （2）Dockerfile

Dockerfile 是一个文本文件，记录了镜像构建的所有步骤。

用 Dockerfile 创建上节的 ubuntu-with-vi:

![](assert\upload-ueditor-image-dockerfile.jpg)



```shell
docker build -t ubuntu-with-vi-dockerfile .  
# 运行 docker build 命令，-t 将新镜像命名为 ubuntu-with-vi-dockerfile，命令末尾的 . 指明 build context 为当前目录。Docker 默认会从 build context 中查找 Dockerfile 文件，我们也可以通过 -f 参数指定 Dockerfile 的位置。
Sending build context to Docker daemon  1.856GB
#镜像真正的构建过程。 首先 Docker 将 build context 中的所有文件发送给 Docker daemon。build context 为镜像构建提供所需要的文件或目录。
#Dockerfile 中的 ADD、COPY 等命令可以将 build context 中的文件添加到镜像。此例中，build context 为当前目录 /root，该目录下的所有文件和子目录都会被发送给 Docker daemon。
#因为我的/root 目录下有好多文件所以都被发上去了。。不然应该只有几k。。 所以使用 build context 就得小心了，不要将多余文件放到 build context
Step 1/2 : FROM ubuntu   #1.执行 FROM，将 ubuntu 作为 base 镜像。
 ---> 94e814e2efa8       #ubuntu 镜像 ID 
Step 2/2 : RUN apt-get update && apt-get install -y vim  #2.执行 RUN，安装 vim
 ---> Running in db4f95fbbd48  # 启动ID为 db4f95fbbd48 的临时容器，在容器中通过 apt-get 安装 vim。
...
Processing triggers for libc-bin (2.27-3ubuntu1) ...
Removing intermediate container db4f95fbbd48 #删除临时容器 db4f95fbbd48。
 ---> 471638b66f9b   #安装成功后，将容器保存为镜像，其 ID 为 471638b66f9b。
               #重点:！！！【这一步底层使用的是类似 docker commit 的命令。】
Successfully built 471638b66f9b          #镜像构建成功。 
Successfully tagged ubuntu-with-vi-dockerfile:latest 
```

ubuntu-with-vi-dockerfile 是通过在 base 镜像的顶部添加一个新的镜像层而得到的。

这个新镜像层的内容由 `RUN apt-get update && apt-get install -y vim` 生成。bdocker history` 命令验证。

![](F:\Desktop\Typora\Docker\assert\history.png)

ubuntu-with-vi-dockerfile 与 ubuntu 镜像相比，确实只是多了顶部的一层471638b66f9b 。



Dockerfile 构建镜像的过程：

1. 从 base 镜像运行一个容器。
2. 执行一条指令，对容器做修改。
3. 执行类似 docker commit 的操作，生成一个新的镜像层。
4. Docker 再基于刚刚提交的镜像运行一个新容器。
5. 重复 2-4 步，直到 Dockerfile 中的所有指令执行完毕。



##### 镜像的缓存特性

Docker 会缓存已有镜像的镜像层，构建新镜像时，如果某镜像层已经存在，就直接使用，无需重新创建。

eg：在前面的 Dockerfile 中添加一点新内容，往镜像中复制一个文件：

![](assert\copy-testfilet.jpg)

```shell
root@ziyonghong:~#  docker build -t ubuntu-with-vi-dockerfile-2 .
Sending build context to Docker daemon  1.856GB
Step 1/3 : FROM ubuntu
 ---> 94e814e2efa8
Step 2/3 : RUN apt-get update && apt-get install -y vim
 ---> Using cache  #！！重点在这里：之前已经运行过相同的 RUN 指令，这次直接使用缓存中的镜像层471638b66f9b
 ---> 471638b66f9b
Step 3/3 : COPY testfile / #执行 COPY 指令。其过程是启动临时容器，复制 testfile，提交新的镜像层 0e489c054b91，删除临时容器。
 ---> 0e489c054b91
Successfully built 0e489c054b91
Successfully tagged ubuntu-with-vi-dockerfile-2:latest
```

在 ubuntu-with-vi-dockerfile 镜像上直接添加一层就得到了新的镜像 ubuntu-with-vi-dockerfile-2。

![](assert\copy.png)



Dockerfile 中每一个指令都会创建一个镜像层，上层是依赖于下层的。只要某一层发生变化，其上面所有层的缓存都会失效。

如果我们改变 Dockerfile 指令的执行顺序，或者修改或添加指令，都会使缓存失效。

![](assert\copy-testfile2.jpg)

由于分层的结构特性，Docker 必须重建受影响的镜像层。



#### debug Dockerfile

#####  

##### Dockerfile 中最常用的指令

**FROM**
指定 base 镜像。

**MAINTAINER**
设置镜像的作者，可以是任意字符串。

**COPY**
将文件从 build context 复制到镜像。
COPY 支持两种形式：

1. COPY src dest
2. COPY ["src", "dest"]

注意：src 只能指定 build context 中的文件或目录。

**ADD**
与 COPY 类似，从 build context 复制文件到镜像。不同的是，如果 src 是归档文件（tar, zip, tgz, xz 等），文件会被自动解压到 dest。

**ENV**
设置环境变量，环境变量可被后面的指令使用。例如：

...

ENV MY_VERSION 1.3

RUN apt-get install -y mypackage=$MY_VERSION

...



**EXPOSE**
指定容器中的进程会监听某个端口，Docker 可以将该端口暴露出来。我们会在容器网络部分详细讨论。

**VOLUME**
将文件或目录声明为 volume。我们会在容器存储部分详细讨论。

**WORKDIR**
为后面的 RUN, CMD, ENTRYPOINT, ADD 或 COPY 指令设置镜像中的当前工作目录。

**RUN**
RUN 执行命令并创建新的镜像层，RUN 经常用于安装软件包。

**CMD**
容器启动时运行指定的命令。
Dockerfile 中可以有多个 CMD 指令，但只有最后一个生效。CMD 可以被 docker run 之后的参数替换。

**ENTRYPOINT**
设置容器启动时运行的命令。
Dockerfile 中可以有多个 ENTRYPOINT 指令，但只有最后一个生效。CMD 或 docker run 之后的参数会被当做参数传递给 ENTRYPOINT。



##### 用 Docker Hub 存取镜像

```shell
root@ziyonghong:~# docker login -u ziyonghong #在 Docker Host 上登录
Password:
Login Succeeded
```

修改镜像的 repository 使之与 Docker Hub 账号匹配。
Docker Hub 为了区分不同用户的同名镜像，镜像的 registry 中要包含用户名，完整格式为：[username]/xxx:tag           docker tag重命名镜像

不然会出现denied: requested access to the resource is denied错误！

```shell
root@ziyonghong:~# docker tag python ziyonghong/python
root@ziyonghong:~# docker push ziyonghong/python
The push refers to repository [docker.io/ziyonghong/python]
bb839e9783c7: Mounted from library/python
237ce60325c6: Mounted from library/python
1b976700da1f: Mounted from library/python
bde41e1d0643: Mounted from library/python
7de462056991: Mounted from library/python
3443d6cf0f1f: Mounted from library/python
f3a38968d075: Mounted from library/python
a327787b3c73: Mounted from library/python
5bb0785f2eee: Mounted from library/python
latest: digest: sha256:00c463148cb09ee8f3d1fce1d35199e6e011d345855c207abbd072945b70991d size: 2218
```



##### 本地Docker Registry

```shell
root@ziyonghong:~# docker run -d -p 5000:5000 -v /myregistry:/var/lib/registry registry:2
Unable to find image 'registry:2' locally
2: Pulling from library/registry
c87736221ed0: Pull complete
1cc8e0bb44df: Pull complete
54d33bcb37f5: Pull complete
e8afc091c171: Pull complete
b4541f6d3db6: Pull complete
Digest: sha256:b224aa2d9a6397e9102b0b887a3e92496eadd76872efb7595bed97f9b76d2056
Status: Downloaded newer image for registry:2
ee32942c6acf5c7fb730c71b19d5cb7e01a423d979d3848a4bc93a5eddca63a9
```

  `d` 是后台启动容器。

`-p` 将容器的 5000 端口映射到 Host 的 5000 端口。5000 是 registry 服务端口。  

`-v` 将容器 /var/lib/registry 目录映射到 Host 的 /myregistry，用于存放镜像数据。

```shell
root@ziyonghong:~# docker tag zyh/ubuntu:v2 ziyonghong/ubuntu:v1
root@ziyonghong:~# docker images
REPOSITORY                    TAG                 IMAGE ID            CREATED             
ziyonghong/ubuntu             v1                  1d82dd41cc51        2 days ago          137MB
zyh/ubuntu                    v2                  1d82dd41cc51        2 days ago          137MB
```

 镜像名称由 repository 和 tag 两部分组成。而 repository 的完整格式为：[registry-host]:[port]/[username]/xxx

只有 Docker Hub 上的镜像可以省略 [registry-host]:[port] 。  



```shell
root@ziyonghong:~# docker tag zyh/ubuntu:v2 47.106.255.52:5050/ziyonghong/ubuntu:v1
root@ziyonghong:~# docker push 47.106.255.52:5050/ziyonghong/ubuntu:v1
The push refers to repository [47.106.255.52:5050/ziyonghong/ubuntu]
Get https://47.106.255.52:5050/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
```

出现request canceled while waiting for connection 错误！！因为在下载官方镜像点的镜像国内访问速度太慢！！

解决：使用加速器







镜像的常用操作子命令：

images    显示镜像列表

history   显示镜像构建历史

commit    从容器创建新镜像

build     从 Dockerfile 构建镜像

tag       给镜像打 tag

pull      从 registry 下载镜像

push      将 镜像 上传到 registry

rmi       删除 Docker host 中的镜像

search    搜索 Docker Hub 中的镜像

### 4.容器-Container

Docker 容器就是 Docker 镜像的运行实例。

##### 运行容器

```shell
root@ziyonghong:~# docker run -d ubuntu /bin/bash
8f2a7dae0052a54d61a97c2ae33ee6508a3004fd4ff45a36aa4e9bf832fb9938
```

`-d` 以后台方式启动容器。



##### 进入容器

```
docker attach <container> 
docker exec -it <container> bash|sh
```

attach 与 exec 主要区别如下:

1. attach 直接进入容器 **启动命令** 的终端，不会启动新的进程。
2. exec 则是在容器中打开新的终端，并且可以启动新的进程。
3. 如果想直接在终端中查看启动命令的输出，用 attach；其他情况使用 exec。

当然，如果只是为了查看启动命令的输出，可以使用 `docker logs` 命令



注：进入容器前一定要先运行run容器！！！



##### stop/start/restart /pause/unpause /rm容器

`docker rm` 一次可以指定多个容器，如果希望批量删除所有已经退出的容器，可以执行如下命令：

docker rm -v $(docker ps -aq -f status=exited)

`docker rm` 是删除容器，而 `docker rmi` 是删除镜像。



 				         ** 容器的生命周期*

![](assert\container-life.jpg)



#### 容器的底层实现技术

cgroup 实现资源限额， namespace 实现资源隔离。

##### cgroup

Linux 操作系统通过 cgroup 可以设置进程使用 CPU、内存 和 IO 资源的限额。--cpu-shares`、`-m`、`--device-write-bps` 实际上就是在配置 cgroup。

##### namespace

在每个容器中，我们都可以看到文件系统，网卡等资源，这些资源看上去是容器自己的。拿网卡来说，每个容器都会认为自己有一块独立的网卡，即使 host 上只有一块物理网卡。这种方式非常好，它使得容器更像一个独立的计算机。

###### Mount namespace 

Mount namespace 让容器看上去拥有整个文件系统。

容器有自己的 `/` 目录，可以执行 `mount` 和 `umount` 命令。当然我们知道这些操作只在当前容器中生效，不会影响到 host 和其他容器。

###### UTS namespace

让容器有自己的 hostname。 默认情况下，容器的 hostname 是它的短ID，可以通过 `-h` 设置。

###### IPC namespace

 让容器拥有自己的共享内存和信号量（semaphore）来实现进程间通信，而不会与 host 和其他容器的 IPC 混在一起。

###### PID namespace

容器在 host 中以进程的形式运行。

通过 `ps axf` 可以查看容器进程。

###### Network namespace

让容器拥有自己独立的网卡、IP、路由等资源。

###### User namespace 

让容器能够管理自己的用户，host 不能看到容器中创建的用户。



### 5.仓库-Registry

Registry 是存放 Docker 镜像的仓库。

先将我的image传到Docker仓库，再从仓库pull下来

Docker Hub（[https://hub.docker.com/）](https://hub.docker.com/%EF%BC%89) 是默认的 Registry



`docker pull` 命令可以从 Registry 下载镜像。
`docker run` 命令则是先下载镜像（如果本地没有），然后再启动容器。

```
docker pull [OPTIONS] NAME[:TAG] //拉取镜像 到docker仓库下载下来的
docker images [OPTIONS]   //查看镜像

docker run [OPTIONS] IMAGE[:TAG] [COMMAN][ARG...] //运行

```

`docker images` 可以查看到 httpd 已经下载到本地。

docker ps` 或者 `docker container ls` 显示容器正在运行。

![](assert\Docker.png) 						Docker 架构图（Client/Server 架构）



## 三、Docker网络

![](assert\docker_net.png)

`docker network ls` 

#### none 网络

```shell
root@ziyonghong:~# docker run -it --network=none busybox
/ # ifconfig
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

/ # exit
```

挂在这个网络下的容器除了 lo，没有其他任何网卡。

应用：某个容器的唯一用途是生成随机密码，就可以放到 none 网络中避免密码被窃取。

#### 

#### host网络

​	Host   容器不会配置自己的网卡等，使用宿主机上的IP和端口

```shell
root@ziyonghong:~# docker run -it --network=host busybox
/ # ip l
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether 00:16:3e:14:93:2e brd ff:ff:ff:ff:ff:ff
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue
    link/ether 02:42:19:75:1a:5d brd ff:ff:ff:ff:ff:ff
86: docker_gwbridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue
    link/ether 02:42:96:fd:6b:3a brd ff:ff:ff:ff:ff:ff
150: vethf00ba39@if149: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue master docker_gwbridge
    link/ether e6:43:9d:b9:07:db brd ff:ff:ff:ff:ff:ff
154: veth1830030@if153: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue master docker0
    link/ether 7e:cf:0c:b1:b7:a5 brd ff:ff:ff:ff:ff:ff
/ # hostname
ziyonghong
```

连接到 host 网络的容器共享 Docker host 的网络栈，容器的网络配置与 host 完全一样。

但会有端口冲突，Docker host 上已经使用的端口就不能再用了。



#### bridge 网络

​       Bridge   端口映射

```shell
root@ziyonghong:~# brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024219751a5d       no              veth1830030
docker_gwbridge         8000.024296fd6b3a       no              vethf00ba39
root@ziyonghong:~# docker run -d httpd
f01d19c92f918e15d1beb2126121ead2abdfe500748c63b18644425de640f1eb
root@ziyonghong:~# brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024219751a5d       no              veth00aa2e3
                                                        veth1830030
docker_gwbridge         8000.024296fd6b3a       no              vethf00ba39
#一个新的网络接口veth00aa2e3 被挂到了 docker0 上，veth00aa2e3就是新创建容器的虚拟网卡。
root@ziyonghong:~# docker exec -it f01d19c92f918e15d1beb2 bash
root@f01d19c92f91:/usr/local/apache2# ip a
bash: ip: command not found
root@f01d19c92f91:/usr/local/apache2# exit
exit
root@ziyonghong:~# docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "ecf6949e3755397beb056885444d07b96613ba5f3ce43ed4f30d0035aeff9076",
        "Created": "2019-03-27T14:58:06.843854114+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
]
root@ziyonghong:~# ifconfig docker0
docker0   Link encap:Ethernet  HWaddr 02:42:19:75:1a:5d
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:23526 errors:0 dropped:0 overruns:0 frame:0
          TX packets:40275 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1346275 (1.3 MB)  TX bytes:116023441 (116.0 MB)

```

容器创建时，docker 会自动从 172.17.0.0/16 中分配一个 IP，这里 16 位的掩码保证有足够多的 IP 可以供容器使用。



## Docker运行nginx静态网站

```shell
root@ziyonghong:~# docker pull hub.c.163.com/library/nginx:latest
latest: Pulling from library/nginx
5de4b4d551f8: Pull complete
d4b36a5e9443: Pull complete
0af1f0713557: Pull complete
Digest: sha256:f84932f738583e0169f94af9b2d5201be2dbacc1578de73b09a6dfaaa07801d6
Status: Downloaded newer image for hub.c.163.com/library/nginx:latest
root@ziyonghong:~# docker run hub.c.163.com/library/nginx
^Croot@ziyonghong:~# docker run -d hub.c.163.com/library/nginx
a9a77c825538ac57b8d051553b79971f690f8507ff32d656d6535e6315cb1af2
root@ziyonghong:~# docker exec -it a9a77 bash
root@a9a77c825538:/# ls
bin  boot  dev  etc  home  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@a9a77c825538:/# which nginx
/usr/sbin/nginx
root@a9a77c825538:/# ps -ef
bash: ps: command not found
root@a9a77c825538:/# exit
exit


docker stop <CONTAINER ID>

```



## 四、容器通信方式

#### 1.容器间的通信

##### IP通信

在容器创建时通过 `--network` 指定相应的网络，或者通过 `docker network connect` 将现有容器加入到指定网络。

##### Docker DNS Server

```shell
root@ziyonghong:~# docker run -it --network=my_net2 --name=bbox1 busybox
/ # ping -c 3 bbox2
PING bbox2 (172.22.16.3): 56 data bytes
64 bytes from 172.22.16.3: seq=0 ttl=64 time=0.101 ms
64 bytes from 172.22.16.3: seq=1 ttl=64 time=0.106 ms
64 bytes from 172.22.16.3: seq=2 ttl=64 time=0.105 ms

--- bbox2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.101/0.104/0.106 ms
/ # exit
root@ziyonghong:~# docker run -it --name=bbox4 busybox
/ # ping -c 3 bbox3
ping: bad address 'bbox3'
```

使用 docker DNS 有个限制：**只能在 user-defined 网络中使用**。默认的 bridge 网络是无法使用 DNS 的。上面的ping -c 3 bbox3就不能ping通了

##### joined容器

使两个或多个容器共享一个网络栈（网卡 mac 地址与 IP 完全一样），共享网卡和配置信息，joined 容器之间可以通过 127.0.0.1 直接通信。

```shell
root@ziyonghong:~# docker run -d -it --name=web1 httpd #创建一个 httpd 容器，名字为 web1
37d61b7b79fbb5bec6fa3b2c976863d85d5c1abc6d26681e1a100ccc2ec04acb
root@ziyonghong:~# docker run -it --network=container:web1 busybox #创建 busybox 容器并通过 --network=container:web1 指定 jointed 容器为 web1
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
185: eth0@if186: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:11:00:06 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.6/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
/ # wget 127.0.0.11
Connecting to 127.0.0.11 (127.0.0.11:80)
index.html           100% |*****************************************************************************************|    45  0:00:00 ETA
/ # cat index.html
<html><body><h1>It works!</h1></body></html>
```

joined 容器适合场景：

1. 不同容器中的程序希望通过 loopback 高效快速地通信，比如 web server 与 app server。
2. 希望监控其他容器的网络流量，比如运行在独立容器中的网络监控程序。



#### 2.容器与外部世界通信

##### 容器访问外部世界

**容器默认就能访问外网**。

```shell
root@ziyonghong:~# ping -c 3 www.baidu.com
PING www.a.shifen.com (14.215.177.39) 56(84) bytes of data.
64 bytes from 14.215.177.39: icmp_seq=1 ttl=52 time=7.73 ms
64 bytes from 14.215.177.39: icmp_seq=2 ttl=52 time=7.79 ms
64 bytes from 14.215.177.39: icmp_seq=3 ttl=52 time=7.78 ms

--- www.a.shifen.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 7.737/7.771/7.791/0.104 ms
root@ziyonghong:~# docker run -it busybox
/ # ping -c 3 baidu.com
PING baidu.com (123.125.114.144): 56 data bytes
64 bytes from 123.125.114.144: seq=0 ttl=50 time=40.444 ms
64 bytes from 123.125.114.144: seq=1 ttl=50 time=40.984 ms
64 bytes from 123.125.114.144: seq=2 ttl=50 time=40.851 ms

--- baidu.com ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 40.444/40.759/40.984 ms
```

busybox 位于 `docker0` 这个私有 bridge 网络中（172.17.0.0/16），当 busybox 从容器向外 ping 时，数据包是怎样到达 baidu.com 的呢？

这里的关键就是 NAT。

查看一下 docker host 上的 iptables 规则：

```shell
root@ziyonghong:~# iptables -t nat -S
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N DOCKER
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
-A POSTROUTING -s 172.22.16.0/24 ! -o br-a12ab58c9d98 -j MASQUERADE
-A POSTROUTING -s 172.20.0.0/16 ! -o br-4aeb549a5a7d -j MASQUERADE
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
-A POSTROUTING -s 172.19.0.0/16 ! -o docker_gwbridge -j MASQUERADE
-A POSTROUTING -s 172.17.0.4/32 -d 172.17.0.4/32 -p tcp -m tcp --dport 80 -j MASQUERADE
-A DOCKER -i br-a12ab58c9d98 -j RETURN
-A DOCKER -i br-4aeb549a5a7d -j RETURN
-A DOCKER -i docker0 -j RETURN
-A DOCKER -i docker_gwbridge -j RETURN
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 80 -j DNAT --to-destination 172.17.0.4:80
```

在 NAT 表中，有这么一条规则：

-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE

来自 172.17.0.0/16 网段的包，目标地址是外网（! -o docker0），就把它交给 MASQUERADE 处理。

MASQUERADE 的处理方式是将包的源地址替换成 host 的地址发送出去，即做了一次网络地址转换（NAT）。

![](assert\iptable_NAT.jpg)

1. busybox 发送 ping 包：172.17.0.2 > www.bing.com。
2. docker0 收到包，发现是发送到外网的，交给 NAT 处理。
3. NAT 将源地址换成 enp0s3 的 IP：10.0.2.15 > www.bing.com。
4. ping 包从 enp0s3 发送出去，到达 www.bing.com。





##### 外部世界访问容器

**端口映射**

```shell
root@ziyonghong:~# docker run -d -p 80 httpd  -p 中指定映射到 host 某个特定端口，如-p 8080：80
d440dbe663fad0c3354d2e129fecb21eaf6f612b693fa76405c033a6ba6dae1f
root@ziyonghong:~# docker port d440dbe663fad0c
80/tcp -> 0.0.0.0:32768 #httpd 容器的 80 端口被映射到 host32768
#可以通过 <host ip>:<32773> 访问容器的 web 服务了
root@ziyonghong:~# curl 172.18.115.79:32768
<html><body><h1>It works!</h1></body></html>
#每一个映射的端口，host 都会启动一个 docker-proxy 进程来处理访问容器的流量：
root@ziyonghong:~# ps -ef|grep docker-proxy
root     19399 30169  0 Mar27 ?        00:00:00 /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 32768 -container-ip 172.17.0.5 -container-port 80

```



![](assert\Network.jpg)

1. docker-proxy 监听 host 的 32773 端口。
2. 当 curl 访问 10.0.2.15:32773 时，docker-proxy 转发给容器 172.17.0.3:80。
3. httpd 容器响应请求并返回结果。



## 五、Docker存储

#### 1.由storage driver 管理的镜像层和容器层

容器由最上面一个可写的容器层，以及若干只读的镜像层组成，容器的数据就存放在这些层中。这样的分层结构最大的特性是 Copy-on-Write：

1. 新数据会直接存放在最上面的容器层。
2. 修改现有数据会先从镜像层将数据复制到容器层，修改后的数据直接保存在容器层中，镜像层保持不变。
3. 如果多个层中有命名相同的文件，用户只能看到最上面那层中的文件。

Docker 支持多种 storage driver，有 AUFS、Device Mapper、Btrfs、OverlayFS、VFS 和 ZFS。它们都能实现分层的架构，同时又有各自的特性。

运行`docker info`可查看 

```shell
root@ziyonghong:~# docker info
Containers: 66
 Running: 5
 Paused: 0
 Stopped: 61
Images: 38
Server Version: 18.09.3
Storage Driver: overlay2  
 Backing Filesystem: extfs  #底层文件系统
```

对于**无状态的应用**（无状态意味着容器没有需要持久化的数据，随时可以从镜像直接创建）直接将数据放在由 storage driver 维护的层中。

 比如 busybox，它是一个工具箱，启动 busybox 是为了执行如 wget，ping 之类的命令，不需要保存数据供以后使用，使用完直接退出，容器删除时存放在容器层中的工作数据也一起被删除，这没问题，下次再启动新容器即可。



#### 2.Data Volume

Data Volume 本质上是 Docker Host 文件系统中的目录或文件，能够直接被 mount 到容器的文件系统中。

Data Volume特点：

1. Data Volume 是目录或文件，而非没有格式化的磁盘（块设备）。
2. 容器可以读写 volume 中的数据。
3. volume 数据可以被**永久的保存**，即使使用它的容器已经销毁。



docker 提供了两种类型的 volume：

##### （1）bind mount 

bind mount 是将 host 上已存在的目录或文件 mount 到容器。

![](assert\mount.jpg)

`-v` 的格式为 `<host path>:<container path>`。/usr/local/apache2/htdocs 就是 apache server 存放静态文件的地方。由于 /usr/local/apache2/htdocs 已经存在，原有数据会被隐藏起来，取而代之的是 host $HOME/htdocs/ 中的数据。

bind mount 可以让 host 与容器共享数据。

即使容器没有了，bind mount 也还在。因为bind mount 是 host 文件系统中的数据，只是借给容器用用而已啦。

bind mount 的使用直观高效，但有不足：bind mount 需要指定 host 文件系统的特定路径，这就限制了容器的可移植性。

##### （2）docker managed volume

docker managed volume 与 bind mount 在使用上的最大区别是不需要指定 mount 源，指明 mount point 就行了。更好的可移植性

![](assert\managed_volume.jpg)

`-v` 告诉 docker 需要一个 data volume，并将其 mount 到 /usr/local/apache2/htdocs。那么这个 data volume 具体在哪儿呢？

执行docker inspect 21accc2ca072...

```shell
"Mounts": [

    {

        "Name": "f4a0a1018968f47960efe760829e3c5738c702533d29911b01df9f18babf3340",

        "Source": "/var/lib/docker/volumes/f4a0a1018968f47960efe760829e3c5738c702533d29911b01df9f18babf3340/_data",

        "Destination": "/usr/local/apache2/htdocs",

        "Driver": "local",

        "Mode": "",

        "RW": true,

        "Propagation": ""

    }
    ],
```

`在Mounts` 这部分，会显示容器当前使用的所有 data volume，包括 bind mount 和 docker managed volume。  Source` 就是该 volume 在 host 上的目录。

每当容器申请 mount docker manged volume 时，docker 都会在`/var/lib/docker/volumes` 下生成一个目录（例子中是 "/var/lib/docker/volumes/f4a0a1018968f47960efe760829e3c5738c702533d29911b01df9f18babf3340/_data ），这个目录就是 mount 源。

docker managed volume 的创建过程：

1. 容器启动时，简单的告诉 docker "我需要一个 volume 存放数据，帮我 mount 到目录 /abc"。
2. docker 在 /var/lib/docker/volumes 中生成一个随机目录作为 mount 源。
3. 如果 /abc 已经存在，则将数据复制到 mount 源，
4. 将 volume mount 到 /abc。

![](assert\volume_inspect.jpg)



两种Data Volume对比：

1. 相同点：两者都是 host 文件系统中的某个路径。

2. 不同点：

   ![](assert\data_volume.png)



## 六、共享数据 volume

#### 1.**容器与 host 共享数据**

bind mount ：直接将要共享的目录 mount 到容器。

docker managed volume : volume 位于 host 中的目录，是在容器启动时才生成，所以需要将共享数据拷贝到 volume 中.

docker cp` 可以在容器和 host 之间拷贝数据.



#### 2.**容器之间共享数据**

######  (1)将共享数据放在 bind mount 中，然后将其 mount 到多个容器。

###### (2)volume container  :**实现了容器与 host 的解耦**。

![](assert\volume_container.jpg)



![](assert\volume_container2.jpg)



###### (3)data-packed volume container:原理是将数据打包到镜像中，然后通过 docker managed volume 共享。



## 七、Docker Machine

Docker Machine 可以批量安装和配置 docker host.

![](assert\docker_machine.jpg)

##### 安装Docker Machine

```shell
curl -L https://github.com/docker/machine/releases/download/v0.9.0/docker-machine-`uname -s`-`uname -m` >/tmp/docker-machine &&

  chmod +x /tmp/docker-machine &&

  sudo cp /tmp/docker-machine /usr/local/bin/docker-machine
```

下载的执行文件被放到 /usr/local/bin 中，执行`docker-mahine version`查看



装 bash completion script，这样在 bash 能够通过 `tab` 键补全 `docker-mahine` 的子命令和参数。安装方法是从<https://github.com/docker/machine/tree/master/contrib/completion/bash>下载 completion script：

将其放置到 `/etc/bash_completion.d` 目录下。然后将如下代码添加到`$HOME/.bashrc`：

```
PS1='[\u@\h \W$(__docker_machine_ps1)]\$ '
```



##### 创建Docker Machine

`Machine` 就是运行 docker daemon 的主机。“创建 Machine” 指的就是在 host 上安装和部署 docker。

 machine 要求能够无密码登录远程主机，所以需要先通过如下命令将 ssh key 拷贝到 192.168.56.104：  ssh-copy-id 192.168.56.104 

![](assert\docker_machine_create.jpg)

① 通过 ssh 登录到远程主机。
② 安装 docker。
③ 拷贝证书。
④ 配置 docker daemon。
⑤ 启动 docker。

可以登录到 host1 查看 docker daemon 的具体配置 /etc/systemd/system/docker.service。

![](assert\host.jpg)

1. `-H tcp://0.0.0.0:2376` 使 docker daemon 接受远程连接。
2. `--tls*` 对远程连接启用安全认证和加密。



##### 管理 Machine

之前要执行远程 docker 命令我们需要通过 `-H` 指定目标主机的连接字符串，比如：  docker -H tcp://192.168.56.105:2376 ps

Docker Machine 则让这个过程更简单。

docker-machine env host1`显示访问 host1 需要的所有环境变量。

执行 `eval $(docker-machine env host1)`：

![](assert\env_host1.jpg)

可以看到命令行提示符已经变了，原因是在`$HOME/.bashrc` 中配置了 `PS1='[\u@\h \W$(__docker_machine_ps1)]\$ '`，用于显示当前 docker host。

在此状态下执行的所有 docker 命令其效果都相当于在 host1 上执行。

`docker-machine scp` 可以在不同 machine 之间拷贝文件，比如：

```
docker-machine scp host1:/tmp/a host2:/tmp/b
```



## 八、容器网络

![](assert\docker_net.jpg)

如此众多的方案是如何与 docker 集成在一起的？**libnetwork & CNM**

libnetwork 是 docker 容器网络库，最核心的内容是其定义的 Container Network Model (CNM)，由三类组件组成：

**Sandbox**

Sandbox 是容器的网络栈，包含容器的 interface、路由表和 DNS 设置。 Linux Network Namespace 是 Sandbox 的标准实现。Sandbox 可以包含来自不同 Network 的 Endpoint。

**Endpoint**

Endpoint 的作用是将 Sandbox 接入 Network。Endpoint 的典型实现是 veth pair，后面我们会举例。一个 Endpoint 只能属于一个网络，也只能属于一个 Sandbox。

**Network**

Network 包含一组 Endpoint，同一 Network 的 Endpoint 可以直接通信。Network 的实现可以是 Linux Bridge、VLAN 等。

![](assert\CNM.jpg)







## 九、Docker  存储

#### 1.跨Docker主机存储

容器可以分为两类：无状态（stateless）容器和有状态（stateful）容器。

状态（state）就是数据，如果容器需要处理并存储数据，它就是有状态的（数据库服务器），反之则无状态（提供静态页面的 web 服务器）。



volume 其本质是 Docker 主机 **本地** 的目录。那如果 Docker Host 宕机了，如何恢复容器？



![](assert\Storage_Provider.jpg)



当 Host1 发生故障，在 Host2 上启动相同的 MySQL 镜像，并挂载 data volume。

![](assert\Storage_Provider2.jpg)



Docker 是如何实现这个跨主机管理 data volume 方案的呢？ **volume driver。**

实践Rex-Ray driver。

Rex-Ray 以 standalone 进程的方式运行在 Docker 主机上

```
curl -sSL https://dl.bintray.com/emccode/rexray/install | sh -  #报错了
```





## 十、Docker 监控方案

![](assert\Docker_ps.png)

Docker 自带的几个监控子命令：ps（`docker container ps` ）, top 和 stats。功能更强的开源监控工具 sysdig, Weave Scope, cAdvisor 和 Prometheus。

##### sysdig

安装和运行 sysdig 的最简单方法是运行 Docker 容器。

```shell
docker container run -it --rm --name=sysdig --privileged=true \

          --volume=/var/run/docker.sock:/host/var/run/docker.sock \

          --volume=/dev:/host/dev \

          --volume=/proc:/host/proc:ro \

          --volume=/boot:/host/boot:ro \

          --volume=/lib/modules:/host/lib/modules:ro \

          --volume=/usr:/host/usr:ro \

          sysdig/sysdig
```

`docker container exec -it sysdig bash`  #进入容器

执行 **`csysdig**` 命令，将以交互方式启动 sysdig。



点击底部 `Views` 菜单（或者按 F2），显示 View 选择列表。

左边列出了sysdig支持的View。将光标移到 `Containers`这一项，回车或者双击 `Containers`，进入容器监控界面。

sysdig 会显示该 Host 所有容器的实时数据，每两秒刷新一次。各列数据的含义也是自解释的，如果不清楚，可以点一下底部 `Legend`（或者按 F7）。如果想按某一列排序，比如按使用的内存量，很简单，点一下列头 `VIRT`。



##### Weave Scope

会自动生成一张 Docker 容器地图，除了监控容器，它还可以监控 Docker Host。和多主机监控



##### cAdvisor

cAdvisor 会显示当前 host 的资源使用情况，包括 CPU、内存、网络、文件系统等。



##### Prometheus

Prometheus 提供了监控数据搜集、存储、处理、可视化和告警一套完整的解决方案。

![](assert\Prometheus.png)

## 

## 十、Docker logs

![](assert\Docker_logs.png)



##### **1,Docker logs**

对于一个运行的容器，Docker 会将日志发送到 **容器的** 标准输出设备（STDOUT）和标准错误设备（STDERR），STDOUT 和 STDERR 实际上就是容器的控制台终端。

1. attach 到该容器。 

2. 用 `docker logs` 命令查看日志。

   ```shell
   root@ziyonghong:~# docker ps
   CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              POR        TS                NAMES
   5f94e2f5fc99        httpd               "httpd-foreground"       About a minute ago   Up About a minute   0.0        .0.0:80->80/tcp   affectionate_nash
   a962b9dede41        sysdig/sysdig       "/docker-entrypoint.…"   16 hours ago         Up 16 hours                                      sysdig
   root@ziyonghong:~# docker attach 5f94e2f5fc99
   [Tue Apr 02 05:15:56.074226 2019] [mpm_event:notice] [pid 1:tid 139991325720640] AH00491: caught SIGTERM, shu        tting down
   root@ziyonghong:~# docker logs -f 5f94e2f5fc99  #-f 参数可以继续打印出新产生的日志
   AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 172.17.0.3. Set         the 'ServerName' directive globally to suppress this message
   AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 172.17.0.3. Set         the 'ServerName' directive globally to suppress this message
   [Tue Apr 02 05:13:32.402028 2019] [mpm_event:notice] [pid 1:tid 139991325720640] AH00489: Apache/2.4.38 (Unix        ) configured -- resuming normal operations
   [Tue Apr 02 05:13:32.402188 2019] [core:notice] [pid 1:tid 139991325720640] AH00094: Command line: 'httpd -D         FOREGROUND'
   [Tue Apr 02 05:15:56.074226 2019] [mpm_event:notice] [pid 1:tid 139991325720640] AH00491: caught SIGTERM, shu        tting down
   ```

   

   Docker 提供了多种日志机制帮助用户从运行的容器中提取日志信息。这些机制被称作 logging driver。     Docker 的默认 logging driver 是 `json-file`。

   查看日志文件：

   ```shell
   cat /var/lib/docker/containers/<contariner ID>/<contariner ID>-json.log
   ```

   除了 `json-file`，Docker 还支持多种 logging driver。完整列表可访问官方文档 <https://docs.docker.com/engine/admin/logging/overview/#supported-logging-drivers>

   **2.EKL**

   ![](assert\EKL.png)

   Logstash 负责从各个 Docker 容器中提取日志，Logstash将日志转发到 Elasticsearch 进行索引和保存，Kibana 分析和可视化数据。,,```shell,docker run -p 5601:5601 -p 9200:9200 -p 5044:5044 -it --name elk sebp/elk,#5601 - Kibana web 接口,#9200 - Elasticsearch JSON 接口,#5044 - Logstash 日志接收接口,```







## 容器平台技术

![](F:\Desktop\Typora\Docker\assert\容器平台技术.png)



#### 1.Docker Swarm

Docker Swarm 管理的是 Docker Host 集群。

一个集群和一堆服务器最显著的区别在于：集群能够像 **单个** 系统那样工作，同时提供高可用、负载均衡和并行处理。



##### Docker Swarm Mode

Docker Swarm 的功能已经完全与 Docker Engine 集成，要管理集群，只需要启动 Swarm Mode。



**swarm**

swarm 运行 Docker Engine 的多个主机组成的集群。

没启动 swarm mode 时，Docker 执行的是容器命令；运行 swarm mode 后，Docker 增加了编排 service 的能力。

Docker 允许在同一个 Docker 主机上既运行 swarm service，又运行单独的容器。

**node**

swarm 中的每个 Docker Engine 都是一个 node，有两种类型的 node：**manager** 和 **worker**。

manager node 负责执行编排和集群管理工作，保持并维护 swarm 处于期望的状态。

woker node 接受并执行由 manager node 派发的任务。

**service**

service 定义了 worker node 上要执行的任务。swarm 的主要编排任务就是保证 service 处于期望的状态下。