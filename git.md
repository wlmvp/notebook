##  TortoiseGit 

报错：Disconnected no supported authentication methods available(server sent: publickey)

解决：将客户端程序替换为git的ssh.exe的程序，这样在推送时会自动加载本地公钥，服务器就能验证通过了 

 ![img](.\images\20180607113335921.png) 

