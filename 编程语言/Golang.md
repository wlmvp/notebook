## Golang安装

下载 Go 语言文件

    wget https://studygolang.com/dl/golang/go1.11.linux-amd64.tar.gz

下载地址：https://studygolang.com/dl

解压二进制文件到 /usr/local 目录

    tar -xzf go1.11.linux-amd64.tar.gz -C /usr/local

使用 vi 在环境变量配置文件  /etc/profile 中增加如下内容：

    export PATH=$PATH:/usr/local/go/bin

检查 Go 语言版本

    go version

定义 GOPATH 环境变量到 workspace 目录

    export GOPATH=$HOME/workspace



## Golang使用笔记

如果项目不在GOPATH下面，并且自带mod，vendor下面的代码不起作用。



 项目带Makefile时，构建方式为：make build 