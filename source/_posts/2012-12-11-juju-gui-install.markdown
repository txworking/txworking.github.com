---
layout: post
title: "Juju-Gui Install"
date: 2012-12-11 11:01
comments: true
categories: [juju, juju-gui, DevOps]
---


在Ubuntu12.04 AMD64上安装成功，基本按照项目中的README照做就行了。

1.juju-gui使用了Node.js和sphinx，所以需要先安装Node环境，jshint是可选的。

sudo add-apt-repository ppa:chris-lea/node.js
sudo apt-get update
sudo apt-get install nodejs npm
sudo npm install -g jshint
sudo apt-get install python-sphinx

2.从Lanunchpad下载代码

sudo apt-get install bzr
bzr launchpad-login yourid
bzr branch lp:juju-gui 
cd juju-gui
make server

这就可以访问localhost:8888看到界面了。要能操作还需要一个Juju环境，根据文档说Juju默认版本里面没有api-server这功能，最好是使用lp:~hazmat/juju/rapi-delta这个分支。

3.安装Juju

cd ~
sudo bzr branch lp:~hazmat/juju/rapi-delta
cd rapi-delta
python setup.py install

然后修改用户目录下的.juju/environments.yaml，在最后添加 api-port: 8081 ，特别要注意缩进，不然启动都出错。

再到bin目录下启动Juju

cd bin
sudo ./juju bootstrap

看到 Starting api server 就说明配置成功了

这时用netstat查看会发现8081端口并没有开始监听，需要先手动部署一个服务。

sudo ./juju deploy mysql

Juju默认是会到官方的Charm Store查找到mysql进行部署。

8081端口也会看到是listen状态了。

4.连接Gui和Juju

默认连接Juju的方式是在页面使用Websocket去连接http://localhost:8081/ws，所以需要修改配置把localhost改了，不然只能在本地访问。

修改Juju-gui里的config.js和app/config.js，把里面的localhost都改成服务器ip或者能解析的域名，重启一下服务，再访问就能看到已经有一个mysql部署好了，其他的charm也随你意安装了。