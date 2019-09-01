 

# Docker

## 一、Docker镜像使用

### 1.管理和使用本地Docker主机镜像

##### 查看镜像列表

decker images

![](F:\Desktop\Typora\Docker\assert\images.png)

 选项说明：

- **REPOSITORY：**表示镜像的仓库源
- **TAG：**镜像的标签
- **IMAGE ID：**镜像ID
- **CREATED：**镜像创建时间
- **SIZE：**镜像大小

同一仓库源可以有多个 TAG，代表这个仓库源的不同个版本，如ubuntu仓库源里，有15.10、14.04等多个不同的版本，我们使用 REPOSITORY:TAG 来定义不同的镜像。

使用版本为15.10的ubuntu系统镜像来运行容器：

```shell
root@ziyonghong:~# docker run -t -i ubuntu:15.10 /bin/bash
root@a125e2e99960:/#
```

不指定一个镜像的版本标签，例如你只使用 ubuntu，docker 将默认使用 ubuntu:latest 镜像。

退出：exit

![1553480429193](F:\Desktop\Typora\Docker\assert\exit.png)



##### 获取一个新镜像

在本地主机上使用一个不存在的镜像时 Docker 就会自动下载这个镜像。如果我们想预先下载这个镜像，我们可以使用 docker pull 命令来下载它。

```shell
root@ziyonghong:~# docker pull ubuntu:13.10
```



##### 查找镜像

从 Docker Hub 网站来搜索镜像，Docker Hub 网址为： *https://hub.docker.com/*

也可以使用 docker search 命令来搜索镜像。

```shell
root@ziyonghong:~# docker search httpd
```

![](F:\Desktop\Typora\Docker\assert\httpd.png)



**NAME:**镜像仓库源的名称

**DESCRIPTION:**镜像的描述

**OFFICIAL:**是否docker官方发布

##### 拖取镜像

```shell
root@ziyonghong:~# docker pull httpd
```

下载完就可以run啦

```shell
root@ziyonghong:~# docker run httpd
```

### 2.创建镜像

##### 两种更改镜像方式:

- 1.从已经创建的容器中更新镜像，并且提交这个镜像
- 2.使用 Dockerfile 指令来创建一个新的镜像



##### 更新镜像

更新前，需要使用镜像来创建一个容器.

```
root@ziyonghong:~# docker run -t -i ubuntu:15.10 /bin/bash
root@02fc63e8cdf7:/# apt-get update #更新
```

![](F:\Desktop\Typora\Docker\assert\update.png)

完成后一定要记得exit退出！！！



ID为02fc63e8cdf7的容器，就是更改的容器。

通过命令 docker commit来提交容器副本。

```shell
root@ziyonghong:~# docker commit -m="has update" -a="ziyonghong" 02fc63e8cdf7 zyh/ubuntu:v2
sha256:1d82dd41cc51c5721a4c678007a8444c262b2c3d185be8ad5dd8eeb788e265ad
```

- **-m:**提交的描述信息
- **-a:**指定镜像作者
- **02fc63e8cdf7：**容器ID
- **zyh/ubuntu:v2:**指定要创建的目标镜像名

 **docker images** 命令来查看新镜像 .



使用新镜像 **zyh/ubuntu** 来启动一个容器

```shell
root@ziyonghong:~# docker run -it zyh/ubuntu:v2 /bin/bash
root@6599cedd05c7:/#
```

- **-t:**在新容器内指定一个伪终端或终端。
- **-i:**允许你对容器内的标准输入 (STDIN) 进行交互。



##### 构建镜像

使用命令 **docker build** ， 从零开始来创建一个新的镜像。

先要创建一个 Dockerfile 文件，包含一组指令来告诉 Docker 如何构建我们的镜像。

```shell
root@ziyonghong:~# cat Dockerfile
cat: Dockerfile: No such file or directory  #没有Dockerfile目录
```

  没有Dockerfile目录不能直接cat  可以cat>>Dockerfile ">>"重定向  或者是touch、mkdir先创建后 vim进入更改。

```shell
root@ziyonghong:~# cat Dockerfile

FROM ubuntu:15.10
MAINTAINER Zyh "1131002466@qq.com"

RUN   /bin/echo 'root:123456' |chpasswd
RUN   useradd zyh
RUN   /bin/echo 'zyh:123456' |chpasswd
RUN   /bin/echo -e "LANG=\"en_US.UTF-8\"" >/etc/default/local
EXPOSE 22
EXPOSE 80
CMD   /usr/sbin/sshd -D
```

每一个指令都会在镜像上创建一个新的层，每一个指令的前缀都必须是大写。

第一条FROM，指定使用哪个镜像源。

RUN 指令告诉docker 在镜像内执行命令，安装了什么。。。

然后使用 Dockerfile 文件，通过 docker build 命令来构建一个镜像。

```shell
root@ziyonghong:~# docker build -t zyh/ubuntu:15.10 .
Sending build context to Docker daemon  1.842GB
.....
```

![](F:\Desktop\Typora\Docker\assert\bulid.png)

- **-t** ：指定要创建的目标镜像名
- **.** ：Dockerfile 文件所在目录，可以指定Dockerfile 的绝对路径 (忘记打了会报错！！！)

使用docker images 查看创建的镜像已经在列表中存在,镜像ID为8f0639c030dc.

![](F:\Desktop\Typora\Docker\assert\images2.png)

就可以使用新的镜像来创建容器啦

```shell
root@ziyonghong:~# docker run -it zyh/ubuntu:v2 /bin/bash
root@60961ba5cda0:/# id zyh
id: zyh: no such user
root@60961ba5cda0:/# id root
uid=0(root) gid=0(root) groups=0(root)
```

然而新镜像并没有包含我创建的用户zyh（我也不知道为啥 இ௰இ



##### 设置镜像标签

使用 docker tag 命令，为镜像添加一个新的标签。

```shell
root@ziyonghong:~# docker tag 8f0639c030dc zyh/ubuntu:dev
root@ziyonghong:~# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
zyh/ubuntu          15.10               8f0639c030dc        21 minutes ago      138MB
zyh/ubuntu          dev                 8f0639c030dc        21 minutes ago      138MB
zyh/ubuntu          v2                  1d82dd41cc51        42 minutes ago      137MB
httpd               latest              2d1e5208483c        2 weeks ago         132MB
hello-world         latest              fce289e99eb9        2 months ago        1.84kB
ubuntu              15.10               9b9cb95443b5        2 years ago         137MB
training/webapp     latest              6fae60ef3446        3 years ago         349MB
root@ziyonghong:~#
```

docker tag 镜像ID，这里是 8f0639c030dc  ,用户名称、镜像源名(repository name)和新的标签名(tag)。

使用 docker images 命令可以看到，ID为8f0639c030dc的镜像多一个标签dev。









## 二、Docker容器连接





