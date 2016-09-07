#部署：Mac环境部署之Redis

Redis 是一个高性能的key-value内存数据库。

访问官方地址[http://www.redis.io/](http://www.redis.io/)下载最新的Redis应用包（这里以3.2.3为例）





	sudo vi /private/etc/redis.conf
	
找到daemonize 把 后面的no改成yes，不然每次启动会占用一个终端的session。

**给启动命令加上参数**

	alias redis-server="redis-server /etc/redis.conf"
	
**开启Redis**

	redis-server
	
要关闭Redis服务也很简单，先用Redis客户端连上Redis服务，用SHUTDOWN命令即可关闭服务

	$ redis-cli
	redis 127.0.0.1:6379> SHUTDOWN