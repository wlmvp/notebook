# 背景

Kubernetes和容器对于有状态应用的支持不佳，因为有状态应用对某些外部资源（如远程存储、网络设备等）有着绑定性的依赖，有状态应用的多个实例之间往往有着拓扑关系，而所有的分布式应用几乎都是有状态应用。  

在没有Operator之前，部署分布式应用需要编写一套复杂的管理脚本，学习大量与项目本身无关的运维知识，造成维护成本和使用门槛非常高。 

Kubernetes Operator 已经成了开发和部署分布式应用的一项事实标准。 Kubernetes Operator 发布之初最大的意义，就在于它将分布式应用的使用门槛直接降到了最低。 

现今 Kubernetes Operator 的意义已经远远超过了“分布式应用部署”的这个原始的范畴，已经成为了容器化时代应用开发与发布的一个全新途径。 


# Operator简介
Operator最初由CoreOS于2016年推出，并已被Red Hat和Kubernetes社区用作打包、部署和管理Kubernetes原生应用程序的方法。Kubernetes原生应用程序是一个部署在Kubernetes上的应用程序，使用Kubernetes API和众所周知的工具进行管理，如kubectl。 

 Operator实现为自定义控制器，用于监视某些Kubernetes资源的显示、修改或删除。这些通常是Operator“拥有”的CustomResourceDefinition。在这些对象的spec属性中，用户声明应用程序或操作的所需状态。Operator的协调循环将选择这些，并执行所需的操作以实现所需的状态。 

Kubernetes 作为领先的容器编排项目之所以如此成功，其中一个原因就在于其可扩展性。借助[自定义资源](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources)，开发者可以扩展  Kubernetes API 以管理超出原生对象范围外的资源（例如，pod 和服务）。此外，Kubernetes Go Client  提供了强大的资源库，用于为自定义资源编写*控制器*。控制器实施闭环控制逻辑，通过持续运行对资源的期望状态与所观测到的状态进行协调。

