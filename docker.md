## docker hub加速访问设置

1.修改或创建容器

```bash
$ cat /etc/shadowsocks.json 
{ "registry-mirrors" :["https://docker.mirrors.ustc.edu.cn"]}
```


国内较快的镜像原地址:

#Docker 官方中国区:https://registry.docker-cn.com
#网易    http://hub-mirror.c.163.com
#ustc中国科技大学   https://docker.mirrors.ustc.edu.cn

行内可以修改为 http://dockerhub.icbc:5000

2.重启docker，就可以非常快速的拉取所需的镜像

```bash
$ systemctl daemon-reload
$ systemctl restart docker
```

## 

