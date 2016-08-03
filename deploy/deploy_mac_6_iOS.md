#部署：Mac环境部署之iOS


####1、安装Xcode
<p>

&#160; &#160; &#160; &#160;没啥说的，直接AppStore上下载最新的（慢就慢点吧，免得糟了XcodeGhost）。


####2、安装CocoaPods
<p>

&#160; &#160; &#160; &#160;CocoaPods是一个负责管理iOS项目中第三方开源库的工具。有了CocoaPods这个工具之后，只需要将用到的第三方开源库放到一个名为Podfile的文件中，然后在命令行执行$ pod install命令。CocoaPods就会自动将这些第三方开源库的源码下载下来，并且为我的工程设置好相应的系统依赖和编译参数。

&#160; &#160; &#160; &#160;安装的方式非常简单，Mac下已经自带了ruby，只要使用ruby的gem命令就可以安装了。打开的Mac的终端，在终端运行下面的命令：

	sudo gem install cocoapods
	
&#160; &#160; &#160; &#160;**但是，且慢！**如果你在天朝，在终端中敲入这个命令之后，会发现半天没有任何反应。原因无他，因为那堵墙阻挡了cocoapods.org。

&#160; &#160; &#160; &#160;但是，是的，又但是（不过是个可喜的“但是”）。我们可以用淘宝的Ruby镜像来访问cocoapods。按照下面的顺序在终端中敲入依次敲入命令：

	gem sources --remove https://rubygems.org/
	//等有反应之后再敲入以下命令
	gem sources -a http://ruby.taobao.org/
	
&#160; &#160; &#160; &#160;为了验证你的Ruby镜像是并且仅是taobao，可以用以下命令查看：

	gem sources -l
	
&#160; &#160; &#160; &#160;只有在终端中出现下面文字才表明你上面的命令是成功的：

	*** CURRENT SOURCES ***	http://ruby.taobao.org/
&#160; &#160; &#160; &#160;这时候，你再次在终端中运行：
	sudo gem install cocoapods
&#160; &#160; &#160; &#160;等上十几秒钟，CocoaPods就可以在你本地下载并且安装好了，使用如下命令安装环境：
	pod setup
	
####3、更新CocoaPods
	sudo gem install -n /usr/local/bin cocoapods

####4、使用CocoaPods
&#160; &#160; &#160; &#160;命令行进入项目主目录，创建Podfile文件，写入如下内容：

	//指定iOS系统的版本	platform:ios,"8.0"
	//指定源地址（可以指定自己定制组件的地址）
	source 'https://github.com/CocoaPods/Specs.git'
	
	use_frameworks!
	target '项目名称’ do
		//组件名称（不写版本为最新版）
    	pod 'AFNetworking','->2.0'
	end
	
&#160; &#160; &#160; &#160;执行如下命令：

	pod install
	
&#160; &#160; &#160; &#160;创建好后就可使用xworkspace打开项目。