Operator将特定于应用程序的控制器与相关自定义资源组合在一起，编撰特定领域的知识，用于管理资源生命周期。第一组  Operator 起初聚焦于 Kubernetes 中运行的有状态服务，但近年来，Operator 的范围越来越广泛，如今社区为各种广泛用例构建的  Operator 数量正与日俱增。例如，[OperatorHub.io](https://operatorhub.io/) 提供了一个社区  Operator 目录，用于处理各种不同种类的软件和服务。

![image-20191028161706118](.\images\20191028161706118.png)

Operator 的吸引力如此之大，原因有多种。如果您已在使用 Kubernetes 来部署和管理应用程序或更大的解决方案，Operator  可提供一致的资源模型来定义和管理应用程序中的所有不同组件。例如，如果应用程序需要 [etcd  ](https://etcd.io/)数据库，那么只需安装 [etcd Operator  ](https://operatorhub.io/operator/etcd)并创建 EtcdCluster 自定义资源即可。etcd Operator 随后就会负责为应用程序部署和管理 etcd  集群，包括次日操作，如备份和复原。由于 Operator 依赖于自定义资源，即 Kubernetes API 扩展，因此默认情况下，Kubernetes  的所有现有工具都适用。无需学习新工具或新方法。您可以使用同样的 Kubernetes CLI ([ kubectl ](https://kubernetes.io/docs/reference/kubectl/kubectl/))  来创建、更新或删除 Pod 和自定义资源。对于自定义资源来说，基于角色的访问控制 (RBAC) 和准入控制的工作方式是相同的。

此外，由于 Operator 可持续将期望状态与当前状态进行比较并加以协调，因此 Operator  可提供自我修复功能，并确保重新启动服务（如果服务不正常或遭意外删除，则可重新创建）。

# Operator的原理

Operator 实际上作为kubernetes自定义扩展资源注册到controller-manager,通过list and  watch的方式监听对应资源的变化，然后在周期内的各个环节做相应的协调处理。所谓的处理就是operator实现由状态的应用的核心部分，当然不同应用其处理的方式也不同。

##  **控制器模式与声明式 API** 

### **控制器模式**

K8s 作为一个“容器编排”平台，其核心的功能是编排，Pod 作为 K8s 调度的最小单位,具备很多属性和字段，K8s 的编排正是通过一个个控制器根据被控制对象的属性和字段来实现。

下面我们看一个例子：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
spec:
  selector:
    matchLabels:
      app: test
  replicas: 2
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```



K8s 集群在部署时包含了 Controllers 组件，里面对于每个 build-in 的资源类型（比如 Deployments, Statefulset, CronJob, ...）都有对应的 Controller，基本是 1:1 的关系。



上面的例子中，Deployment 资源创建之后，对应的 Deployment Controller 编排动作很简单，确保携带了 app=test 的 Pod 个数永远等于 2，Pod 由 template 部分定义，具体来说，K8s 里面是 kube-controller-manager 这个组件在做这件事，可以看下 K8s 项目的 pkg/controller 目录，里面包含了所有控制器，都以独有的方式负责某种编排功能，但是它们都遵循一个通用编排模式，即：调谐循环（Reconcile loop），其伪代码逻辑为：



```go
for {
actualState := GetResourceActualState(rsvc)
expectState := GetResourceExpectState(rsvc)
if actualState == expectState {
// do nothing
} else {
Reconcile(rsvc)
}
}
```

就是一个无限循环（实际是事件驱动+定时同步来实现，不是无脑循环）不断地对比期望状态和实际状态，如果有出入则进行 Reconcile（调谐）逻辑将实际状态调整为期望状态。

期望状态就是我们的对象定义（通常是 YAML 文件），实际状态是集群里面当前的运行状态（通常来自于 K8s 集群内外相关资源的状态汇总），控制器的编排逻辑主要是第三步做的，这个操作被称为调谐（Reconcile），整个控制器调谐的过程称为“Reconcile Loop”，调谐的最终结果一般是对被控制对象的某种写操作，比如增/删/改 Pod。

在控制器中定义被控制对象是通过“模板”完成的，比如 Deployment 里面的 template 字段里的内容跟一个标准的 Pod 对象的 API 定义一样，所有被这个 Deployment 管理的 Pod 实例，都是根据这个 template 字段的创建的，这就是 PodTemplate，一个控制对象的定义一般是由上半部分的控制定义（期望状态），加上下半部分的被控制对象的模板组成。

### **声明式 API**

所谓声明式就是“告诉 K8s 你要什么，而不是告诉它怎么做的命令”，一个很熟悉的例子就是 SQL，你“告诉 DB 根据条件和各类算子返回数据，而不是告诉它怎么遍历，过滤，聚合”。

在 K8s 里面，声明式的体现就是 kubectl apply 命令，在对象创建和后续更新中一直使用相同的 apply 命令，告诉 K8s 对象的终态即可，底层是通过执行了一个对原有 API 对象的 PATCH 操作来实现的，可以一次性处理多个写操作，具备 Merge 能力 diff 出最终的 PATCH，而命令式一次只能处理一个写请求。

声明式 API 让 K8s 的“容器编排”世界看起来温柔美好，而控制器（以及容器运行时，存储，网络模型等）才是这太平盛世的幕后英雄。说到这里，就会有人希望也能像 build-in 资源一样构建自己的自定义资源（CRD-Customize Resource Definition），然后为自定义资源写一个对应的控制器，推出自己的声明式 API。K8s 提供了 CRD 的扩展方式来满足用户这一需求，而且由于这种扩展方式十分灵活，在最新的1.15 版本对 CRD 做了相当大的增强。对于用户来说，实现 CRD 扩展主要做两件事：

- 编写 CRD 并将其部署到 K8s 集群里；

  这一步的作用就是让 K8s 知道有这个资源及其结构属性，在用户提交该自定义资源的定义时（通常是 YAML 文件定义），K8s 能够成功校验该资源并创建出对应的 Go struct 进行持久化，同时触发控制器的调谐逻辑。

- 编写 Controller 并将其部署到 K8s 集群里。

  这一步的作用就是实现调谐逻辑。

# operator开发

operator-sdk(来自于CoreOS) 和 Kubebuilder (来自于kubernetes)  是目前开发operator两种常用的SDK，或者叫framework,不管是哪种方式只是规范了步骤和一些必要的元素，生成相应的模板而已，实际上如果自己熟悉的话也可以按照自己的方式写，主要满足controller-manager中的controller要求就行。

###  operator-sdk

[OPERATOR-SDK:  https://github.com/operator-framework/operator-sdk](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Foperator-framework%2Foperator-sdk)

### Kubebuilder

[kubebuilder:  https://github.com/kubernetes-sigs/kubebuilder](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fkubernetes-sigs%2Fkubebuilder)



## operator开发示例

```
#1.在golang的工作目录src下创建自己的工程，工程域名和模块名称
[root@node3 src]# mkdir -p cloud.io/etcd-op 
#2.初始化工程依赖，注意开启 export GO111MODULE=on
[root@node3 src]# cd cloud.io/etcd-op/
[root@node3 etcd-op]# export GO111MODULE=on
[root@node3 etcd-op]# go mod init etcd-op
go: creating new go.mod: module etcd-op
#3.初始化operator工程，此处会有包下载不了的问题，见后续QA
[root@node3 etcd-op]# kubebuilder init --domain cloud.io
go mod tidy
此处省略n行...
#在实际创建过程中由于golang很多包无妨下载，需要在go.mod中replace,文件内容贴在文章末尾
[root@node3 etcd-op]# kubebuilder create api --group etcd --version v1 --kind EtcdCluster
Create Resource [y/n]
y
Create Controller [y/n]
y
Writing scaffold for you to edit...
此处省略n行...
go build -o bin/manager main.go
```

- 生成的工程结构

```
[root@node3 src]# tree cloud.io/etcd-op/
cloud.io/etcd-op/
├── api
│   └── v1
│       ├── etcdcluster_types.go  # ①定义EtcdOp的数据结构，对应部署yaml中的属性，同时注册结构体
│       ├── groupversion_info.go # 定义Group和version信息
│       └── zz_generated.deepcopy.go # 封装了①中结构体的相关方法，主要是用于深度拷贝
├── bin
│   └── manager # 编译生成的可执行文件，执行包含三个参数。是否启用选主（-enable-leader-election）；kubecofnig路径（-kubeconfig）运行在集群外是需要指定；组件指标接口地址（-metrics-addr）
├── config
│   ├── certmanager # 整个目录下的内容用于为 TLS(https)双向认证机制签名生成证书
│   │   ├── certificate.yaml
│   │   ├── kustomization.yaml
│   │   └── kustomizeconfig.yaml
│   ├── crd # 用于将该工程注册到k8s使之成为controller-manager中的一个controller，即CRD，但是仅仅是结构
│   │   ├── kustomization.yaml
│   │   ├── kustomizeconfig.yaml
│   │   └── patches
│   │       ├── cainjection_in_etcdclusters.yaml
│   │       └── webhook_in_etcdclusters.yaml
│   ├── default # 是对crd，rbac，manager三个目录的整合，及其本身需要的配置
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   ├── manager_image_patch.yaml
│   │   ├── manager_prometheus_metrics_patch.yaml
│   │   ├── manager_webhook_patch.yaml
│   │   └── webhookcainjection_patch.yaml
│   ├── manager # 该工程实际的部署实例
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── rbac # 默认需要的ClusterRole，ServiceAccount ，绑定关系ClusterRoleBinding，及其具体需要的权限
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   ├── leader_election_role.yaml
│   │   └── role_binding.yaml
│   ├── samples # 实际部署的demo
│   │   └── etcd_v1_etcdcluster.yaml
│   └── webhook # k8s中需要经过的webhooks拦截器
│       ├── kustomization.yaml
│       ├── kustomizeconfig.yaml
│       └── service.yaml
├── controllers
│   ├── etcdcluster_controller.go ## ②具体的执行控制逻辑，结合上面① etcdop_types.go ，在Reconcile中完成执行逻辑，
│   └── suite_test.go  # 测试单元
├── Dockerfile # 为该控制器生成镜像的配置文件
├── go.mod # 依赖golang包
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── main.go # 执行入口
├── Makefile # 通过make编译，其中包含上述的镜像打包，安装部署的一系列流程
└── PROJECT
```

- 编译打镜像

```
[root@node3 etcd-op]# go build -o bin/manager main.go
[root@node3 etcd-op]# docker build -t  controller:latest .
#安装CRD
[root@node3 etcd-op]# kubectl apply -f config/crd/bases
customresourcedefinition.apiextensions.k8s.io/etcdclusters.etcd.cloud.io configured
[root@node3 etcd-op]# kubectl get crd
NAME                         CREATED AT
etcdclusters.etcd.cloud.io   2019-08-14T08:27:58Z
```

## 

