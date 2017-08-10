# 杂学札记之大数据系列：使用Docker搭建Hadoop集群

（搭建集群部分借鉴了[kiwenlau](https://github.com/kiwenlau)/[**hadoop-cluster-docker**](https://github.com/kiwenlau/hadoop-cluster-docker)中的内容，不过那里的基础环境是Ubuntu，本人这里是用的CentOS7，因此也糟了不少坑！）

### 目录索引

[一、编辑Hadoop运行环境中的配置文件](#一编辑hadoop运行环境中的配置文件)

[二、使用Dockerfile制作Hadoop的镜像](#二使用dockerfile制作hadoop的镜像)

[三、激情的排坑之旅](#三激情的排坑之旅)

[四、最终的文件内容](四最终的文件内容)

​	— [config/core-site.xml](#configcore-sitexml)

​	— [config/hadoop-env.sh](#confighadoop-envsh)

​	— [config/hdfs-site.xml](#confighdfs-sitexml)

​	— [config/mapred-site.xml](#configmapred-sitexml)

​	— [config/run-wordcount.sh](#configrun-wordcountsh)

​	— [config/slaves](#configslaves)

​	— [config/ssh_config](#configssh_config)

​	— [config/start-hadoop.sh](#configstart-hadoopsh)

​	— [config/yarn-site.xml](#configyarn-sitexml)

​	— [Dockerfile](#dockerfile)

​	— [start-container.sh](#start-containersh)

​	— [stop-container.sh](#stopcontainersh)

​	— [remove-container.sh](#remove-containersh)

​	— [resize-cluster.sh](#resize-clustersh)

[附：命令行纯净版](#附命令行纯净版)



### 一、编辑Hadoop运行环境中的配置文件

1. 创建文件夹和文件

   先创建个文件夹来放相关的文件，并创建配置文件的文件夹，新建几个文件。

   ```
   $ mkdir -p hadoop-docker/config
   $ cd hadoop-docker
   $ touch Dockerfile start-container.sh config/ssh_config config/start-hadoop.sh config/run-wordcount.sh
   ```

2. 复制Hadoop的64位编译文件

   将编译好的64位版本的Hadoop包复制到当前目录中。（编译64位Hadoop，看[这里](3_hadoop_compile.md)）

3. 复制Hadoop中的配置文件

   解压编译好的64位版本的Hadoop包，从里面复制点配置项出来修改（不然命令行下全手写还不累死啦！）。

   ```
   $ export version=2.7.3
   $ tar -xzvf hadoop-$version.tar.gz
   $ copy hadoop-$version/etc/hadoop/core-site.xml config/core-site.xml
   $ copy hadoop-$version/etc/hadoop/hadoop-env.sh config/hadoop-env.sh
   $ copy hadoop-$version/etc/hadoop/hdfs-site.xml config/hdfs-site.xml
   $ copy hadoop-$version/etc/hadoop/mapred-site.xml.template config/mapred-site.xml
   $ copy hadoop-$version/etc/hadoop/yarn-site.xml config/yarn-site.xml
   ```

4. 编辑配置文件：ssh_config

   Hadoop节点之间通讯使用的是ssh，这里设置ssh_config的配置文件，增加无密码登录设置。使用vi编辑config文件夹下的ssh_config，加入以下的内容：

   ```
   Host localhost
     StrictHostKeyChecking no

   Host 0.0.0.0
     StrictHostKeyChecking no
     
   Host hadoop-*
      StrictHostKeyChecking no
      UserKnownHostsFile=/dev/null
   ```

5. 编辑配置文件：core-site.xml

   使用vi编辑config文件夹下的core-site.xml，在configuration中间加入以下内容：

   ```
   <!--指定namenode的地址-->
   <property>
   	<name>fs.defaultFS</name>
       <value>hdfs://hadoop-master:9000/</value>
   </property>
   ```

6. 编辑配置文件：hdfs-site.xml

   使用vi编辑config文件夹下的hdfs-site.xml，在configuration中间加入以下内容：

   ```
   <!--指定hdfs的Name节点的保存目录-->
   <property>
   	<name>dfs.namenode.name.dir</name>
       <value>file:///root/hdfs/namenode</value>
       <description>NameNode directory for namespace and transaction logs storage.</description></property>
   <!--指定hdfs的Data节点的保存目录-->
   <property>
   	<name>dfs.datanode.data.dir</name>
       <value>file:///root/hdfs/datanode</value>
       <description>DataNode directory</description>
   </property>
   <!--指定hdfs保存数据的副本数量-->
   <property>
   	<name>dfs.replication</name>
       <value>2</value>
   </property>
   ```

7. 编辑配置文件：mapred-site.xml

   使用vi编辑config文件夹下的mapred-site.xml，在configuration中间加入以下内容：

   ```
   <!--告诉hadoop以后MR运行在YARN上-->
   <property>
   	<name>mapreduce.framework.name</name>
       <value>yarn</value>
   </property>
   ```

8. 编辑配置文件：yarn-site.xml

   使用vi编辑config文件夹下的yarn-site.xml，在configuration中间加入以下内容：

   ```
   <!--nodeManager获取数据的方式是shuffle-->
   <property>
   	<name>yarn.nodemanager.aux-services</name>
       <value>mapreduce_shuffle</value>
   </property>
   <property>
   	<name>yarn.nodemanager.aux-services.mapreduce_shuffle.class</name>
       <value>org.apache.hadoop.mapred.ShuffleHandler</value>
   </property>
   <!--指定Yarn的老大(ResourceManager)的地址-->
   <property>
   	<name>yarn.resourcemanager.hostname</name>
       <value>hadoop-master</value>
   </property>
   ```

9. 编辑环境配置脚本：hadoop-env.sh

   使用vi编辑config文件夹下的hadoop-env.sh，找到JAVA_HOME设置，修改为如下（其他内容不变）：

   ```
   export JAVA_HOME=/usr/lib/jvm/java-1.7.0-openjdk
   ```

   (呵呵，其实这里有问题，后面排坑的时候再说！)

10. 编辑从节点记录：slaves

 ```
 hadoop-slave1
 hadoop-slave2
 ```

 这个后面有个脚本可以根据从节点数量自动生成。

11. 编辑启动Hadoop的脚本：start-hadoop.sh

   使用vi编辑config文件夹下的start-hadoop.sh，用于在Master上执行启动hadoop的命令：

   ```
   #!/bin/bash

   echo -e "\n"

   $HADOOP_HOME/sbin/start-dfs.sh

   echo -e "\n"

   $HADOOP_HOME/sbin/start-yarn.sh

   echo -e "\n"
   ```

12. 编辑运行入门程序WordCount的脚本：run-wordcount.sh

    使用vi编辑config文件夹下的run-wordcount.sh，用来运行Hadoop的入门程序WordCount：

    ```
    #!/bin/bash

    # test the hadoop cluster by running wordcount

    # create input files 
    mkdir input
    echo "Hello Docker" >input/file2.txt
    echo "Hello Hadoop" >input/file1.txt

    # create input directory on HDFS
    hadoop fs -mkdir -p input

    # put input files to HDFS
    hdfs dfs -put ./input/* input

    # run wordcount 
    hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/sources/hadoop-mapreduce-examples-2.7.3-sources.jar org.apache.hadoop.examples.WordCount input output

    # print the input files
    echo -e "\ninput file1.txt:"
    hdfs dfs -cat input/file1.txt

    echo -e "\ninput file2.txt:"
    hdfs dfs -cat input/file2.txt

    # print the output of wordcount
    echo -e "\nwordcount output:"
    hdfs dfs -cat output/part-r-00000
    ```




### 二、使用Dockerfile制作Hadoop的镜像

Hadoop镜像中到相关配置文件和脚本都写好了，开始编辑Dockerfile并制作Hadoop的镜像。

使用vi打开Dockerfile，开始编辑其中的内容。

1. 添加基础镜像和基本信息

   这里用的基础镜像是centos7环境并开通了systemd启动管理程序，具体生成可参见之前的文章（[使用Docker编译64位的Hadoop](3_hadoop_compile.md)）。

   ```
   # 镜像来源
   FROM centos7-systemd

   # 镜像创建者(写入自己的信息)
   MAINTAINER "you" <your@email.here>

   # 指定目录
   WORKDIR /root
   ```

2. 安装运行环境需要的软件

   安装Java jdk和openssh。

   ```
   RUN yum update -y && \
       yum install -y java-1.7.0-openjdk \
       			  openssh-server
   ```

   （其实这里还不够，还是后面排坑的时候再说。）

3. 复制Hadoop并安装

   这里设置了个环境变量方便在更换版本的时候修改。

   ```
   # 复制Hadoop
   ENV HADOOP_VERSION=2.7.3
   COPY hadoop-$HADOOP_VERSION.tar.gz /root/hadoop-$HADOOP_VERSION.tar.gz
   # 安装
   RUN tar -xzvf hadoop-$HADOOP_VERSION.tar.gz && \
       mv hadoop-$HADOOP_VERSION /usr/local/hadoop && \
       rm hadoop-$HADOOP_VERSION.tar.gz
   ```

4. 设置环境变量

   设置JAVA_HOME，HADOOP_HOME。

   ```
   ENV JAVA_HOME=/usr/lib/jvm/java-1.7.0-openjdk
   ENV HADOOP_HOME=/usr/local/hadoop
   ENV PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
   ```

   （这里也漏了一个，另外JAVA_HOME也有问题，后面排坑的时候……）

5. 设置SSH免密码登录

   ```
   RUN ssh-keygen -t rsa -f ~/.ssh/id_rsa -P '' && \
       cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
   ```

6. 复制配置文件到Hadoop下

   ```
   COPY config/* /tmp/
   RUN mv /tmp/ssh_config ~/.ssh/config && \
       mv /tmp/hadoop-env.sh /usr/local/hadoop/etc/hadoop/hadoop-env.sh && \
       mv /tmp/hdfs-site.xml $HADOOP_HOME/etc/hadoop/hdfs-site.xml && \ 
       mv /tmp/core-site.xml $HADOOP_HOME/etc/hadoop/core-site.xml && \
       mv /tmp/mapred-site.xml $HADOOP_HOME/etc/hadoop/mapred-site.xml && \
       mv /tmp/yarn-site.xml $HADOOP_HOME/etc/hadoop/yarn-site.xml && \
       mv /tmp/slaves $HADOOP_HOME/etc/hadoop/slaves && \
       mv /tmp/start-hadoop.sh ~/start-hadoop.sh && \
       mv /tmp/run-wordcount.sh ~/run-wordcount.sh
   RUN chmod +x ~/start-hadoop.sh && \
       chmod +x ~/run-wordcount.sh && \
       chmod +x $HADOOP_HOME/sbin/start-dfs.sh && \
       chmod +x $HADOOP_HOME/sbin/start-yarn.sh
   ```

   （这里也有个权限的问题，后面排坑……）

7. 设置节点

   创建目录，并格式化HDFS。

   ```
   RUN mkdir -p ~/hdfs/namenode && \ 
       mkdir -p ~/hdfs/datanode && \
       mkdir $HADOOP_HOME/logs
   RUN $HADOOP_HOME/bin/hdfs namenode -format
   ```

8. 设置容器打开后运行ssh

   CentOS7基础镜像中开通了systemd作为启动守护，所以这里使用systemctl开启ssh服务。

   ```
   CMD [ "sh", "-c", "systemctl start sshd; bash"]
   ```

这样Dockerfile就编写完成，再增加一个脚本用来启动指定节点数量的Hadoop集群，内容如下：

```
#!/bin/bash

# 默认节点数3个(即一个master,两个slave)
N=${1:-3}


# 开启Hadoop-Master容器
sudo docker rm -f hadoop-master &> /dev/null
echo "start hadoop-master container..."
sudo docker run -itd \
                --net=hadoop \
                -p 50070:50070 \
                -p 8088:8088 \
                --name hadoop-master \
                --hostname hadoop-master \
                hadoop-docker &> /dev/null

# 开启Hadoop-Slave容器
i=1
while [ $i -lt $N ]
do
	sudo docker rm -f hadoop-slave$i &> /dev/null
	echo "start hadoop-slave$i container..."
	sudo docker run -itd \
	                --net=hadoop \
	                --name hadoop-slave$i \
	                --hostname hadoop-slave$i \
	                hadoop-docker &> /dev/null
	i=$(( $i + 1 ))
done 

# 进入Hadoop-Master容器的命令行
sudo docker exec -it hadoop-master bash
```

（这里其实也有问题，后面……）

好了，一切准备就绪了开始运行！…………这是咋的了呢？……



### 三、激情的排坑之旅

果然没有一切顺利的，从弄镜像开始就出问题了，一一排查解决吧！想直接看最后的正确内容可以直接看【[四、最终的文件内容](#四最终的文件内容)】或者【[附：命令行纯净版](#命令行纯净版)】。

1. 构建镜像：hdfs，命令没找到

   ```
   $ docker build -t hadoop-docker .
   ```

   报错：hdfs命令没找到，仔细再看报错其实是:

   libexec/hdfs-config.sh:No such file or directory

   查找了一下，原来是缺少一个环境变量，在Dockerfile环境变量设置那里增加一行：

   ```
   ENV HADOOP_LIBEXEC_DIR=$HADOOP_HOME/libexec
   ```

2. 启动Hadoop：JAVA_HOME，没找到这个目录

   再次构建，构建成功了。使用脚本启动全部集群容器（默认的3个Node）：

   ```
   $ ./start-container.sh
   ```

   进入Hadoop-Master的命令行后，使用脚本启动Hadoop：

   ```
   $ ./start-hadoop.sh
   ```

   报错：/usr/lib/jvm/java-1.7.0-openjdk:No such file or directory

   这个目录其实是根据网上yum安装的路径自己猜的，应该有问题，即然在容器里直接去看看好了：

   ```
   $ ls /usr/lib/jvm
   java-1.7.0-openjdk-1.7.0.131-2.6.9.0.e17_3.x86_64 jre jre-1.7.0 jre-1.7.0-openjdk jre-1.7.0-openjdk-1.7.0.131-2.6.9.0.e17_3.x86_64 jre-openjdk
   ```

   晕……原来是有版本的小编号的，这以后是不是每次安装版本不同了就不一样了啊，如果写这个进去岂不是每次都要生成好看看小编号再重新构建？！想了想反正这里只是用运行环境，直接改成jre试试，一次尝试修改JAVA_HOME为jre-1.7.0-openjdk，包括Dockerfile以及config/hadoop-env.sh两个文件。

3. 启动Hadoop：ssh，连接失败

   重新构建，再次启动Hadoop，还是报错：ssh无法连接。试了一下ssh的服务根本没启动，直接运行：

   ```
   $ systemctl start sshd
   ```

   报错：Failed to get D-Bus connection

   网上反映这个错误的不少，据说是CentOS7在Docker下著名的Bug，最后找到解决方案，在我们启动容器的脚本start-container.sh中修改启动主节点和从节点的docker run命令，增加内容：

   ```
   sudo docker run -itd \
   			   # 这行新加
          		    --privileged -e "container=docker" -v /sys/fs/cgroup:/sys/fs/cgroup \
                   --net=hadoop \
                   -p 50070:50070 \
                   -p 8088:8088 \
                   --name hadoop-master \
                   --hostname hadoop-master \
                   hadoop-docker &> /dev/null \
                   # 这行新加
   			   /usr/sbin/init
   ```

   这样保证在容器开启时运行/usr/sbin/init，以此开启D-Bus服务。

4. 启动Hadoop：ssh，不好的权限设置

   再次启动容器，启动Hadoop，还是报错：

   ```
   Bad owner or permissions on ~/.ssh/config
   ```

   再次找到解决方案，在Dockerfile镜像生成时修改这个config的权限为600，即在Dockerfile中增加一行：

   ```
   RUN chmod 600 ~/.ssh/config
   ```

5. 启动Hadoop：ssh，无法连接

   再次构建，再次启动容器，再次启动Hadoop，还来ssh无法连接！

   原来我只安装了server没装client吗，怎么连接！修改Dockerfile的软件安装为：

   ```
   RUN yum update -y && \
       yum install -y java-1.7.0-openjdk \
    		   openssh-server \
   		   openssh-clients 
   ```

6. 启动Hadoop：hdfs，命令没找到

   还没完啊！这次又是万恶的hdfs命令没找到，开始一直纠结在提示里的hdfs-config.sh这个文件没有的问题，排查了很久，后来再仔细看了看发现了一个错误：

   which:command not found

   这……个which是个命令？查一下原来就是这个which没有安装的原因！！再次修改Dockerfile中的软件安装为：

   ```
   RUN yum update -y && \
       yum install -y java-1.7.0-openjdk \
    		   openssh-server \
   		   openssh-clients \
   		   which
   ```

再次构建……再次开启容器……再次开启Hadoop，正常了……

运行WordCount：

```
$ ./run-wordcount.sh
```

也正确了！



### 四、最终的文件内容

这里列出全部文件的内容，觉得麻烦也可以直接访问项目[hadoop-centos-docker](https://github.com/ProteanBear/hadoop-centos-docker)。

##### config/core-site.xml

```
<?xml version="1.0"?>
<configuration>
	<!--指定namenode的地址-->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop-master:9000/</value>
    </property>
</configuration>
```

##### config/hadoop-env.sh

```
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Set Hadoop-specific environment variables here.

# The only required environment variable is JAVA_HOME.  All others are
# optional.  When running a distributed configuration it is best to
# set JAVA_HOME in this file, so that it is correctly defined on
# remote nodes.

# The java implementation to use.
export JAVA_HOME=/usr/lib/jvm/jre-1.7.0-openjdk

# The jsvc implementation to use. Jsvc is required to run secure datanodes
# that bind to privileged ports to provide authentication of data transfer
# protocol.  Jsvc is not required if SASL is configured for authentication of
# data transfer protocol using non-privileged ports.
#export JSVC_HOME=${JSVC_HOME}

export HADOOP_CONF_DIR=${HADOOP_CONF_DIR:-"/etc/hadoop"}

# Extra Java CLASSPATH elements.  Automatically insert capacity-scheduler.
for f in $HADOOP_HOME/contrib/capacity-scheduler/*.jar; do
  if [ "$HADOOP_CLASSPATH" ]; then
    export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:$f
  else
    export HADOOP_CLASSPATH=$f
  fi
done

# The maximum amount of heap to use, in MB. Default is 1000.
#export HADOOP_HEAPSIZE=
#export HADOOP_NAMENODE_INIT_HEAPSIZE=""

# Extra Java runtime options.  Empty by default.
export HADOOP_OPTS="$HADOOP_OPTS -Djava.net.preferIPv4Stack=true"

# Command specific options appended to HADOOP_OPTS when specified
export HADOOP_NAMENODE_OPTS="-Dhadoop.security.logger=${HADOOP_SECURITY_LOGGER:-INFO,RFAS} -Dhdfs.audit.logger=${HDFS_AUDIT_LOGGER:-INFO,NullAppender} $HADOOP_NAMENODE_OPTS"
export HADOOP_DATANODE_OPTS="-Dhadoop.security.logger=ERROR,RFAS $HADOOP_DATANODE_OPTS"

export HADOOP_SECONDARYNAMENODE_OPTS="-Dhadoop.security.logger=${HADOOP_SECURITY_LOGGER:-INFO,RFAS} -Dhdfs.audit.logger=${HDFS_AUDIT_LOGGER:-INFO,NullAppender} $HADOOP_SECONDARYNAMENODE_OPTS"

export HADOOP_NFS3_OPTS="$HADOOP_NFS3_OPTS"
export HADOOP_PORTMAP_OPTS="-Xmx512m $HADOOP_PORTMAP_OPTS"

# The following applies to multiple commands (fs, dfs, fsck, distcp etc)
export HADOOP_CLIENT_OPTS="-Xmx512m $HADOOP_CLIENT_OPTS"
#HADOOP_JAVA_PLATFORM_OPTS="-XX:-UsePerfData $HADOOP_JAVA_PLATFORM_OPTS"

# On secure datanodes, user to run the datanode as after dropping privileges.
# This **MUST** be uncommented to enable secure HDFS if using privileged ports
# to provide authentication of data transfer protocol.  This **MUST NOT** be
# defined if SASL is configured for authentication of data transfer protocol
# using non-privileged ports.
export HADOOP_SECURE_DN_USER=${HADOOP_SECURE_DN_USER}

# Where log files are stored.  $HADOOP_HOME/logs by default.
#export HADOOP_LOG_DIR=${HADOOP_LOG_DIR}/$USER

# Where log files are stored in the secure data environment.
export HADOOP_SECURE_DN_LOG_DIR=${HADOOP_LOG_DIR}/${HADOOP_HDFS_USER}

###
# HDFS Mover specific parameters
###
# Specify the JVM options to be used when starting the HDFS Mover.
# These options will be appended to the options specified as HADOOP_OPTS
# and therefore may override any similar flags set in HADOOP_OPTS
#
# export HADOOP_MOVER_OPTS=""

###
# Advanced Users Only!
###

# The directory where pid files are stored. /tmp by default.
# NOTE: this should be set to a directory that can only be written to by 
#       the user that will run the hadoop daemons.  Otherwise there is the
#       potential for a symlink attack.
export HADOOP_PID_DIR=${HADOOP_PID_DIR}
export HADOOP_SECURE_DN_PID_DIR=${HADOOP_PID_DIR}

# A string representing this instance of hadoop. $USER by default.
export HADOOP_IDENT_STRING=$USER
```

##### config/hdfs-site.xml

```
<?xml version="1.0"?>
<configuration>
	<!--指定hdfs的Name节点的保存目录-->
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:///root/hdfs/namenode</value>
        <description>NameNode directory for namespace and transaction logs storage.</description>
    </property>
    <!--指定hdfs的Data节点的保存目录-->
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:///root/hdfs/datanode</value>
        <description>DataNode directory</description>
    </property>
    <!--指定hdfs保存数据的副本数量-->
    <property>
        <name>dfs.replication</name>
        <value>2</value>
    </property>
</configuration>
```

##### config/mapred-site.xml

```
<?xml version="1.0"?>
<configuration>
	<!--告诉hadoop以后MR运行在YARN上-->
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

##### config/run-wordcount.sh

```
#!/bin/bash

# test the hadoop cluster by running wordcount

# create input files 
mkdir input
echo "Hello Docker" >input/file2.txt
echo "Hello Hadoop" >input/file1.txt

# create input directory on HDFS
hadoop fs -mkdir -p input

# put input files to HDFS
hdfs dfs -put ./input/* input

# run wordcount 
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/sources/hadoop-mapreduce-examples-2.7.3-sources.jar org.apache.hadoop.examples.WordCount input output

# print the input files
echo -e "\ninput file1.txt:"
hdfs dfs -cat input/file1.txt

echo -e "\ninput file2.txt:"
hdfs dfs -cat input/file2.txt

# print the output of wordcount
echo -e "\nwordcount output:"
hdfs dfs -cat output/part-r-00000
```

##### config/slaves

```
hadoop-slave1
hadoop-slave2
```

##### config/ssh_config

```
Host localhost
  StrictHostKeyChecking no

Host 0.0.0.0
  StrictHostKeyChecking no
  
Host hadoop-*
   StrictHostKeyChecking no
   UserKnownHostsFile=/dev/null
```

##### config/start-hadoop.sh

```
#!/bin/bash

echo -e "\n"

$HADOOP_HOME/sbin/start-dfs.sh

echo -e "\n"

$HADOOP_HOME/sbin/start-yarn.sh

echo -e "\n"
```

##### config/yarn-site.xml

```
<?xml version="1.0"?>
<configuration>
	<!--nodeManager获取数据的方式是shuffle-->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services.mapreduce_shuffle.class</name>
        <value>org.apache.hadoop.mapred.ShuffleHandler</value>
    </property>
    <!--指定Yarn的老大(ResourceManager)的地址-->
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hadoop-master</value>
    </property>
</configuration>
```

##### Dockerfile

```
# 镜像来源
FROM centos7-systemd

# 镜像创建者(写入自己的信息)
MAINTAINER "you" <your@email.here>

# 指定目录
WORKDIR /root

# 安装软件
RUN yum update -y && \
    yum install -y java-1.7.0-openjdk \
 		   openssh-server \
		   openssh-clients \
		   which

# 复制Hadoop
ENV HADOOP_VERSION=2.7.3
COPY hadoop-$HADOOP_VERSION.tar.gz /root/hadoop-$HADOOP_VERSION.tar.gz
# 安装
RUN tar -xzvf hadoop-$HADOOP_VERSION.tar.gz && \
    mv hadoop-$HADOOP_VERSION /usr/local/hadoop && \
    rm hadoop-$HADOOP_VERSION.tar.gz

# 设置环境变量
ENV JAVA_HOME=/usr/lib/jvm/jre-1.7.0-openjdk
ENV HADOOP_HOME=/usr/local/hadoop
ENV PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
ENV HADOOP_LIBEXEC_DIR=$HADOOP_HOME/libexec 

# ssh无密码登录
RUN ssh-keygen -t rsa -f ~/.ssh/id_rsa -P '' && \
    cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

# 复制配置文件及配置权限
COPY config/* /tmp/
RUN mv /tmp/ssh_config ~/.ssh/config && \
    mv /tmp/hadoop-env.sh /usr/local/hadoop/etc/hadoop/hadoop-env.sh && \
    mv /tmp/hdfs-site.xml $HADOOP_HOME/etc/hadoop/hdfs-site.xml && \ 
    mv /tmp/core-site.xml $HADOOP_HOME/etc/hadoop/core-site.xml && \
    mv /tmp/mapred-site.xml $HADOOP_HOME/etc/hadoop/mapred-site.xml && \
    mv /tmp/yarn-site.xml $HADOOP_HOME/etc/hadoop/yarn-site.xml && \
    mv /tmp/slaves $HADOOP_HOME/etc/hadoop/slaves && \
    mv /tmp/start-hadoop.sh ~/start-hadoop.sh && \
    mv /tmp/run-wordcount.sh ~/run-wordcount.sh
RUN chmod 600 ~/.ssh/config && \
    chmod +x ~/start-hadoop.sh && \
    chmod +x ~/run-wordcount.sh && \
    chmod +x $HADOOP_HOME/sbin/start-dfs.sh && \
    chmod +x $HADOOP_HOME/sbin/start-yarn.sh 

# 格式化namenode
RUN mkdir -p ~/hdfs/namenode && \ 
    mkdir -p ~/hdfs/datanode && \
    mkdir $HADOOP_HOME/logs
RUN $HADOOP_HOME/bin/hdfs namenode -format

# 启动容器后开启ssh服务
CMD [ "sh", "-c", "systemctl start sshd; bash"]
```

##### start-container.sh

```
#!/bin/bash

# 默认节点数3个(即一个master,两个slave)
N=${1:-3}


# 开启Hadoop-Master容器
sudo docker rm -f hadoop-master &> /dev/null
echo "start hadoop-master container..."
sudo docker run -itd \
       		    --privileged -e "container=docker" -v /sys/fs/cgroup:/sys/fs/cgroup \
                --net=hadoop \
                -p 50070:50070 \
                -p 8088:8088 \
                --name hadoop-master \
                --hostname hadoop-master \
                hadoop-docker &> /dev/null \
			   /usr/sbin/init

# 开启Hadoop-Slave容器
i=1
while [ $i -lt $N ]
do
	sudo docker rm -f hadoop-slave$i &> /dev/null
	echo "start hadoop-slave$i container..."
	sudo docker run -itd \
				   --privileged -e "container=docker" -v /sys/fs/cgroup:/sys/fs/cgroup \
	                --net=hadoop \
	                --name hadoop-slave$i \
	                --hostname hadoop-slave$i \
	                hadoop-docker &> /dev/null \
				  /usr/sbin/init
	i=$(( $i + 1 ))
done 

# 进入Hadoop-Master容器的命令行
sudo docker exec -it hadoop-master bash
```

##### stop-container.sh

增加了一个关闭全部主从节点容器的脚本。

```
#!/bin/bash

# 默认节点数3个(即一个master,两个slave)
N=${1:-3}

# 关闭Hadoop-Master容器
sudo docker container stop hadoop-master
echo "stop hadoop-master container..."

# 开启Hadoop-Slave容器
i=1
while [ $i -lt $N ]
do
	echo "stop hadoop-slave$i container..."
	sudo docker container stop hadoop-slave$i
	i=$(( $i+1 ))
done
```

##### remove-container.sh

Docker失败过程中会生成一些none镜像，而且因为有依赖所以清除起来比较麻烦，这里是在网上搜集的清除方法写成单独的脚本，堪称强迫症患者的福音！

```
#!/bin/bash

# 默认为none镜像
name=${1:-none}

# 删除容器镜像
docker ps -a | grep "Exited" | awk '{print $1 }' |xargs docker stop
docker ps -a | grep "Exited" | awk '{print $1 }' |xargs docker rm
docker images| grep $name | awk '{print $3 }' |xargs docker rmi
```

##### resize-cluster.sh

重新设置从节点数量并重构镜像的脚本。

```
#!/bin/bash

# N is the node number of hadoop cluster
N=$1

if [ $# = 0 ]
then
	echo "Please specify the node number of hadoop cluster!"
	exit 1
fi

# change slaves file
i=1
rm config/slaves
while [ $i -lt $N ]
do
	echo "hadoop-slave$i" >> config/slaves
	((i++))
done 

echo -e "\nrebuild docker hadoop image!\n"

# rebuild hadoop image
sudo docker build -t hadoop-docker .

# clear none image
./remove-container.sh
```



### 附：命令行纯净版

```
[xxx@localhost ~]$ mkdir -p hadoop-docker/config
[xxx@localhost ~]$ cd hadoop-docker
[xxx@localhost hadoop-docker]$ touch Dockerfile start-container.sh config/ssh_config config/start-hadoop.sh config/run-wordcount.sh
[xxx@localhost hadoop-docker]$ export version=2.7.3
[xxx@localhost hadoop-docker]$ cp ../hadoop-src/hadoop-$version-src/hadoop-dist/target/hadoop-$version.tar.gz hadoop-$version.tar.gz
[xxx@localhost hadoop-docker]$ tar -xzvf hadoop-$version.tar.gz
[xxx@localhost hadoop-docker]$ copy hadoop-$version/etc/hadoop/core-site.xml config/core-site.xml
[xxx@localhost hadoop-docker]$ copy hadoop-$version/etc/hadoop/hadoop-env.sh config/hadoop-env.sh
[xxx@localhost hadoop-docker]$ copy hadoop-$version/etc/hadoop/hdfs-site.xml config/hdfs-site.xml
[xxx@localhost hadoop-docker]$ copy hadoop-$version/etc/hadoop/mapred-site.xml.template config/mapred-site.xml
[xxx@localhost hadoop-docker]$ copy hadoop-$version/etc/hadoop/yarn-site.xml config/yarn-site.xml
[xxx@localhost hadoop-docker]$ vi config/core-site.xml

<?xml version="1.0"?>
<configuration>
	<!--指定namenode的地址-->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop-master:9000/</value>
    </property>
</configuration>
~
~
~
[xxx@localhost hadoop-docker]$ vi config/hadoop-env.sh

# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Set Hadoop-specific environment variables here.

# The only required environment variable is JAVA_HOME.  All others are
# optional.  When running a distributed configuration it is best to
# set JAVA_HOME in this file, so that it is correctly defined on
# remote nodes.

# The java implementation to use.
export JAVA_HOME=/usr/lib/jvm/jre-1.7.0-openjdk

# The jsvc implementation to use. Jsvc is required to run secure datanodes
# that bind to privileged ports to provide authentication of data transfer
# protocol.  Jsvc is not required if SASL is configured for authentication of
# data transfer protocol using non-privileged ports.
#export JSVC_HOME=${JSVC_HOME}

export HADOOP_CONF_DIR=${HADOOP_CONF_DIR:-"/etc/hadoop"}

# Extra Java CLASSPATH elements.  Automatically insert capacity-scheduler.
for f in $HADOOP_HOME/contrib/capacity-scheduler/*.jar; do
  if [ "$HADOOP_CLASSPATH" ]; then
    export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:$f
  else
    export HADOOP_CLASSPATH=$f
  fi
done

# The maximum amount of heap to use, in MB. Default is 1000.
#export HADOOP_HEAPSIZE=
#export HADOOP_NAMENODE_INIT_HEAPSIZE=""

# Extra Java runtime options.  Empty by default.
export HADOOP_OPTS="$HADOOP_OPTS -Djava.net.preferIPv4Stack=true"

# Command specific options appended to HADOOP_OPTS when specified
export HADOOP_NAMENODE_OPTS="-Dhadoop.security.logger=${HADOOP_SECURITY_LOGGER:-INFO,RFAS} -Dhdfs.audit.logger=${HDFS_AUDIT_LOGGER:-INFO,NullAppender} $HADOOP_NAMENODE_OPTS"
export HADOOP_DATANODE_OPTS="-Dhadoop.security.logger=ERROR,RFAS $HADOOP_DATANODE_OPTS"

export HADOOP_SECONDARYNAMENODE_OPTS="-Dhadoop.security.logger=${HADOOP_SECURITY_LOGGER:-INFO,RFAS} -Dhdfs.audit.logger=${HDFS_AUDIT_LOGGER:-INFO,NullAppender} $HADOOP_SECONDARYNAMENODE_OPTS"

export HADOOP_NFS3_OPTS="$HADOOP_NFS3_OPTS"
export HADOOP_PORTMAP_OPTS="-Xmx512m $HADOOP_PORTMAP_OPTS"

# The following applies to multiple commands (fs, dfs, fsck, distcp etc)
export HADOOP_CLIENT_OPTS="-Xmx512m $HADOOP_CLIENT_OPTS"
#HADOOP_JAVA_PLATFORM_OPTS="-XX:-UsePerfData $HADOOP_JAVA_PLATFORM_OPTS"

# On secure datanodes, user to run the datanode as after dropping privileges.
# This **MUST** be uncommented to enable secure HDFS if using privileged ports
# to provide authentication of data transfer protocol.  This **MUST NOT** be
# defined if SASL is configured for authentication of data transfer protocol
# using non-privileged ports.
export HADOOP_SECURE_DN_USER=${HADOOP_SECURE_DN_USER}

# Where log files are stored.  $HADOOP_HOME/logs by default.
#export HADOOP_LOG_DIR=${HADOOP_LOG_DIR}/$USER

# Where log files are stored in the secure data environment.
export HADOOP_SECURE_DN_LOG_DIR=${HADOOP_LOG_DIR}/${HADOOP_HDFS_USER}

###
# HDFS Mover specific parameters
###
# Specify the JVM options to be used when starting the HDFS Mover.
# These options will be appended to the options specified as HADOOP_OPTS
# and therefore may override any similar flags set in HADOOP_OPTS
#
# export HADOOP_MOVER_OPTS=""

###
# Advanced Users Only!
###

# The directory where pid files are stored. /tmp by default.
# NOTE: this should be set to a directory that can only be written to by 
#       the user that will run the hadoop daemons.  Otherwise there is the
#       potential for a symlink attack.
export HADOOP_PID_DIR=${HADOOP_PID_DIR}
export HADOOP_SECURE_DN_PID_DIR=${HADOOP_PID_DIR}

# A string representing this instance of hadoop. $USER by default.
export HADOOP_IDENT_STRING=$USER
~
~
~
[xxx@localhost hadoop-docker]$ vi config/hdfs-site.xml

<?xml version="1.0"?>
<configuration>
	<!--指定hdfs的Name节点的保存目录-->
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:///root/hdfs/namenode</value>
        <description>NameNode directory for namespace and transaction logs storage.</description>
    </property>
    <!--指定hdfs的Data节点的保存目录-->
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:///root/hdfs/datanode</value>
        <description>DataNode directory</description>
    </property>
    <!--指定hdfs保存数据的副本数量-->
    <property>
        <name>dfs.replication</name>
        <value>2</value>
    </property>
</configuration>
~
~
~
[xxx@localhost hadoop-docker]$ vi config/mapred-site.xml

<?xml version="1.0"?>
<configuration>
	<!--告诉hadoop以后MR运行在YARN上-->
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
~
~
~
[xxx@localhost hadoop-docker]$ vi config/run-wordcount.sh

#!/bin/bash

# test the hadoop cluster by running wordcount

# create input files 
mkdir input
echo "Hello Docker" >input/file2.txt
echo "Hello Hadoop" >input/file1.txt

# create input directory on HDFS
hadoop fs -mkdir -p input

# put input files to HDFS
hdfs dfs -put ./input/* input

# run wordcount 
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/sources/hadoop-mapreduce-examples-2.7.3-sources.jar org.apache.hadoop.examples.WordCount input output

# print the input files
echo -e "\ninput file1.txt:"
hdfs dfs -cat input/file1.txt

echo -e "\ninput file2.txt:"
hdfs dfs -cat input/file2.txt

# print the output of wordcount
echo -e "\nwordcount output:"
hdfs dfs -cat output/part-r-00000
~
~
~
[xxx@localhost hadoop-docker]$ vi config/slaves

hadoop-slave1
hadoop-slave2
~
~
~
[xxx@localhost hadoop-docker]$ vi config/ssh_config

Host localhost
  StrictHostKeyChecking no

Host 0.0.0.0
  StrictHostKeyChecking no
  
Host hadoop-*
   StrictHostKeyChecking no
   UserKnownHostsFile=/dev/null
~
~
~
[xxx@localhost hadoop-docker]$ vi config/start-hadoop.sh

#!/bin/bash

echo -e "\n"

$HADOOP_HOME/sbin/start-dfs.sh

echo -e "\n"

$HADOOP_HOME/sbin/start-yarn.sh

echo -e "\n"
~
~
~
[xxx@localhost hadoop-docker]$ vi config/yarn-site.xml

<?xml version="1.0"?>
<configuration>
	<!--nodeManager获取数据的方式是shuffle-->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services.mapreduce_shuffle.class</name>
        <value>org.apache.hadoop.mapred.ShuffleHandler</value>
    </property>
    <!--指定Yarn的老大(ResourceManager)的地址-->
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hadoop-master</value>
    </property>
</configuration>
~
~
~
[xxx@localhost hadoop-docker]$ vi Dockerfile

# 镜像来源
FROM centos7-systemd

# 镜像创建者(写入自己的信息)
MAINTAINER "you" <your@email.here>

# 指定目录
WORKDIR /root

# 安装软件
RUN yum update -y && \
    yum install -y java-1.7.0-openjdk \
 		   openssh-server \
		   openssh-clients \
		   which

# 复制Hadoop
ENV HADOOP_VERSION=2.7.3
COPY hadoop-$HADOOP_VERSION.tar.gz /root/hadoop-$HADOOP_VERSION.tar.gz
# 安装
RUN tar -xzvf hadoop-$HADOOP_VERSION.tar.gz && \
    mv hadoop-$HADOOP_VERSION /usr/local/hadoop && \
    rm hadoop-$HADOOP_VERSION.tar.gz

# 设置环境变量
ENV JAVA_HOME=/usr/lib/jvm/jre-1.7.0-openjdk
ENV HADOOP_HOME=/usr/local/hadoop
ENV PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
ENV HADOOP_LIBEXEC_DIR=$HADOOP_HOME/libexec 

# ssh无密码登录
RUN ssh-keygen -t rsa -f ~/.ssh/id_rsa -P '' && \
    cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

# 复制配置文件及配置权限
COPY config/* /tmp/
RUN mv /tmp/ssh_config ~/.ssh/config && \
    mv /tmp/hadoop-env.sh /usr/local/hadoop/etc/hadoop/hadoop-env.sh && \
    mv /tmp/hdfs-site.xml $HADOOP_HOME/etc/hadoop/hdfs-site.xml && \ 
    mv /tmp/core-site.xml $HADOOP_HOME/etc/hadoop/core-site.xml && \
    mv /tmp/mapred-site.xml $HADOOP_HOME/etc/hadoop/mapred-site.xml && \
    mv /tmp/yarn-site.xml $HADOOP_HOME/etc/hadoop/yarn-site.xml && \
    mv /tmp/slaves $HADOOP_HOME/etc/hadoop/slaves && \
    mv /tmp/start-hadoop.sh ~/start-hadoop.sh && \
    mv /tmp/run-wordcount.sh ~/run-wordcount.sh
RUN chmod 600 ~/.ssh/config && \
    chmod +x ~/start-hadoop.sh && \
    chmod +x ~/run-wordcount.sh && \
    chmod +x $HADOOP_HOME/sbin/start-dfs.sh && \
    chmod +x $HADOOP_HOME/sbin/start-yarn.sh 

# 格式化namenode
RUN mkdir -p ~/hdfs/namenode && \ 
    mkdir -p ~/hdfs/datanode && \
    mkdir $HADOOP_HOME/logs
RUN $HADOOP_HOME/bin/hdfs namenode -format

# 启动容器后开启ssh服务
CMD [ "sh", "-c", "systemctl start sshd; bash"]
~
~
~
[xxx@localhost hadoop-docker]$ vi start-container.sh

#!/bin/bash

# 默认节点数3个(即一个master,两个slave)
N=${1:-3}


# 开启Hadoop-Master容器
sudo docker rm -f hadoop-master &> /dev/null
echo "start hadoop-master container..."
sudo docker run -itd \
       		    --privileged -e "container=docker" -v /sys/fs/cgroup:/sys/fs/cgroup \
                --net=hadoop \
                -p 50070:50070 \
                -p 8088:8088 \
                --name hadoop-master \
                --hostname hadoop-master \
                hadoop-docker &> /dev/null \
			   /usr/sbin/init

# 开启Hadoop-Slave容器
i=1
while [ $i -lt $N ]
do
	sudo docker rm -f hadoop-slave$i &> /dev/null
	echo "start hadoop-slave$i container..."
	sudo docker run -itd \
				   --privileged -e "container=docker" -v /sys/fs/cgroup:/sys/fs/cgroup \
	                --net=hadoop \
	                --name hadoop-slave$i \
	                --hostname hadoop-slave$i \
	                hadoop-docker &> /dev/null \
				  /usr/sbin/init
	i=$(( $i + 1 ))
done 

# 进入Hadoop-Master容器的命令行
sudo docker exec -it hadoop-master bash
~
~
~
[xxx@localhost hadoop-docker]$ sudo docker build -t hadoop-docker .
[xxx@localhost hadoop-docker]$ ./start-container.sh
@hadoop-master[root@hadoop-master ~]# ./start-hadoop.sh
@hadoop-master[root@hadoop-master ~]# ./run-wordcount.sh
@hadoop-master[root@hadoop-master ~]# exit
```
