# 背景
 WebIDE 是为了解决函数计算本地环境差异和配置繁琐的问题。WebIDE 能让函数的开发、测试和部署更加流畅，降低了函数计算的学习成本和缩短了函数的开发周期。 允许用户无需安装任何程序就能使用标准的开发环境。

# 简介
## 简介

# 开发人员团队的Kubernetes-Native IDE

Eclipse Che使Kubernetes开发可用于开发人员团队，提供一键式开发人员工作区并消除了整个团队的本地环境配置。Che将您的Kubernetes应用程序带入您的开发环境中，并提供了一个浏览器内置IDE，使您可以完全在任何机器上的应用程序中对它们进行编码，构建，测试和运行。

 为任何语言/框架创建工作区。 

官方网站：https://operatorhub.io/operator/eclipse-che
GitHub Repo：https://github.com/eclipse/che-operator



## 基本原理

## 基本架构 

 我们正在建立一个世界，任何人都可以在不安装软件的情况下为项目做出贡献。Che定义了一种新型的基于容器的工作区，该工作区由项目和运行时组成，使其状态可分发，可移植且可版本化。 

![img](.\images\hero-home-technology.png) 

### 云端IDE

Che提供Che的默认基于Web的默认编辑器（基于Eclipse Theia），提供与大多数开发人员已经熟悉的Visual Studio Code相似的体验，并且基于最新的工具协议（例如Language Server Protocol和Language Server）构建。调试适配器。可以使用其他编辑器（Eclipse Dirigible，Jupyter）或从Eclipse Theia构建的编辑器。

### 工作区服务器

Che服务器控制工作区的生命周期，并且可以使用插件进行自定义。任何客户端都可以与工作区服务器和任何衍生的工作区进行通信。

### 集装箱式工作区

工作区是隔离的，是供开发人员工作的个人空间。无论开发人员是使用API，浏览器，CLI还是SSH访问工作区，他们的项目都是同步的并保持一致。

### 可扩展的

