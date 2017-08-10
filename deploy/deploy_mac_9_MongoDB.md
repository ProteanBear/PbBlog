# 部署：Mac环境部署之MongoDB

MongoDB是一个基于分布式文件存储的数据库。由C++语言编写。旨在为WEB应用提供可扩展的高性能数据存储解决方案。

#### 安装

访问[http://www.mongodb.org/downloads](http://www.mongodb.org/downloads)下载MacOS下最新的MongoDB压缩包。

然后使用编译方式安装MongoDB，使用如下命令（以最新版本为mongodb-osx-x86_64-3.2.9.tgz为例）：

	tar xzf mongodb-osx-x86_64-3.2.9.tgz
	sudo mv mongodb-osx-x86_64-3.2.9 /usr/local/mongodb
	sudo mkdir /usr/local/mongodb_data /var/log/mongodb
	sudo chown -R root /usr/local/mongodb

#### 配置

	sudo vi /usr/local/mongodb/mongod.conf

增加如下配置内容：

	#/usr/local/mongodb/mongod.conf
		# 自定义数据库文件存储位置	dbpath = /usr/local/mongodb_data		# 只接受本地连接 	bind_ip = 127.0.0.1

#### 启动
	cd /usr/local/mongodb/bin	./mongod
#### 设置开机自启动
首先创建一个plist文件，创建lauch job，用来mongodb开机启动，关机停止，也设置一些日志输出：
	sudo vi /Library/LaunchDaemons/org.mongodb.mongod.plist
增加配置内容：	<plist version="1.0">
		<dict>
			<key>Label</key>
			<string>org.mongodb.mongod</string>
			<key>ProgramArguments</key>
			<array>
				<string>/Applications/mongodb/bin/mongod</string>
				<string>-f</string>
				<string>/Applications/mongodb/conf/mongod.conf</string>
  			</array>
  			<key>RunAtLoad</key>
  			<true/>
  			<key>KeepAlive</key>
  			<false/>
  			<key>WorkingDirectory</key>
  			<string>/Applications/mongodb</string>
  			<key>StandardErrorPath</key>
  			<string>/Applications/mongodb/log/output.log</string>
  			<key>StandardOutPath</key>
  			<string>/Applications/mongodb/log/output.log</string>
  			<key>HardResourceLimits</key>
  			<dict>
    			<key>NumberOfFiles</key>
    			<integer>1024</integer>
  			</dict>
  			<key>SoftResourceLimits</key>
  			<dict>
    			<key>NumberOfFiles</key>
    			<integer>1024</integer>
  			</dict>
		</dict>
	</plist>

plist 文件创建好后 执行如下命令加载到 开机启动中：
​	
	sudo launchctl load /Library/LaunchDaemons/org.mongodb.mongod.plist 

命令执行后 mongodb 将会马上启动，下次也会随开机而启动。