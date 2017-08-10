# 杂学札记之大数据系列：使用Docker编译64位的Hadoop

### 目录索引

[前言：为什么要编译64位的Hadoop](#前言为什么要编译64位的hadoop)

[一、制作CentOS7基础镜像](#一制作centos7基础镜像)

[二、使用Dockerfile制作CentOS7环境下的编译镜像](#二使用dockerfile制作centos7环境下的编译镜像)

[三、使用Docker镜像编译Hadoop](#三使用docker镜像编译hadoop)

[附：命令行纯净版](#附命令行纯净版)



### 前言：为什么要编译64位的Hadoop

（简单来点介绍，来自百度百科）

Hadoop是一个由Apache基金会所开发的分布式系统基础架构。

用户可以在不了解分布式底层细节的情况下，开发分布式程序。充分利用集群的威力进行高速运算和存储

Hadoop实现了一个分布式文件系统（Hadoop Distributed File System），简称HDFS。HDFS有高容错性的特点，并且设计用来部署在低廉的（low-cost）硬件上；而且它提供高吞吐量（high throughput）来访问应用程序的数据，适合那些有着超大数据集（large data set）的应用程序。HDFS放宽了（relax）POSIX的要求，可以以流的形式访问（streaming access）文件系统中的数据。

Hadoop的框架最核心的设计就是：HDFS和MapReduce。HDFS为海量的数据提供了存储，则MapReduce为海量的数据提供了计算。

**但是，Hadoop只提供了32位的编译版本，而据说此编译版本用在64位的系统上就会报错，所以想在64位系统上学习下Hadoop先得学自己编译。**

本人想在CentOS7上使用Docker做集群，系统是最小安装的，而编译Hadoop需要安装一些依赖，为了不安装过多的东西而污染本地环境，所以参考了[kiwenlau](https://github.com/kiwenlau)/[**compile-hadoop**](https://github.com/kiwenlau/compile-hadoop)使用Docker容器内的环境来编译Hadoop，不过是自己新制作的编译用的镜像，下面为具体内容。



### 一、制作CentOS7基础镜像

Docker Hub上已经提供了[CentOS7的官方镜像](https://hub.docker.com/_/centos/)，不过根据官方说明这个镜像中的CentOS7并未激活Systemd（用来启动守护进程,已成为大多数发行版的标准配置），所以如果要使用systemd要自己根据官方镜像在制作启动了systemd的CentOS7的基础镜像。

(这里编译Hadoop其实用不到systemd，不过先做个基础镜像以后弄集群总用的到吧)

1. 创建文件夹并进入

   ```
   $ mkdir centos7-systemd
   $ cd centos7-systemd
   ```

2. 创建Dockerfile

   ```
   $ touch Dockerfile
   ```

3. 编辑Dockerfile

   使用vi进入编辑器，对Dockerfile进行编辑。

   ```
   $ vi Dockerfile
   ```

   使用**i**键进入编辑模式，写入如下内容：

   ```
   # 镜像来源
   FROM centos:7

   # 镜像创建者
   MAINTAINER "you" <your@email.here>

   # 设置一个环境变量
   ENV container docker

   # 运行命令
   # 设置systemd
   RUN (cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [ $i == \
   systemd-tmpfiles-setup.service ] || rm -f $i; done); \
   rm -f /lib/systemd/system/multi-user.target.wants/*;\
   rm -f /etc/systemd/system/*.wants/*;\
   rm -f /lib/systemd/system/local-fs.target.wants/*; \
   rm -f /lib/systemd/system/sockets.target.wants/*udev*; \
   rm -f /lib/systemd/system/sockets.target.wants/*initctl*; \
   rm -f /lib/systemd/system/basic.target.wants/*;\
   rm -f /lib/systemd/system/anaconda.target.wants/*;

   # 挂载一个本地文件夹
   VOLUME [ "/sys/fs/cgroup" ]

   # 设置容器启动时的执行命令
   CMD ["/usr/sbin/init"]
   ```

   编辑完成后，按**Esc**键退出编辑模式，输入“:wq”写入文件并退出。

4. 生成基础镜像

   使用如下命令生成基础镜像。

   ```
   $ sudo docker build -t centos7-systemd .
   ```

5. 查看镜像

   ```
   $ docker images
   ```




### 二、使用Dockerfile制作CentOS7环境下的编译镜像

1. 创建文件夹并进入

   ```
   $ mkdir centos7-hadoop-complier
   $ cd centos7-hadoop-complier
   ```

2. 创建Dockerfile和编译脚本

   ```
   $ touch Dockerfile compile.sh
   ```

3. 编写Dockerfile

   使用vi进入编辑器，对Dockerfile进行编辑。

   ```
   $ vi Dockerfile
   ```

   使用**i**键进入编辑模式，写入如下内容：

   ```
   # 镜像来源(上面生成的本地镜像)
   FROM centos7-systemd

   # 镜像创建者
   MAINTAINER "you" <your@email.here>

   # 运行命令安装环境依赖
   # 使用 -y 同意全部询问
   RUN yum update -y && \
       yum groupinstall -y "Development Tools" && \
       yum install -y wget \
       			  java-1.7.0-openjdk \
       			  protobuf-devel \
       			  protobuf-compiler \
       			  maven \
       			  cmake \
       			  pkgconfig \
       			  openssl-devel \
       			  zlib-devel \
       			  gcc \
       			  automake \
       			  autoconf \
       			  make
       			  
   # 复制编辑脚本文件到镜像中
   COPY compile.sh /root/compile.sh
   # 设置脚本文件的可运行权限
   RUN chmod +x /root/compile.sh
   ```

   （[kiwenlau](https://github.com/kiwenlau)/[**compile-hadoop**](https://github.com/kiwenlau/compile-hadoop)中是用的Ubuntu作为基础环境镜像的，这里用的CentOS7，所以安装依赖包的名字不太一样。）

   编辑完成后，按**Esc**键退出编辑模式，输入“:wq”写入文件并退出。

4. 编写编译脚本

   使用vi进入编辑器，对compile.sh进行编译。

   ```
   $ vi compile.sh
   ```

   使用**i**键进入编辑模式，写入如下内容：

   ```
   #!/bin/bash

   # 设置默认编译版本(支持传参)
   version=${1:-2.7.3}

   # 进入源代码目录
   cd /hadoop-$version-src

   # 开始编译
   echo -e "\n\ncompile hadoop $version..."
   mvn package -Pdist,native -DskipTests -Dtar

   # 输出结果
   if [[ $? -eq 0]]; then
   	echo -e "\n\ncompile hadoop $version success!\n\n"
   else
   	echo -e "\n\ncompile hadoop $version fail!\n\n"
   fi
   ```

   编辑完成后，按**Esc**键退出编辑模式，输入“:wq”写入文件并退出。

5. 创建编译用的镜像

   ```
   $ sudo docker build -t centos7-hadoop-compiler .
   ```




### 三、使用Docker镜像编译Hadoop

1. 下载Hadoop源代码

   到[Apache Hadoop官网](http://hadoop.apache.org/releases.html)，找到想编译版本的Hadoop的下载地址。只用wget下载：

   ```
   $ wget http://apache.fayea.com/hadoop/common/hadoop-2.7.3/hadoop-2.7.3-src.tar.gz
   ```

   （这里是用的本文编写时的最新版本2.7.3。）

2. 解压压缩包

   ```
   $ tar -xzvf hadoop-2.7.3-src.tar.gz
   ```

3. 编译Hadoop

   先设置一个版本的临时变量，这样直接复制Docker命令，就可以运行上面生成的编译用镜像编译Hadoop了！

   ```
   $ export VERSION=2.7.3
   $ sudo docker run -v $(pwd)/hadoop-$VERSION-src:/hadoop-$VERSION-src centos7-hadoop-complier /root/compile.sh $VERSION
   ```

等待……等待……等待……应该有10多分钟才能编译好……

编译好后，进入下面位置就能找到编译好的64位版本的Hadoop了。

```
$ cd hadoop-$VERSION-src/hadoop-dist/target
$ ls
```



### 附：命令行纯净版

```
[xxx@localhost ~]$ su root
Password:
[root@localhost ~]$ mkdir centos7-systemd
[root@localhost ~]$ cd centos7-systemd
[root@localhost ~]$ touch Dockerfile
[root@localhost ~]$ vi Dockerfile

# 镜像来源
FROM centos:7

# 镜像创建者
MAINTAINER "you" <your@email.here>

# 设置一个环境变量
ENV container docker

# 运行命令
# 设置systemd
RUN (cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [ $i == \
systemd-tmpfiles-setup.service ] || rm -f $i; done); \
rm -f /lib/systemd/system/multi-user.target.wants/*;\
rm -f /etc/systemd/system/*.wants/*;\
rm -f /lib/systemd/system/local-fs.target.wants/*; \
rm -f /lib/systemd/system/sockets.target.wants/*udev*; \
rm -f /lib/systemd/system/sockets.target.wants/*initctl*; \
rm -f /lib/systemd/system/basic.target.wants/*;\
rm -f /lib/systemd/system/anaconda.target.wants/*;

# 挂载一个本地文件夹
VOLUME [ "/sys/fs/cgroup" ]

# 设置容器启动时的执行命令
CMD ["/usr/sbin/init"]
~
~
~
[root@localhost ~]$ docker build -t centos7-systemd .
[root@localhost ~]$ cd ../
[root@localhost ~]$ mkdir centos7-hadoop-complier
[root@localhost ~]$ cd centos7-hadoop-complier
[root@localhost ~]$ touch Dockerfile compile.sh
[root@localhost ~]$ vi Dockerfile

# 镜像来源(上面生成的本地镜像)
FROM centos7-systemd

# 镜像创建者
MAINTAINER "you" <your@email.here>

# 运行命令安装环境依赖
# 使用 -y 同意全部询问
RUN yum update -y && \
    yum groupinstall -y "Development Tools" && \
    yum install -y wget \
    			  java-1.7.0-openjdk \
    			  protobuf-devel \
    			  protobuf-compiler \
    			  maven \
    			  cmake \
    			  pkgconfig \
    			  openssl-devel \
    			  zlib-devel \
    			  gcc \
    			  automake \
    			  autoconf \
    			  make
    			  
# 复制编辑脚本文件到镜像中
COPY compile.sh /root/compile.sh
# 设置脚本文件的可运行权限
RUN chmod +x /root/compile.sh
~
~
~
[root@localhost ~]$ vi compile.sh

#!/bin/bash

# 设置默认编译版本(支持传参)
version=${1:-2.7.3}

# 进入源代码目录
cd /hadoop-$version-src

# 开始编译
echo -e "\n\ncompile hadoop $version..."
mvn package -Pdist,native -DskipTests -Dtar

# 输出结果
if [[ $? -eq 0]]; then
	echo -e "\n\ncompile hadoop $version success!\n\n"
else
	echo -e "\n\ncompile hadoop $version fail!\n\n"
fi
~
~
~
[root@localhost ~]$ docker build -t centos7-hadoop-compiler .
[root@localhost ~]$ cd ../
[root@localhost ~]$ wget http://apache.fayea.com/hadoop/common/hadoop-2.7.3/hadoop-2.7.3-src.tar.gz
[root@localhost ~]$ tar -xzvf hadoop-2.7.3-src.tar.gz
[root@localhost ~]$ export VERSION=2.7.3
[root@localhost ~]$ docker run -v $(pwd)/hadoop-$VERSION-src:/hadoop-$VERSION-src centos7-hadoop-complier /root/compile.sh $VERSION
[root@localhost ~]$ cd hadoop-$VERSION-src/hadoop-dist/target
[root@localhost ~]$ ls
```