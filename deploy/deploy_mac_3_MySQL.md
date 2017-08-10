# 部署：Mac环境部署之MySQL


#### 1、安装MySQL

<p>
到[http://dev.mysql.com/downloads/mysql/](http://dev.mysql.com/downloads/mysql/)下载最新的安装包dmg格式的，双击打开该dmg文件。

运行pkg，安装主程序包。

#### 2、配置环境变量

<p>
在终端输入：

	sudo chmod +w bashrc	sudo vi /etc/bashrc
在bashrc的末尾增加以下两个命令别名，便于快速使用mysql
	#mysql	alias mysql='/usr/local/mysql/bin/mysql'	alias mysqladmin='/usr/local/mysql/bin/mysqladmin'
（提示：在bashrc中添加命令别名之后，需要重新启动终端。）
#### 3、Root密码修改
<p>修改mysql默认密码，在终端输入：
	mysqladmin -u root password "123"
（其中123位置你可以指定任意密码。）
如果要更改密码可以输入：
	mysqladmin -u root -p password "123"
更改密码前先需要输入以前正确的密码才可以。