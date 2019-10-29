## Python3安装

```bash
$ wget https://www.python.org/ftp/python/3.8.0/Python-3.8.0.tgz
$ tar -zxvf Python-3.8.0.tgz
$ cd Python-3.8.0
$ ./configure --prefix=/usr/local/python3
$ make && make install
$ ln -s /usr/local/python3/bin/python3.8 /usr/bin/python3
$ python3 -V
Python 3.8.0
```

如报错：zipimport.ZipImportError: can't decompress data; zlib not available

则安装改lib依赖：yum -y install zlib* 