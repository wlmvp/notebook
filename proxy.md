
## Shadowsocks配置

```bash
$ cat /etc/shadowsocks.json 
{
    "server":"c30s3.jamjams.net",
    "server_port":29915,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"ELhAQ7Mpx2",
    "timeout":300,
    "method":"aes-256-gcm",
    "fast_open": false
}

$ sslocal -c /etc/shadowsocks.json -d start
INFO: loading config from /etc/shadowsocks.json
2019-10-28 15:57:05 INFO     loading libcrypto from libcrypto.so.10
load libsodium failed with path None
started

```

## 安装libsodium

如果是gcm加密方式，需要安装libsodium

```bash
    #因为这库是基于C语言的，所以我们先去安装GCC
    yum -y groupinstall "Development Tools"
    #下载最新稳定版本
    wget https://download.libsodium.org/libsodium/releases/LATEST.tar.gz
    #解压
    tar xf LATEST.tar.gz && cd libsodium-1.0.11
    #编译
    ./configure && make -j4 && make install
    echo /usr/local/lib > /etc/ld.so.conf.d/usr_local_lib.conf
    ldconfig

```

## 一般代理配置


```bash
$ vi /etc/profile
export http_proxy="socks5://127.0.0.1:1080"
export https_proxy="socks5://127.0.0.1:1080"
```

## Github代理配置


```bash
$ git config --list

$ git config --global http.proxy 'socks5://127.0.0.1:1080'
$ git config --global https.proxy 'socks5://127.0.0.1:1080'

$ git config --global https.proxy http://127.0.0.1:1080
$ git config --global https.proxy https://127.0.0.1:1080

$ git config --global --unset http.proxy
$ git config --global --unset https.proxy
```
## Docker代理配置


```bash
[root@izttsn6fen661yz ~]# cd /etc/systemd/system/docker.service.d/
[root@izttsn6fen661yz docker.service.d]# ll
total 8
-rw-r--r-- 1 root root 59 Aug  5 18:36 http-proxy.conf
-rw-r--r-- 1 root root 60 Aug  5 18:37 https-proxy.conf
[root@izttsn6fen661yz docker.service.d]# 
[root@izttsn6fen661yz docker.service.d]# cat http-proxy.conf 
[Service]
Environment="HTTP_PROXY=socks5://127.0.0.1:1080"
[root@izttsn6fen661yz docker.service.d]# cat https-proxy.conf 
[Service]
Environment="HTTPS_PROXY=socks5://127.0.0.1:1080"
[root@izttsn6fen661yz docker.service.d]# 
```
## Golang代理配置

配置好上述一般配置加github配置即可

