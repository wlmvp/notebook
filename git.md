## git安装
yum install git

一 、设置git：
设置git的user name和email：
$ git config --global user.name "xxx"
$ git config --global user.email "xxx@gmail.com"
查看git配置：
$git config --lis

二、生成SSH密钥过程：

1.查看是否已经有了ssh密钥：cd ~/.ssh
如果没有密钥则不会有此文件夹，有则备份删除

2.生存密钥：
$ ssh-keygen -t rsa -C "gudujianjsk@gmail.com"
按3个回车，密码为空这里一般不使用密钥。
最后得到了两个文件：id_rsa和id_rsa.pub

3.添加 私密钥 到ssh：ssh-add id_rsa
需要之前输入密码（如果有）。

4.在github上添加ssh密钥，这要添加的是“id_rsa.pub”里面的公钥。
打开 http://github.com,登陆xushichao，然后添加ssh。
注意在这里由于直接复制粘帖公钥，可能会导致增加一些字符或者减少些字符

##  TortoiseGit 

报错：Disconnected no supported authentication methods available(server sent: publickey)

解决：将客户端程序替换为git的ssh.exe的程序，这样在推送时会自动加载本地公钥，服务器就能验证通过了 

 ![img](.\images\20180607113335921.png) 

