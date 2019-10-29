# 背景
Kubernetes

# 简介
## 简介

官方网站：https://operatorhub.io/operator/eclipse-che
GitHub Repo：https://github.com/eclipse/che-operator



## 基本原理



# 安装部署

部署metallb负载均衡器
Metallb 支持 Helm 和 YAML 两种安装方法，这里我们使用第二种：

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

