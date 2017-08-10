# 部署：Mac环境部署之Java


#### 1、安装Jdk
<p>

**注意：**这里主要还是安装Oracle的Jdk官方版本，非社区版本。

到Oracle官网下载最新的JDK安装包，安装。默认安装目录为：

	/Library/Java/JavaVirtualMachines/

​	
#### 2、如何同时安装多个版本
<p>

在用户目录下的bash配置文件.bashrc中配置JAVA_HOME的路径，使用命令：

	cd /etc	sudo chmod +w bashrc	sudo vi /etc/bashrc
然后在vi中编辑bashrc，配置JAVA_HOME的路径
	export JAVA_6_HOME=/System/Library/Java/JavaVirtualMachines/1.6.0.jdk/Contents/Home	export JAVA_7_HOME=/Library/Java/JavaVirtualMachines/jdk1.7.0.jdk/Contents/Home	export JAVA_8_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0.jdk/Contents/Home	export JAVA_HOME=$JAVA_7_HOME
创建alias命令动态切换JAVA_HOME的配置
	alias jdk8='export JAVA_HOME=$JAVA_8_HOME'	alias jdk7='export JAVA_HOME=$JAVA_7_HOME'	alias jdk6='export JAVA_HOME=$JAVA_6_HOME'
验证当前版本：
	java -version
#### 3、IDE安装
<p>

###### IntelliJ IDEA
访问[https://www.jetbrains.com/idea/](https://www.jetbrains.com/idea/)，下载最新版，激活方式网上搜索吧。

###### MyEclipse
访问[http://www.myeclipseide.cn/mac.html](http://www.myeclipseide.cn/mac.html)

###### Netbeans
访问[https://netbeans.org/downloads/](https://netbeans.org/downloads/)下载最新安装包，安装即可。