## java安装

wget https://download.oracle.com/otn-pub/java/jdk/8u191-b12/2787e4a523244c269598db4e85c51e0c/jdk-8u191-linux-x64.tar.gz?AuthParam=1544682402_fbd7d40c620473651a616693aea61863

mv jdk-8u191-linux-x64.tar.gz\?AuthParam\=1544682402_fbd7d40c620473651a616693aea61863 jdk-8u191-linux-x64.tar.gz

tar xzvf jdk-8u191-linux-x64.tar.gz

mkdir /usr/java
mv /root/jdk1.8.0_191 /usr/java/


vi /etc/profile

    JAVA_HOME=/usr/java/jdk1.8.0_191
    CLASSPATH=$JAVA_HOME/lib/
    PATH=$PATH:$JAVA_HOME/bin
    export PATH JAVA_HOME CLASSPATH

source vi /etc/profile

[root@localhost zipkin]# java -version
java version "1.8.0_191"
Java(TM) SE Runtime Environment (build 1.8.0_191-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.191-b12, mixed mode)
[root@localhost zipkin]# 