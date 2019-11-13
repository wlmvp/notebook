# WSL安装

## 第一步：启用虚拟机平台和 Linux 子系统功能

```
Enable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform
```

以管理员权限启动 PowerShell，然后输入以下命令启用 Linux 子系统功能：

```
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```

在以上每一步命令执行完之后，PowerShell 中可能会提示你重新启动计算机。按“Y”可以重新启动。

![启用 VirtualMachinePlatform](https://walterlv.com/static/posts/2019-07-05-08-22-01.png)

![启用 Microsoft-Windows-Subsystem-Linux](https://walterlv.com/static/posts/2019-07-05-08-25-26.png)

![正在启用 Linux 子系统](https://walterlv.com/static/posts/2019-07-05-08-26-11.png)

当然，这个命令跟你在控制面板中启用“适用于 Windows 的 Linux 子系统”功能是一样的。

![在控制面板中启用虚拟机平台和 Linux 子系统](https://walterlv.com/static/posts/2019-07-05-08-53-22.png)

## 第二步：安装一个 Linux 发行版



![搜索 Linux](https://walterlv.com/static/posts/2019-07-05-08-30-08.png)

选择一个 Linux 发行版本然后安装：

![安装一个 Linux 发行版](https://walterlv.com/static/posts/2019-07-05-08-31-34.png)

需要注意，在商店中的安装并没有实际上完成 Linux 子系统的安装，你还需要运行一次已安装的 Linux 发行版以执行真正的安装操作。

![安装 Linux](https://walterlv.com/static/posts/2019-07-05-09-26-03.png)



- 管理员模式打开ubuntu18.04
- `sudo passwd` 输入密码与root密码
- `su root` 输入密码

但是这时每次打开ubuntu的时候还是普通用户，我们可以使用命令设置默认用户
`ubuntu config --default-user root`



## 第三步：启用 WSL2



使用 `wsl -l` 可以列出当前系统上已经安装的 Linux 子系统名称。注意这里的 `-l` 是列表“list”的缩写，是字母 `l` 不是其他字符。

```
wsl -l
```

如果提示 `wsl` 不是内部或外部命令，说明你没有启用“适用于 Windows 的 Linux 子系统”，请先完成本文第一步。

如果提示没有发现任何已安装的 Linux，说明你没有安装 Linux 发行版，或者只是去商店下载了，没有运行它执行真正的安装，请先完成本文第二步。

使用 `wsl --set-version  2` 命令可以设置一个 Linux 发行版的 WSL 版本。命令中 `` 替换为你安装的 Linux 发型版本的名称，也就是前面通过 `wsl -l` 查询到的名称。

本文的示例使用的是小白门喜欢的 Ubuntu 发行版。

```
wsl --set-version Ubuntu> 2
```

![设置 WSL2](https://walterlv.com/static/posts/2019-07-05-10-12-35.png)

当然，使用以下命令可以在以后安装 Linux 的时候默认启用 WSL2：

```
wsl --set-default-version 2
```

# Docker安装

因为都知道的网络原因安装时可能会timeout等其他情况,我们可以使用国内镜像 https://mirrors.tuna.tsinghua.edu.cn/ 清华大学开源软件镜像站替换下面的链接
`https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu/gpg`
`https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu/`

- `sudo apt-get remove docker docker-engine docker.io`
- `sudo apt-get update`
- `sudo apt-get install \ apt-transport-https \ ca-certificates \ curl \ software-properties-common`
- `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`
- `sudo add-apt-repository \ "deb [arch=amd64] https://download.docker.com/linux/ubuntu \ $(lsb_release -cs) \ stable"`
- `sudo apt-get update`
- `sudo apt-get install docker-ce`

一顿操作已经安装结束,使用 `sudo service docker start` 开启docker守护进程
使用 `docker version` 查看版本

注意：如果`sudo service docker start`启动成功，请使用管理员身份打开ubuntu进行操作。

# SSH登录

自带的ssh有问题，卸载重装：

```shell
sudo apt-get remove openssh-server
sudo apt-get install openssh-server
```

编辑sshd_config文件，修改几处配置才能正常使用用户名/密码的方式连接

```shell
sudo vi /etc/ssh/sshd_config

Port 22 #默认即可，如果有端口占用可以自己修改`
PasswordAuthentication yes # 允许用户名密码方式登录
PermitRootLogin yes # 允许直接root用户登录
```

修改完之后重启ssh服务

```undefined
sudo service ssh restart
```

接下来就可以直接xShell登录了，使用pc网卡ip登录即可：

```shell
root@LAPTOP-S8SPAS15:~# ifconfig
```