Che包括越来越多的插件和可编写代码的开发人员堆栈。您可以[创建和打包](https://www.eclipse.org/che/docs/che-7/developing-che-theia-plug-ins/)自己的文件。您还可以配置Che的[堆栈和插件。](https://www.eclipse.org/che/docs/che-7/customizing-the-devfile-and-plug-in-registries/)

### Eclipse Che如何工作

- 一键式集中托管工作区
- Kubernetes原生容器化开发
- 浏览器内可扩展IDE

### Kubernetes中的整个开发工作流程

Eclipse Che使开发人员团队可以更轻松，更快，更安全地进行kubernetes开发，提供一键式开发人员工作区，并为整个团队消除Docker或Kubernetes的本地安装和配置。Che将您的Kubernetes应用程序带入您的开发环境，并提供了一个可选的浏览器内置IDE，使您可以编码，构建，测试和运行应用程序。

 ![img](.\images\img-features-a-new-kind-of-workspace.png) 

### 团队的中央托管Kubernetes工作区

Eclipse Che在容器中运行。所有开发人员工具，IDE及其插件都作为容器化服务运行。您不必担心如何配置它们，安装它们的依赖项或保持它们的活动状态-一切都打包在容器中。车可以让你建立你的团队的开发环境和技术堆栈集中配置。

 ![img](.\images\img-features-kube-powered.png) 

### 浏览器内可扩展IDE

Eclipse Che带有一个基于Web的IDE，该IDE基于Eclipse Theia，它提供了浏览器内VSCode体验，并带有最新的工具协议：语言服务器，调试适配器以及**与VSCode扩展的兼容性。**如果您更喜欢台式机IDE，Che也支持它，那么您将获得Che的工作区的优势。Eclipse Che还使您能够与容器交互，指示命令和终端。

 <img src=".\images\img-features-cloud-ide-7.png" alt="img" style="zoom: 50%;" />



##### 支持的编程语言

 支持多种编程语言的语法突出显示，代码助手，调试器，堆栈和代码示例。同时还可以 支持git、ssh等工具。

- Java
- C++
- JavaSript
- Python
- PHP
- Node
- TypeScript
- YAML
- JSON
- Camel

# 安装部署

下载che-operator安装包，使用deploy脚本安装，安装之前使用kubectl替换openshift命令oc：

```
$ wget https://github.com/eclipse/che-operator/archive/7.3.0.tar.gz
$ ./deploy.sh
```

查看运行的pod
metalLB包含两个部分： a cluster-wide controller, and a per-machine protocol speaker.


```bash
[centos@k8s-master ~]$ kubectl get pod -n metallb-system  -o wide
NAME                          READY   STATUS    RESTARTS   AGE    IP              NODE         NOMINATED NODE   READINESS GATES
controller-7cc9c87cfb-n25kc   1/1     Running   1          166m   10.244.1.39     k8s-node1    <none>           <none>
speaker-cbhcg                 1/1     Running   1          166m   192.168.92.56   k8s-master   <none>           <none>
speaker-l6vv2                 1/1     Running   1          166m   192.168.92.58   k8s-node2    <none>           <none>
speaker-pxscm                 1/1     Running   1          166m   192.168.92.57   k8s-node1    <none>           <none>
[centos@k8s-master ~]$ 
```



当所有Eclipse Che容器都为running状态后，che-operator将打印Eclipse Che URL，可以通过搜索查看che-operator的日志得到Eclipse Che URL：

```bash
$ kubectl logs -f -n my-eclipse-che che-operator-7b6b4bcb9c-m4m2m | grep "Eclipse Che is now available"
time="2019-08-01T13:31:05Z" level=info msg="Eclipse Che is now available at: http://che-my-eclipse-che.gcp.my-ide.cloud"
```

# 源码修改编译

## che-server

https://github.com/eclipse/che

代码配置修改，编译生成assembly-wsmaster-war-7.3.0.war：

```bash
cd /root/che-operator/che-7.3.0/assembly/assembly-wsmaster-war
vi src/main/webapp/WEB-INF/classes/che/che.properties
mvn clean install
```

整体编译生成assembly-main压缩包：

```bash
cd /root/che-operator/che-7.3.0/assembly/assembly-main
mvn clean install
```

编译生成镜像：

```bash
cd /root/che-operator/che-7.3.0/dockerfiles
./build.sh
```

## che-plugin-registry

代码修改：

修改che-plugin-registry-7.3.0\v3\plugins\redhat\java8\latest\meta.yaml等文件中的

```
https://github.com/microsoft/vscode-java-debug/releases/download/0.20.0/vscode-java-debug-0.20.0.vsix
https://download.jboss.org/jbosstools/static/jdt.ls/stable/java-0.50.0-1825.vsix
```


 为

```
http://122.66.189.10/che/vscode-java-debug-0.20.0.vsix
http://122.66.189.10/che/java-0.50.0-1825.vsix
```

编译：

```
http://122.66.189.10/che/vscode-java-debug-0.20.0.vsix
http://122.66.189.10/che/java-0.50.0-1825.vsix
```

## che-operator

修改 pkg/deploy/extra_images.go中的

quay.io/eclipse/che-jwtproxy:dbd0578

为

eclipse/che-jwtproxy:dbd0578

```bash
cd /root/che-operator/che-operator-7.3.0
./release-operator-code.sh
```



# 测试使用

创建后端应用和服务测试

```bash
$ wget https://raw.githubusercontent.com/google/metallb/master/manifests/tutorial-2.yaml
$ kubectl apply -f tutorial-2.yaml
```

查看yaml文件配置，包含了一个deployment和一个LoadBalancer类型的service，默认即可。

```bash
[centos@k8s-master ~]$ vim tutorial-2.yaml 
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1
        ports:
        - name: http
          containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
```

查看service分配的EXTERNAL-IP

```bash
[centos@k8s-master ~]$ kubectl get service 
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1      <none>           443/TCP        3d15h
nginx        LoadBalancer   10.101.112.1   192.168.92.201   80:31274/TCP   123m
[centos@k8s-master ~]$
```


集群内访问该IP地址

```bash
[centos@k8s-master ~]$ curl 192.168.92.201
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
......
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[centos@k8s-master ~]$ 
```


ping该ip地址，发现从pod所在节点192.168.92.57发起的请求：

```bash
[centos@k8s-master ~]$ ping 192.168.92.201
PING 192.168.92.201 (192.168.92.201) 56(84) bytes of data.
From 192.168.92.57: icmp_seq=2 Redirect Host(New nexthop: 192.168.92.201)
From 192.168.92.57: icmp_seq=3 Redirect Host(New nexthop: 192.168.92.201)
From 192.168.92.57: icmp_seq=4 Redirect Host(New nexthop: 192.168.92.201)
```


从集群外访问该IP地址

另外使用nodeip+nodeport也可以访问（未知原因）

此IP地址在集群外无法ping通，但可以访问80端口：
命令行终端执行telnet 192.168.92.201 80测试成功。

到这里metallb loadbalancer部署完成，你可以定义kubernetes dashboard，granafa dashboard等各种应用服务，以loadbalancer的方式直接访问，不是一般的方便。

