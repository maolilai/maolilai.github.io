git clone https://GitHub.com/maolilai/maolilai.GitHub.io


git config --global 497248666@qq.com
git config --global maolilai


  git config --global user.email "497248666@qq.com"
  git config --global user.name "maolilai"
  
  
 --- gengxin update 
git add --all
git commit -m "Firs Push"
git push -u origin master


---- 


环境需求：  在Centos7.3中，通过yum安装ruby的版本是2.0.0，但是如果有些应用需要高版本的ruby环境，比如2.2，2.3，2.4...

　　　　　　那就有点麻烦了，譬如：我准备使用redis官方给的工具：redis-trib.rb 这个工具构建redis集群的时候，报错了：

　　　　　　　　　　　　　　　　　　“redis requires Ruby version >= 2.2.2”

　　　　　　网上搜索了一圈，概括为以下三种方法：

　　　　　　①添加yum源安装：CentOS SCLo Software collections Repository（简单快捷）

　　　　　　②下载tar压缩包安装（略微繁琐）

　　　　　　③RVM（Ruby Version Manager）安装（相对官方）

 　　　　　　

具体操作：

　　方法一：换yum源安装

　　　　~]# yum install centos-release-scl-rh　　　　//会在/etc/yum.repos.d/目录下多出一个CentOS-SCLo-scl-rh.repo源

　　　　~]# yum install rh-ruby23  -y　　　　//直接yum安装即可　　

　　　　~]# scl  enable  rh-ruby23 bash　　　　//必要一步

　　　　~]# ruby -v　　　　//查看安装版本


简介
CentOS 7 自带的 Ruby 版本太低，因此需要使用 rvm 安装较新版本的 Ruby。
注，自带的ruby版本是2.0.0， 安装Jekyll要求的版本在2.1以上，我们选择最新稳定版2.5.1。

安装 rvm:
gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
curl -sSL https://get.rvm.io | bash -s stable
source /home/crazy5/.rvm/scripts/rvm
1
2
3
注：最后一句并不是网上的“source /etc/profile.d/rvm.sh”这句，而是rvm安装成功后自动给出的提示

安装 ruby 2.5.1：
sudo yum install libyaml
rvm install 2.5.1
1
2
系统会自动下载ruby2.5.1源代码进行编译，速度较慢,ruby 2.5.1最终安装在：

/home/.rvm/gems/ruby-2.5.1
1
再次运行:

source /home/crazy5/.rvm/scripts/rvm
设置ruby默认版本:
rvm use 2.5.1 --default
1
2
3
安装 Nodejs:
sudo yum install nodejs
1
修改 gem 源:
使用中科大源：

gem sources --remove https://rubygems.org/
gem sources -a http://mirrors.ustc.edu.cn/rubygems/
1
2
安装 Jekyll:
gem install jekyll
1
修改 ./bashrc
再次打开bash时，发现ruby版本又回到旧版，没法运行jekyll。
所以要修改 .bashrc 文件，是因为 RVM 将作为 Shell 函数使用，我们需要在初始化 Shell 时（如打开终端窗口或执行某个命令）加载 RVM。
向home目录下的./bashrc的最后一行加入：

source "$HOME/.rvm/scripts/rvm"
1
RVM 是否安装成功：

$ type rvm | head -n1
1
如果返回 “rvm is a function”/ rvm是一个函数，则说明一切正常。

后记：
如果在安装 RVM 之前我们已经在系统上安装了 Ruby，我们可以使用这个命令使用系统上原有的 Ruby 版本：

$ rvm system
1
当然，也可以指定系统原有的 Ruby 为默认的版本：

$ rvm system –default
1
若同时安装 2.4.1 和2.5.1 两个版本：

$ rvm install 2.4.1
或
$ rvm install 2.5.1
进行切换
————————————————
版权声明：本文为CSDN博主「hooyying」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/hooyying/article/details/83119948