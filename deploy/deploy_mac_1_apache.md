#部署：Mac环境部署之Apache
MacOS中自带了Apache，因此无需安装，只需要配置一下即可。

####1、启动Apache
	sudo apachectl start
打开浏览器，输入http://localhost

应该可以看到”It works!“的页面，该页面位于
/Library/WebServer/Documents/目录下，这是Apache的默认根目录。

####2、关闭Apache
	sudo apachectl stop

####3、重启Apache
	sudo apachectl restart