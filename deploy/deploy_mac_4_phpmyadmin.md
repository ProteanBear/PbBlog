#部署：Mac环境部署之phpMyAdmin


####1、安装phpMyAdmin
<p>
到http://www.phpmyadmin.net/home_page/index.php下载最新的phpMyAdmin版本。

解压后重命名为phpmyadmin，复制到：

	/Library/WebServer/Documents/
	
复制其中的config.sample.inc.php，并命名为config.inc.php

打开config.inc.php,做如下修改：
	
	//用于Cookie加密，添加随意的长字符串
	$cfg['blowfish_secret'] = '';
	
**注意：**当phpMyAdmin中出现“#2002 无法登录 MySQL 服务器”时，请把localhost改成127.0.0.1就ok了，这是因为MySQL守护程序做了IP绑定（bind-address =127.0.0.1）造成的。

	$cfg['Servers'][$i]['host'] = '127.0.0.1';
	
(还可以安装Navicat来连接MySQL)