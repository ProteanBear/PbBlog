#部署：Mac环境部署之php
Mac OS X自带PHP功能，但默认是不开启的。

####1、配置Apache

在终端中输入：

	sudo vi /etc/apache2/httpd.conf
	
在vi中，输入/php搜索包含php的文本，找到：

	#LoadModule php5_module libexec/apache2/libphp5.so

删除前面的#，然后保存退出。（按shift+i行首输入，按ESC退出编辑，按x删除当前字符，及#，输入:wq，保存并退出。）

####2、增加php配置文件

在终端中输入：

	cd /etc	sudo cp php.ini.default php.ini

####3、重启Apache

	sudo apachectl restart
	
####4、测试php

在终端输入（Apache的Web根目录默认是/Library/WebServer/Documents/）：

	cd /Library/WebServer/Documents/
	sudo vi info.php
	
然后在info.php中输入以下内容：
	
	<html>
		<body>
			<?php phpinfo(); ?>
		</body>
	</html>

然后在浏览器中输入：
	
	http://localhost/info.php
	
显示php信息界面，则php配置成功。