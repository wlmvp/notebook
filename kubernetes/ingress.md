﻿# 背景
**私有云裸金属架构的kubernetes集群不支持LoadBalance**
Kubernetes没有为裸机群集提供网络负载均衡器（类型为LoadBalancer的服务）的实现，如果你的kubernetes集群没有在公有云的IaaS平台（GCP，AWS，Azure …）上运行，则LoadBalancers将在创建时无限期地保持“挂起”状态，也就是说只有公有云厂商自家的kubernetes支持LoadBalancer。
裸机群集运营商留下了两个较小的工具来将用户流量带入其集群，“NodePort”和“externalIPs”服务。这两种选择都对生产使用产生了重大影响，类型为 LoadBalancer 的服务在 Kubernetes 中并没有直接支持，NodePort 和 ExternalIP 方案让很多私有云用户成为了 K8S 世界中的二等公民。

![在这里插入图片描述](.\images\20191024160316678.png)
![在这里插入图片描述](.\images\2019102416041446.png)

# MetalLB简介
## 简介



# **Ingress 控制器**

这一节将从 Kubernetes 官方控制器开始，逐渐扩展到其他广为人知的 Ingress 控制器。

## **Kubernetes Ingress Controller**

github.com/kubernetes/ingress-nginx

实现：Go/Lua（nginx 是用 C 写的）

许可证：Apache 2.0

Kubernetes 的“官方”控制器（之所以称为官方，是想把它区别于 NGINX 公司的控制器）。这是社区开发的控制器，它基于 nginx Web 服务器，并补充了一组用于实现额外功能的 Lua 插件。

由于 NGINX 十分流行，再加上把它用作控制器时所需的修改较少，**它对于 K8s 普通工程师来说，可能是最简单和最直接的选择**。

## **NGINX Ingress Controller**

github.com/nginxinc/kubernetes-ingress

实现：Go

许可证：Apache 2.0

这是 NGINX 公司开发的官方产品，它也有一个基于 NGINX Plus 的商业版。NGINX 的控制器具有很高的稳定性、持续的向后兼容性，且没有任何第三方模块。

由于消除了 Lua 代码，和官方控制器相比，它保证了较高的速度，但也因此受到较大限制。相较之下，它的付费版本有更广泛的附加功能，如实时指标、JWT 验证、主动健康检查等。

NGINX Ingress 重要的优势是对 TCP/UDP 流量的全面支持，最主要缺点是缺乏流量分配功能。

## **其他Controller**

其他比较关注度较高的还有如下的一些Ingress Controller：Kong Ingress，Traefik，HAProxy Ingress，Voyager，Contour，Istio Ingress，Ambassador，Gloo**，**Skipper。

 ![img](.\images\f771ff4db28b5f02efe5122be7aee102.png) 

## 基本原理

Metallb 会在 Kubernetes 内运行，监控服务对象的变化，一旦察觉有新的LoadBalancer 服务运行，并且没有可申请的负载均衡器之后，就会完成两部分的工作：
1.地址分配
用户需要在配置中提供一个地址池，Metallb 将会在其中选取地址分配给服务。
2.地址广播
根据不同配置，Metallb 会以二层（ARP/NDP）或者 BGP 的方式进行地址的广播。
关于裸金属集群的负载均衡方案介绍
https://kubernetes.github.io/ingress-nginx/deploy/baremetal/
基本原理图：

![在这里插入图片描述](.\images\20191024160741157.png)

### 支持范围

不支持 IPVS

| 网络插件    | 兼容性                 |
| ----------- | ---------------------- |
| Calico      | 部分支持（有附加文档） |
| Flannel     | 支持                   |
| Kube-router | 不支持（正在跟进）     |
| Romana      | 支持（有附加文档）     |
| Weave Net   | 支持                   |

# 安装部署

部署metallb负载均衡器
Metallb 支持 Helm 和 YAML 两种安装方法，这里我们使用第二种：

```
$ wget https://raw.githubusercontent.com/google/metallb/v0.8.1/manifests/metallb.yaml
$ kubectl apply -f metallb.yaml
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

查看其它信息
包含了 “controller” deployment,和 the “speaker” DaemonSet.

```bash
[centos@k8s-master ~]$ kubectl get daemonset -n metallb-system 
NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
speaker   3         3         3       3            3           <none>          13h
[centos@k8s-master ~]$ kubectl get deployment -n metallb-system 
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
controller   1/1     1            1           13h
[centos@k8s-master ~]$ 
```

目前还没有宣布任何内容，因为我们没有提供ConfigMap，也没有提供负载均衡地址的服务。
接下来我们要生成一个 Configmap 文件，为 Metallb 设置网址范围以及协议相关的选择和配置，这里以一个简单的二层配置为例。

创建config.yaml提供IP池
wget https://raw.githubusercontent.com/google/metallb/v0.7.3/manifests/example-layer2-config.yaml
1
修改ip地址池和集群节点网段相同

```bash
[centos@k8s-master ~]$ vim example-layer2-config.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.92.200-192.168.92.210
```


注意：这里的 IP 地址范围需要跟集群实际情况相对应。
执行yaml文件

```bash
kubectl apply -f example-layer2-config.yaml
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

