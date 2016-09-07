#部署：Mac环境部署之Nginx

未安装homebrew的看[这里](deploy_mac_7_homebrew.md)怎么安装

安装好homebrew后使用brew命令

**安装**

	brew search nginx
	brew install nginx

**安装成功后台配置文件所在位置**

	/usr/local/etc/nginx/nginx.conf
	
**查看版本**

	nginx -v
	
**指定配置文件**

	nginx -c filename
