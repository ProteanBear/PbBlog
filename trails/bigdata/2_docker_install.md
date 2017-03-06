#杂学札记之大数据系列：CentOS7下Docker安装和配置

Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。容器是完全使用沙箱机制，相互之间不会有任何接口。

详细介绍就不普及了，下面内容来自[Docker官网](https://www.docker.com/)给的[CentOS下Docker的安装指导](https://docs.docker.com/engine/installation/linux/centos/)和自我实践。



### 目录索引

[前提条件](#前提条件)

​	— [删除方法](#删除方法)

[安装Docker](#安装docker)

​	— [方法一：使用存储库安装](#方法一使用存储库安装)

​	—— [设置存储库](#设置存储库)

​	—— [安装Docker](#安装docker-1)

​	—— [升级Docker的方法](#升级docker的方法)

​	— [方法二：下载包安装](#方法二下载包安装)

[Docker配置](#Docker配置)

​	— [让非Root用户管理Docker](#让非root用户管理docker)

​	— [开机自动启动Docker](#开机自动启动docker)

​	— [Docker加速器](#docker加速器)

[附：安装命令纯净版](#附docker安装命令纯净版)



### 前提条件

1. 64位版本的CentOS7（系统安装看[这里](deploy_centos_0_install.md)）
2. 删除非官方的Docker包

###### 删除方法：

Red Hat的操作系统存储库包含旧版本的Docker，使用程序包名称`docker`而不是`docker-engine`。如果您安装了此版本的Docker，请使用以下命令删除它：

```
$ sudo yum -y remove docker docker-common container-selinux
```

您可能还需要删除`docker-selinux`与官方`docker-engine`软件包冲突的软件包。使用以下命令删除它：

```
$ sudo yum -y remove docker-selinux
```

（命令不会删除`/var/lib/docker`中的内容，因此使用旧版本的Docker创建的任何图像，容器或卷都会保留。）



> **本人实践：**
>
> ​	因为我是CentOS7最小安装，这些东西都没有╮(╯_╰)╭！



### 安装Docker

可以根据不同的需求以不同方式安装Docker，包括：

- 设置Docker's repositories并从中安装，以方便安装和升级任务。这是推荐的方法。
- 某些用户下载RPM软件包并手动安装并完全手动管理升级。
- 一些用户不能使用第三方存储库，并且必须依赖于CentOS存储库中的Docker版本。



#### 方法一：使用存储库安装

第一次安装需要先设置存储库，以后就可以从存储库进行安装、更新或降级Docker了。

###### 设置存储库

1. 安装`yum-utils`，它提供`yum-config-manager`实用程序：

   ```
   $ sudo yum install -y yum-utils
   ```

2. 使用一下命令设置稳定版本的存储库：

   ```
   $ sudo yum-config-manager \
       --add-repo \
       https://docs.docker.com/engine/installation/linux/repo_files/centos/docker.repo
   ```

3. 可选：启动测试存储库。此存储库包含在`docker.repo`上面的文件中，但默认情况下禁用。您可以在稳定存储库旁边启用它。**不要在生产系统或非测试工作负载上使用不稳定的存储库。**

   > **警告**：如果启用了稳定和不稳定的存储库，则在`yum install` or `yum update`命令中指定版本的安装或更新将始终安装尽可能高的版本，这几乎肯定是不稳定的版本。

   ```
   $ sudo yum-config-manager --enable docker-testing
   ```

   您可以`testing`通过运行`yum-config-manager` 带有`--disable`标志的命令来禁用存储库。要重新启用它，请使用 `--enable`标志。以下命令禁用存储`testing` 库。

   ```
   $ sudo yum-config-manager --disable docker-testing
   ```

###### 安装Docker

1. 更新`yum`包索引。

   ```
   $ sudo yum makecache fast
   ```

   如果这是您在添加Docker存储库之后第一次刷新包索引，将提示您接受GPG密钥，并且将显示密钥的指纹。验证指纹是否匹配`58118E89F3A912897C070ADBF76221572C52609D`，如果匹配 ，请接受密钥。

2. 安装最新版本的Docker，或转到下一步安装特定版本。

   ```
   $ sudo yum -y install docker-engine
   ```

   > **警告**：如果启用了稳定和不稳定的存储库，则安装或更新Docker而不在 `yum install`or `yum upgrade`命令中指定版本将始终安装最高可用版本，这几乎肯定是不稳定版本。

3. 在生产系统上，您应该安装特定版本的Docker，而不是总是使用最新版本。列出可用的版本。此示例使用`sort -r`命令按版本号对结果进行排序，从最高到最低，并被截断。

   > **注意**：此`yum list`命令仅显示二进制包。要显示源包以及，`.x86_64`从包名称中省略。

   ```
   $ yum list docker-engine.x86_64  --showduplicates |sort -r

   docker-engine.x86_64  1.13.0-1.el7                               docker-main
   docker-engine.x86_64  1.12.5-1.el7                               docker-main   
   docker-engine.x86_64  1.12.4-1.el7                               docker-main   
   docker-engine.x86_64  1.12.3-1.el7                               docker-main 
   ```
   列表的内容取决于启用哪些存储库，并且将特定于您的版本的CentOS（`.el7`在本示例中由版本上的后缀指示）。选择要安装的特定版本。第二列是版本字符串。第三列是存储库名称，指示软件包来自哪个存储库，其扩展名其稳定性级别。要安装特定版本，请将版本字符串附加到软件包名称，并用连字符（`-`）分隔它们：

   ```
   $ sudo yum -y install docker-engine-<VERSION_STRING>
   ```

4. 启动Docker。

   ```
   $ sudo systemctl start docker
   ```

5. `docker`通过运行`hello-world` 映像验证是否已正确安装。

   ```
   $ sudo docker run hello-world
   ```

   此命令下载测试映像并在容器中运行它。当容器运行时，它打印一个信息消息并退出。

Docker已安装并运行。您需要使用`sudo`运行Docker命令。如果想允许非特权用户运行Docker命令，请参见后面的配置部分。

###### 升级Docker的方法

要升级Docker，首先运行`sudo yum makecache fast`，然后再选择新版本进行安装。



> **本人实践：**
>
> ​	设置稳定版本那句命令为了显示用了”\“换行，其实就是一句命令下来的，其他执行正常安装成功。



#### 方法二：下载包安装

如果不能使用Docker的存储库来安装Docker，可以下载该`.rpm`版本的 文件并手动安装。每次要升级Docker时，都需要下载新文件。

1. 转到[https://yum.dockerproject.org/repo/main/centos/](https://yum.dockerproject.org/repo/main/centos/) 并选择您的CentOS版本的子目录。下载`.rpm`要安装的Docker版本的文件。

2. 安装Docker，将下面的路径更改为您下载Docker包的路径。

   ```
   $ sudo yum -y install /path/to/package.rpm
   ```

3. 启动Docker。

   ```
   $ sudo systemctl start docker
   ```

4. `docker`通过运行`hello-world` 映像验证是否已正确安装。

   ```
   $ sudo docker run hello-world
   ```

同样，您需要使用`sudo`运行Docker命令。如果想允许非特权用户运行Docker命令，请参见后面的配置部分。



### Docker配置

#### 让非Root用户管理Docker

默认情况下，其他用户只能使用`sudo`来使用root账号运行Docker。如果在使用`docker`命令时不想使用`sudo`，需要创建一个名为`docker`的用户组，并将当前用户添加进去。

1. 创建`docker`组。

   ```
   $ sudo groupadd docker
   ```

2. 将您的用户添加到`docker`组。

   ```
   $ sudo usermod -aG docker $USER
   ```

3. 注销并重新登录，以便重新评估您的组成员资格。

4. 验证您可以`docker`没有命令`sudo`。

   ```
   $ docker run hello-world
   ```

#### 开机自动启动Docker

开启：

```
$ sudo systemctl enable docker
```

关闭：

```
$ sudo systemctl disable docker
```

#### Docker加速器

万恶的那啥，在国内连接Docker Hub非常的不稳定，还好有[DaoCloud](https://www.daocloud.io/)的加速器。注册用户登录后，选择[加速器](https://www.daocloud.io/mirror#accelerator-doc)。根据提示就可以将 --registry-mirror 加入到你的 Docker 配置文件 /etc/default/docker 中，方便国内用户使用。



### 附Docker安装命令纯净版

```
[xxx@localhost ~]$ su root
Password:
[root@localhost ~]$ yum -y remove docker docker-common container-selinux docker-selinux
[root@localhost ~]$ yum install -y yum-utils
[root@localhost ~]$ yum-config-manager --add-repo https://docs.docker.com/engine/installation/linux/repo_files/centos/docker.repo
[root@localhost ~]$ yum-config-manager --disable docker-testing
[root@localhost ~]$ yum makecache fast
[root@localhost ~]$ yum -y install docker-engine
[root@localhost ~]$ docker version
[root@localhost ~]$ docker run hello-world
```
