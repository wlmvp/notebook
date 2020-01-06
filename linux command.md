## 查看端口

```bash
netstat -ntlp   //查看当前所有tcp端口·
netstat -ntulp |grep 80   //查看所有80端口使用情况·
netstat -an | grep 3306   //查看所有3306端口使用情况·
查看一台服务器上面哪些服务及端口
netstat  -lanp
查看一个服务有几个端口。比如要查看mysqld
ps -ef |grep mysqld
查看某一端口的连接数量,比如3306端口
netstat -pnt |grep :3306 |wc
查看某一端口的连接客户端IP 比如3306端口
netstat -anp |grep 3306
```

查找字符串的文件

```
grep -rin "hello,world!" ./

./ : 表示路径为[当前目录](http://www.baidu.com/s?wd=当前目录&tn=SE_PcZhidaonwhc_ngpagmjz&rsv_dl=gh_pc_zhidao).

-r 是递归查找

-n 是显示行号 

i,忽略大小写
```

