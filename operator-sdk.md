# 背景


自定义资源 CRD（Custom Resource Definition）可以扩展 Kubernetes API，掌握 CRD 是成为 Kubernetes 高级玩家的必备技能。 
# 简介

Kubebuilder 是一个使用 CRDs 构建 K8s API 的 SDK，主要是：

- 提供脚手架工具初始化 CRDs 工程，自动生成 boilerplate 代码和配置；
- 提供代码库封装底层的 K8s go-client；

 方便用户从零开始开发 CRDs，Controllers 和 Admission Webhooks 来扩展 K8s。 

# **核心概念**

- **GVKs&GVRs**

GVK = GroupVersionKind，GVR = GroupVersionResource。

- API Group & Versions（GV）：API Group 是相关 API 功能的集合，每个 Group 拥有一或多个 Versions，用于接口的演进。

- Kinds & Resources：每个 GV 都包含多个 API 类型，称为 Kinds，在不同的 Versions 之间同一个 Kind 定义可能不同， Resource 是 Kind 的对象标识（resource type），一般来说 Kinds 和 Resources 是 1:1 的，比如 pods Resource 对应 Pod Kind，但是有时候相同的 Kind 可能对应多个 Resources，比如 Scale Kind 可能对应很多 Resources：deployments/scale，replicasets/scale，对于 CRD 来说，只会是 1:1 的关系。

每一个 GVK 都关联着一个 package 中给定的 root Go type，比如 apps/v1/Deployment 就关联着 K8s 源码里面 k8s.io/api/apps/v1 package 中的 Deployment struct，我们提交的各类资源定义 YAML 文件都需要写：

- apiVersion：这个就是 GV 。
- kind：这个就是 K。

根据 GVK K8s 就能找到你到底要创建什么类型的资源，根据你定义的 Spec 创建好资源之后就成为了 Resource，也就是 GVR。GVK/GVR 就是 K8s 资源的坐标，是我们创建/删除/修改/读取资源的基础。

- **Scheme**

每一组 Controllers 都需要一个 Scheme，提供了 Kinds 与对应 Go types 的映射，也就是说给定 Go type 就知道他的 GVK，给定 GVK 就知道他的 Go type，比如说我们给定一个 Scheme: "tutotial.kubebuilder.io/api/v1".CronJob{} 这个 Go type 映射到 batch.tutotial.kubebuilder.io/v1 的 CronJob GVK，那么从 Api Server 获取到下面的 JSON:

```json
{
    "kind": "CronJob",
    "apiVersion": "batch.tutorial.kubebuilder.io/v1",
    ...
}
```

就能构造出对应的 Go type了，通过这个 Go type 也能正确地获取 GVR 的一些信息，控制器可以通过该 Go type 获取到期望状态以及其他辅助信息进行调谐逻辑。

- **Manager**

Kubebuilder 的核心组件，具有 3 个职责：

- 负责运行所有的 Controllers；
- 初始化共享 caches，包含 listAndWatch 功能；
- 初始化 clients 用于与 Api Server 通信。

- **Cache**

Kubebuilder 的核心组件，负责在 Controller 进程里面根据 Scheme 同步 Api Server 中所有该 Controller 关心 GVKs 的 GVRs，其核心是 GVK -> Informer 的映射，Informer 会负责监听对应 GVK 的 GVRs 的创建/删除/更新操作，以触发 Controller 的 Reconcile 逻辑。

- **Controller**

Kubebuidler 为我们生成的脚手架文件，我们只需要实现 Reconcile 方法即可。

- **Clients**

在实现 Controller 的时候不可避免地需要对某些资源类型进行创建/删除/更新，就是通过该 Clients 实现的，其中查询功能实际查询是本地的 Cache，写操作直接访问 Api Server。

- **Index**

由于 Controller 经常要对 Cache 进行查询，Kubebuilder 提供 Index utility 给 Cache 加索引提升查询效率。

- **Finalizer**

在一般情况下，如果资源被删除之后，我们虽然能够被触发删除事件，但是这个时候从 Cache 里面无法读取任何被删除对象的信息，这样一来，导致很多垃圾清理工作因为信息不足无法进行，K8s 的 Finalizer 字段用于处理这种情况。

在 K8s 中，只要对象 ObjectMeta 里面的 Finalizers 不为空，对该对象的 delete 操作就会转变为 update 操作，具体说就是 update  deletionTimestamp 字段，其意义就是告诉 K8s 的 GC“在deletionTimestamp 这个时刻之后，只要 Finalizers 为空，就立马删除掉该对象”。

所以一般的使用姿势就是在创建对象时把 Finalizers 设置好（任意 string），然后处理 DeletionTimestamp 不为空的 update 操作（实际是 delete），根据 Finalizers 的值执行完所有的 pre-delete hook（此时可以在 Cache 里面读取到被删除对象的任何信息）之后将 Finalizers 置为空即可。

- **OwnerReference**

K8s GC 在删除一个对象时，任何 ownerReference 是该对象的对象都会被清除，与此同时，Kubebuidler 支持所有对象的变更都会触发 Owner 对象 controller 的 Reconcile 方法。

所有概念集合在一起如图 1 所示：

![image-20191028170622922](.\images\20191028170100.jpg)

# 安装部署

## 前提条件

安装Golang（version >= 1.13）

配置GOPATH

开启go module （`export GO111MODULE=on`） 

## 安装operator-sdk

```bash
$ wget https://github.com/operator-framework/operator-sdk/releases/download/v0.12.0/operator-sdk-v0.12.0-x86_64-linux-gnu
$ mv operator-sdk-v0.12.0-x86_64-linux-gnu /usr/local/kubebuilder/bin/operator-sdk
$ export PATH=$PATH:/usr/local/kubebuilder/bin
$ operator-sdk version
operator-sdk version: "v0.12.0", commit: "2445fcda834ca4b7cf0d6c38fba6317fb219b469", go version: "go1.13.3 linux/amd64"
```

## 安装dep

kubebuilder使用到Golang包管理工具dep，因此需要安装dep


```bash
$ curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh
$ dep version
dep:
 version     : v0.5.4
 build date  : 2019-07-01
 git hash    : 1f7c19e
 go version  : go1.12.6
 go compiler : gc
 platform    : linux/amd64
 features    : ImportDuringSolve=false

```

# Operator开发

## Workflow

Operator SDK 提供以下工作流来开发一个新的 Operator：

- 1. 使用 SDK 创建一个新的 Operator 项目
- 2. 通过添加自定义资源（CRD）定义新的资源 API
- 3. 指定使用 SDK API 来 watch 的资源
- 4. 定义 Operator 的协调（reconcile）逻辑
- 5. 使用 Operator SDK 构建并生成 Operator 部署清单文件

## Demo

我们平时在部署一个简单的 Webserver 到 Kubernetes 集群中的时候，都需要先编写一个 Deployment 的控制器，然后创建一个 Service 对象，通过 Pod 的 label 标签进行关联，最后通过 Ingress 或者 type=NodePort 类型的 Service 来暴露服务，每次都需要这样操作，是不是略显麻烦，我们就可以创建一个自定义的资源对象，通过我们的 CRD 来描述我们要部署的应用信息，比如镜像、服务端口、环境变量等等，然后创建我们的自定义类型的资源对象的时候，通过控制器去创建对应的 Deployment 和 Service，是不是就方便很多了，相当于我们用一个资源清单去描述了 Deployment 和 Service 要做的两件事情。

这里我们将创建一个名为 AppService 的 CRD 资源对象，然后定义如下的资源清单进行应用部署：

```yaml
apiVersion: app.example.com/v1
kind: AppService
metadata:
  name: nginx-app
spec:
  size: 2
  image: nginx:1.7.9
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30002
```

通过这里的自定义的 AppService 资源对象去创建副本数为2的 Pod，然后通过 nodePort=30002 的端口去暴露服务，接下来我们就来一步一步的实现我们这里的这个简单的 Operator 应用。

## 脚手架工程搭建

### 1、初始化工程

初始化operator工程， 创建脚手架工程 

此处会有包下载不了的问题，需科学上网

```bash
[root@izttsn6fen661yz opdemo]# pwd
/root/workspace/src/github.com/icbc
[root@izttsn6fen661yz icbc]# operator-sdk new opdemo
INFO[0000] Creating new Go operator 'opdemo'.           
此处省略n行...
[root@izttsn6fen661yz icbc]# cd opdemo/
[root@izttsn6fen661yz opdemo]# tree
.
├── build
│   ├── bin
│   │   ├── entrypoint
│   │   └── user_setup
│   └── Dockerfile
├── cmd
│   └── manager
│       └── main.go
├── deploy
│   ├── operator.yaml
│   ├── role_binding.yaml
│   ├── role.yaml
│   └── service_account.yaml
├── go.mod
├── go.sum
├── pkg
│   ├── apis
│   │   └── apis.go
│   └── controller
│       └── controller.go
├── tools.go
└── version
    └── version.go

9 directories, 14 files

```

### 2、创建 API** 

 为自定义资源添加一个新的 API，按照上面我们预定义的资源清单文件，在 Operator 相关根目录下面执行如下命令： 

```bash
[root@izttsn6fen661yz opdemo]# operator-sdk add api --api-version=app.example.com/v1 --kind=AppService
INFO[0000] Generating api version app.example.com/v1 for kind AppService. 
INFO[0000] Created pkg/apis/app/group.go                
INFO[0063] Created pkg/apis/app/v1/appservice_types.go  
INFO[0063] Created pkg/apis/addtoscheme_app_v1.go       
INFO[0063] Created pkg/apis/app/v1/register.go          
INFO[0063] Created pkg/apis/app/v1/doc.go               
INFO[0063] Created deploy/crds/app.example.com_v1_appservice_cr.yaml 
INFO[0066] Created deploy/crds/app.example.com_appservices_crd.yaml 
INFO[0066] Running deepcopy code-generation for Custom Resource group versions: [app:[v1], ] 
INFO[0077] Code-generation complete.                    
INFO[0077] Running OpenAPI code-generation for Custom Resource group versions: [app:[v1], ] 
INFO[0089] Created deploy/crds/app.example.com_appservices_crd.yaml 
INFO[0089] Code-generation complete.                    
INFO[0089] API generation complete.                     
[root@izttsn6fen661yz opdemo]# tree
.
├── build
│   ├── bin
│   │   ├── entrypoint
│   │   └── user_setup
│   └── Dockerfile
├── cmd
│   └── manager
│       └── main.go
├── deploy
│   ├── crds
│   │   ├── app.example.com_appservices_crd.yaml
│   │   └── app.example.com_v1_appservice_cr.yaml
│   ├── operator.yaml
│   ├── role_binding.yaml
│   ├── role.yaml
│   └── service_account.yaml
├── go.mod
├── go.sum
├── pkg
│   ├── apis
│   │   ├── addtoscheme_app_v1.go
│   │   ├── apis.go
│   │   └── app
│   │       ├── group.go
│   │       └── v1
│   │           ├── appservice_types.go
│   │           ├── doc.go
│   │           ├── register.go
│   │           ├── zz_generated.deepcopy.go
│   │           └── zz_generated.openapi.go
│   └── controller
│       └── controller.go
├── tools.go
└── version
    └── version.go

12 directories, 23 files
[root@izttsn6fen661yz opdemo]# 
```

### 3、创建 Controller 

 添加对应的自定义 API 的具体实现 Controller： 

```bash
[root@izttsn6fen661yz opdemo]# operator-sdk add controller --api-version=app.example.com/v1 --kind=AppService
INFO[0000] Generating controller version app.example.com/v1 for kind AppService. 
INFO[0000] Created pkg/controller/appservice/appservice_controller.go 
INFO[0000] Created pkg/controller/add_appservice.go     
INFO[0000] Controller generation complete.              
[root@izttsn6fen661yz opdemo]# tree
.
├── build
│   ├── bin
│   │   ├── entrypoint
│   │   └── user_setup
│   └── Dockerfile
├── cmd
│   └── manager
│       └── main.go
├── deploy
│   ├── crds
│   │   ├── app.example.com_appservices_crd.yaml
│   │   └── app.example.com_v1_appservice_cr.yaml
│   ├── operator.yaml
│   ├── role_binding.yaml
│   ├── role.yaml
│   └── service_account.yaml
├── go.mod
├── go.sum
├── pkg
│   ├── apis
│   │   ├── addtoscheme_app_v1.go
│   │   ├── apis.go
│   │   └── app
│   │       ├── group.go
│   │       └── v1
│   │           ├── appservice_types.go
│   │           ├── doc.go
│   │           ├── register.go
│   │           ├── zz_generated.deepcopy.go
│   │           └── zz_generated.openapi.go
│   └── controller
│       ├── add_appservice.go
│       ├── appservice
│       │   └── appservice_controller.go
│       └── controller.go
├── tools.go
└── version
    └── version.go

13 directories, 25 files
```

 这样整个 Operator 项目的脚手架就已经搭建完成了，接下来就是具体的实现了。 

## 编码实现

### 自定义 API

打开源文件`pkg/apis/app/v1/appservice_types.go`，需要我们根据我们的需求去自定义结构体 AppServiceSpec，我们最上面预定义的资源清单中就有 size、image、ports 这些属性，所有我们需要用到的属性都需要在这个结构体中进行定义：

```golang
type AppServiceSpec struct {
    // INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
    // Important: Run "operator-sdk generate k8s" to regenerate code after modifying this file
    // Add custom validation using kubebuilder tags: https://book.kubebuilder.io/beyond_basics/generating_crd.html
    Size      *int32                      `json:"size"`
    Image     string                      `json:"image"`
    Resources corev1.ResourceRequirements `json:"resources,omitempty"`
    Envs      []corev1.EnvVar             `json:"envs,omitempty"`
    Ports     []corev1.ServicePort        `json:"ports,omitempty"`
}
```

代码中会涉及到一些包名的导入，由于包名较多，所以我们会使用一些别名进行区分，主要的包含下面几个：

```golang
import (
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    appv1 "github.com/cnych/opdemo/pkg/apis/app/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)
```

这里的 resources、envs、ports 的定义都是直接引用的`"k8s.io/api/core/v1"`中定义的结构体，而且需要注意的是我们这里使用的是`ServicePort`，而不是像传统的 Pod 中定义的 ContanerPort，这是因为我们的资源清单中不仅要描述容器的 Port，还要描述 Service 的 Port。

然后一个比较重要的结构体`AppServiceStatus`用来描述资源的状态，当然我们可以根据需要去自定义状态的描述，我这里就偷懒直接使用 Deployment 的状态了：

```shell
type AppServiceStatus struct {
    // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
    // Important: Run "operator-sdk generate k8s" to regenerate code after modifying this file
    // Add custom validation using kubebuilder tags: https://book.kubebuilder.io/beyond_basics/generating_crd.html
    appsv1.DeploymentStatus `json:",inline"`
}
```

定义完成后，在项目根目录下面执行如下命令：

```shell
$ operator-sdk generate k8s
```

该命令是用来根据我们自定义的 API 描述来自动生成一些代码，目录`pkg/apis/app/v1/`下面以`zz_generated`开头的文件就是自动生成的代码，里面的内容并不需要我们去手动编写。

这样我们就算完成了对自定义资源对象的 API 的声明。

#### 实现业务逻辑

上面 API 描述声明完成了，接下来就需要我们来进行具体的业务逻辑实现了，编写具体的 controller 实现，打开源文件`pkg/controller/appservice/appservice_controller.go`，需要我们去更改的地方也不是很多，核心的就是`Reconcile`方法，该方法就是去不断的 watch 资源的状态，然后根据状态的不同去实现各种操作逻辑，核心代码如下：

```golang
func (r *ReconcileAppService) Reconcile(request reconcile.Request) (reconcile.Result, error) {
    reqLogger := log.WithValues("Request.Namespace", request.Namespace, "Request.Name", request.Name)
    reqLogger.Info("Reconciling AppService")

    // Fetch the AppService instance
    instance := &appv1.AppService{}
    err := r.client.Get(context.TODO(), request.NamespacedName, instance)
    if err != nil {
        if errors.IsNotFound(err) {
            // Request object not found, could have been deleted after reconcile request.
            // Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
            // Return and don't requeue
            return reconcile.Result{}, nil
        }
        // Error reading the object - requeue the request.
        return reconcile.Result{}, err
    }

    if instance.DeletionTimestamp != nil {
        return reconcile.Result{}, err
    }

    // 如果不存在，则创建关联资源
    // 如果存在，判断是否需要更新
    //   如果需要更新，则直接更新
    //   如果不需要更新，则正常返回

    deploy := &appsv1.Deployment{}
    if err := r.client.Get(context.TODO(), request.NamespacedName, deploy); err != nil && errors.IsNotFound(err) {
        // 创建关联资源
        // 1. 创建 Deploy
        deploy := resources.NewDeploy(instance)
        if err := r.client.Create(context.TODO(), deploy); err != nil {
            return reconcile.Result{}, err
        }
        // 2. 创建 Service
        service := resources.NewService(instance)
        if err := r.client.Create(context.TODO(), service); err != nil {
            return reconcile.Result{}, err
        }
        // 3. 关联 Annotations
        data, _ := json.Marshal(instance.Spec)
        if instance.Annotations != nil {
            instance.Annotations["spec"] = string(data)
        } else {
            instance.Annotations = map[string]string{"spec": string(data)}
        }

        if err := r.client.Update(context.TODO(), instance); err != nil {
            return reconcile.Result{}, nil
        }
        return reconcile.Result{}, nil
    }

    oldspec := appv1.AppServiceSpec{}
    if err := json.Unmarshal([]byte(instance.Annotations["spec"]), oldspec); err != nil {
        return reconcile.Result{}, err
    }

    if !reflect.DeepEqual(instance.Spec, oldspec) {
        // 更新关联资源
        newDeploy := resources.NewDeploy(instance)
        oldDeploy := &appsv1.Deployment{}
        if err := r.client.Get(context.TODO(), request.NamespacedName, oldDeploy); err != nil {
            return reconcile.Result{}, err
        }
        oldDeploy.Spec = newDeploy.Spec
        if err := r.client.Update(context.TODO(), oldDeploy); err != nil {
            return reconcile.Result{}, err
        }

        newService := resources.NewService(instance)
        oldService := &corev1.Service{}
        if err := r.client.Get(context.TODO(), request.NamespacedName, oldService); err != nil {
            return reconcile.Result{}, err
        }
        oldService.Spec = newService.Spec
        if err := r.client.Update(context.TODO(), oldService); err != nil {
            return reconcile.Result{}, err
        }

        return reconcile.Result{}, nil
    }

    return reconcile.Result{}, nil

}
```

上面就是业务逻辑实现的核心代码，逻辑很简单，就是去判断资源是否存在，不存在，则直接创建新的资源，创建新的资源除了需要创建 Deployment 资源外，还需要创建 Service 资源对象，因为这就是我们的需求，当然你还可以自己去扩展，比如在创建一个 Ingress 对象。更新也是一样的，去对比新旧对象的声明是否一致，不一致则需要更新，同样的，两种资源都需要更新的。

另外两个核心的方法就是上面的`resources.NewDeploy(instance)`和`resources.NewService(instance)`方法，这两个方法实现逻辑也很简单，就是根据 CRD 中的声明去填充 Deployment 和 Service 资源对象的 Spec 对象即可。

NewDeploy 方法实现如下：

```golang
func NewDeploy(app *appv1.AppService) *appsv1.Deployment {
    labels := map[string]string{"app": app.Name}
    selector := &metav1.LabelSelector{MatchLabels: labels}
    return &appsv1.Deployment{
        TypeMeta: metav1.TypeMeta{
            APIVersion: "apps/v1",
            Kind:       "Deployment",
        },
        ObjectMeta: metav1.ObjectMeta{
            Name:      app.Name,
            Namespace: app.Namespace,

            OwnerReferences: []metav1.OwnerReference{
                *metav1.NewControllerRef(app, schema.GroupVersionKind{
                    Group: v1.SchemeGroupVersion.Group,
                    Version: v1.SchemeGroupVersion.Version,
                    Kind: "AppService",
                }),
            },
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: app.Spec.Size,
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: labels,
                },
                Spec: corev1.PodSpec{
                    Containers: newContainers(app),
                },
            },
            Selector: selector,
        },
    }
}

func newContainers(app *v1.AppService) []corev1.Container {
    containerPorts := []corev1.ContainerPort{}
    for _, svcPort := range app.Spec.Ports {
        cport := corev1.ContainerPort{}
        cport.ContainerPort = svcPort.TargetPort.IntVal
        containerPorts = append(containerPorts, cport)
    }
    return []corev1.Container{
        {
            Name: app.Name,
            Image: app.Spec.Image,
            Resources: app.Spec.Resources,
            Ports: containerPorts,
            ImagePullPolicy: corev1.PullIfNotPresent,
            Env: app.Spec.Envs,
        },
    }
}
```

newService 对应的方法实现如下：

```golang
func NewService(app *v1.AppService) *corev1.Service {
    return &corev1.Service {
        TypeMeta: metav1.TypeMeta {
            Kind: "Service",
            APIVersion: "v1",
        },
        ObjectMeta: metav1.ObjectMeta{
            Name: app.Name,
            Namespace: app.Namespace,
            OwnerReferences: []metav1.OwnerReference{
                *metav1.NewControllerRef(app, schema.GroupVersionKind{
                    Group: v1.SchemeGroupVersion.Group,
                    Version: v1.SchemeGroupVersion.Version,
                    Kind: "AppService",
                }),
            },
        },
        Spec: corev1.ServiceSpec{
            Type: corev1.ServiceTypeNodePort,
            Ports: app.Spec.Ports,
            Selector: map[string]string{
                "app": app.Name,
            },
        },
    }
}
```

这样我们就实现了 AppService 这种资源对象的业务逻辑。

#### 调试

如果我们本地有一个可以访问的 Kubernetes 集群，我们也可以直接进行调试，在本地用户`~/.kube/config`文件中配置集群访问信息，下面的信息表明可以访问 Kubernetes 集群：

```shell
$ kubectl cluster-info
Kubernetes master is running at https://ydzs-master:6443
KubeDNS is running at https://ydzs-master:6443/api/v1/namespaces/kube-system/services/kube-dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

首先，在集群中安装 CRD 对象：

```shell
$ kubectl create -f deploy/crds/app_v1_appservice_crd.yaml
customresourcedefinition "appservices.app.example.com" created
$ kubectl get crd
NAME                                   AGE
appservices.app.example.com            <invalid>
......
```

当我们通过`kubectl get crd`命令获取到我们定义的 CRD 资源对象，就证明我们定义的 CRD 安装成功了。其实现在只是 CRD 的这个声明安装成功了，但是我们这个 CRD 的具体业务逻辑实现方式还在我们本地，并没有部署到集群之中，我们可以通过下面的命令来在本地项目中启动 Operator 的调试：

```shell
$ operator-sdk up local                                                     
INFO[0000] Running the operator locally.                
INFO[0000] Using namespace default.                     
{"level":"info","ts":1559207203.964137,"logger":"cmd","msg":"Go Version: go1.11.4"}
{"level":"info","ts":1559207203.964192,"logger":"cmd","msg":"Go OS/Arch: darwin/amd64"}
{"level":"info","ts":1559207203.9641972,"logger":"cmd","msg":"Version of operator-sdk: v0.7.0"}
{"level":"info","ts":1559207203.965905,"logger":"leader","msg":"Trying to become the leader."}
{"level":"info","ts":1559207203.965945,"logger":"leader","msg":"Skipping leader election; not running in a cluster."}
{"level":"info","ts":1559207206.928867,"logger":"cmd","msg":"Registering Components."}
{"level":"info","ts":1559207206.929077,"logger":"kubebuilder.controller","msg":"Starting EventSource","controller":"appservice-controller","source":"kind source: /, Kind="}
{"level":"info","ts":1559207206.9292521,"logger":"kubebuilder.controller","msg":"Starting EventSource","controller":"appservice-controller","source":"kind source: /, Kind="}
{"level":"info","ts":1559207209.622659,"logger":"cmd","msg":"failed to initialize service object for metrics: OPERATOR_NAME must be set"}
{"level":"info","ts":1559207209.622693,"logger":"cmd","msg":"Starting the Cmd."}
{"level":"info","ts":1559207209.7236018,"logger":"kubebuilder.controller","msg":"Starting Controller","controller":"appservice-controller"}
{"level":"info","ts":1559207209.8284118,"logger":"kubebuilder.controller","msg":"Starting workers","controller":"appservice-controller","worker count":1}
```

上面的命令会在本地运行 Operator 应用，通过`~/.kube/config`去关联集群信息，现在我们去添加一个 AppService 类型的资源然后观察本地 Operator 的变化情况，资源清单文件就是我们上面预定义的（deploy/crds/app_v1_appservice_cr.yaml）

```yaml
apiVersion: app.example.com/v1
kind: AppService
metadata:
  name: nginx-app
spec:
  size: 2
  image: nginx:1.7.9
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30002
```

直接创建这个资源对象：

```shell
$ kubectl create -f deploy/crds/app_v1_appservice_cr.yaml
appservice "nginx-app" created
```

我们可以看到我们的应用创建成功了，这个时候查看 Operator 的调试窗口会有如下的信息出现：

```shell
......
{"level":"info","ts":1559207416.670523,"logger":"controller_appservice","msg":"Reconciling AppService","Request.Namespace":"default","Request.Name":"nginx-app"}
{"level":"info","ts":1559207417.004226,"logger":"controller_appservice","msg":"Reconciling AppService","Request.Namespace":"default","Request.Name":"nginx-app"}
{"level":"info","ts":1559207417.004331,"logger":"controller_appservice","msg":"Reconciling AppService","Request.Namespace":"default","Request.Name":"nginx-app"}
{"level":"info","ts":1559207418.33779,"logger":"controller_appservice","msg":"Reconciling AppService","Request.Namespace":"default","Request.Name":"nginx-app"}
{"level":"info","ts":1559207418.951193,"logger":"controller_appservice","msg":"Reconciling AppService","Request.Namespace":"default","Request.Name":"nginx-app"}
......
```

然后我们可以去查看集群中是否有符合我们预期的资源出现：

```shell
$ kubectl get AppService
NAME        AGE
nginx-app   <invalid>
$ kubectl get deploy
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-app                2         2         2            2           <invalid>
$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        76d
nginx-app    NodePort    10.108.227.5   <none>        80:30002/TCP   <invalid>
$ kubectl get pods
NAME                                      READY     STATUS    RESTARTS   AGE
nginx-app-76b6449498-2j82j                1/1       Running   0          <invalid>
nginx-app-76b6449498-m4h58                1/1       Running   0          <invalid>
```

看到了吧，我们定义了两个副本（size=2），这里就出现了两个 Pod，还有一个 NodePort=30002 的 Service 对象，我们可以通过该端口去访问下应用：

```

```

## 编译打镜像

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

#  **源码阅读** 

## main入口

 main.go 是整个项目的入口 

```go
func init() {
	_ = clientgoscheme.AddToScheme(scheme)
	_ = appsv1.AddToScheme(scheme)
	// +kubebuilder:scaffold:scheme
}

func main() {
  ...
        // 1、init Manager
	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		Scheme:             scheme,
		MetricsBindAddress: metricsAddr,
		LeaderElection:     enableLeaderElection,
		Port:               9443,
	})
  ...
        // 2、init Reconciler（Controller）
	if err = (&controllers.ApplicationReconciler{
		Client: mgr.GetClient(),
		Log:    ctrl.Log.WithName("controllers").WithName("Application"),
	}).SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create controller", "controller", "Application")
		os.Exit(1)
	}
	// +kubebuilder:scaffold:builder
    
        // 3、start Manager
	setupLog.Info("starting manager")
	if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
		setupLog.Error(err, "problem running manager")
		os.Exit(1)
	}
}
```

可以看到在 init 方法里面我们将 appsv1 注册到 Scheme 里面去了，这样一来 Cache 就知道 watch 谁了，main 方法里面的逻辑基本都是 Manager的：

1. 初始化了一个 Manager；
2. 将 Manager 的 Client 传给 Controller，并且调用 SetupWithManager 方法传入 Manager 进行 Controller 的初始化；
3. 启动 Manager。

我们的核心就是看这 3 个流程。

## **Manager 初始化**

Manager 初始化代码如下：

```go
// New returns a new Manager for creating Controllers.
func New(config *rest.Config, options Options) (Manager, error) {
  ...
  // Create the cache for the cached read client and registering informers
  cache, err := options.NewCache(config, cache.Options{Scheme: options.Scheme, Mapper: mapper, Resync: options.SyncPeriod, Namespace: options.Namespace})
  if err != nil {
    return nil, err
  }
  apiReader, err := client.New(config, client.Options{Scheme: options.Scheme, Mapper: mapper})
  if err != nil {
    return nil, err
  }
  writeObj, err := options.NewClient(cache, config, client.Options{Scheme: options.Scheme, Mapper: mapper})
  if err != nil {
    return nil, err
  }
  ...
  return &controllerManager{
    config:           config,
    scheme:           options.Scheme,
    errChan:          make(chan error),
    cache:            cache,
    fieldIndexes:     cache,
    client:           writeObj,
    apiReader:        apiReader,
    recorderProvider: recorderProvider,
    resourceLock:     resourceLock,
    mapper:           mapper,
    metricsListener:  metricsListener,
    internalStop:     stop,
    internalStopper:  stop,
    port:             options.Port,
    host:             options.Host,
    leaseDuration:    *options.LeaseDuration,
    renewDeadline:    *options.RenewDeadline,
    retryPeriod:      *options.RetryPeriod,
  }, nil
}
```

可以看到主要是创建 Cache 与 Clients：

- **创建 Cache**

Cache 初始化代码如下：

```go
// New initializes and returns a new Cache.
func New(config *rest.Config, opts Options) (Cache, error) {
  opts, err := defaultOpts(config, opts)
  if err != nil {
    return nil, err
  }
  im := internal.NewInformersMap(config, opts.Scheme, opts.Mapper, *opts.Resync, opts.Namespace)
  return &informerCache{InformersMap: im}, nil
}
// newSpecificInformersMap returns a new specificInformersMap (like
// the generical InformersMap, except that it doesn't implement WaitForCacheSync).
func newSpecificInformersMap(...) *specificInformersMap {
  ip := &specificInformersMap{
    Scheme:            scheme,
    mapper:            mapper,
    informersByGVK:    make(map[schema.GroupVersionKind]*MapEntry),
    codecs:            serializer.NewCodecFactory(scheme),
    resync:            resync,
    createListWatcher: createListWatcher,
    namespace:         namespace,
  }
  return ip
}
// MapEntry contains the cached data for an Informer
type MapEntry struct {
  // Informer is the cached informer
  Informer cache.SharedIndexInformer
  // CacheReader wraps Informer and implements the CacheReader interface for a single type
  Reader CacheReader
}
func createUnstructuredListWatch(gvk schema.GroupVersionKind, ip *specificInformersMap) (*cache.ListWatch, error) {
        ...
  // Create a new ListWatch for the obj
  return &cache.ListWatch{
    ListFunc: func(opts metav1.ListOptions) (runtime.Object, error) {
      if ip.namespace != "" && mapping.Scope.Name() != meta.RESTScopeNameRoot {
        return dynamicClient.Resource(mapping.Resource).Namespace(ip.namespace).List(opts)
      }
      return dynamicClient.Resource(mapping.Resource).List(opts)
    },
    // Setup the watch function
    WatchFunc: func(opts metav1.ListOptions) (watch.Interface, error) {
      // Watch needs to be set to true separately
      opts.Watch = true
      if ip.namespace != "" && mapping.Scope.Name() != meta.RESTScopeNameRoot {
        return dynamicClient.Resource(mapping.Resource).Namespace(ip.namespace).Watch(opts)
      }
      return dynamicClient.Resource(mapping.Resource).Watch(opts)
    },
  }, nil
}
```

可以看到 Cache 主要就是创建了 InformersMap，Scheme 里面的每个 GVK 都创建了对应的 Informer，通过 informersByGVK 这个 map 做 GVK 到 Informer 的映射，每个 Informer 会根据 ListWatch 函数对对应的 GVK 进行 List 和 Watch。

- **创建 Clients**

创建 Clients 很简单：

```go
// defaultNewClient creates the default caching client
func defaultNewClient(cache cache.Cache, config *rest.Config, options client.Options) (client.Client, error) {
  // Create the Client for Write operations.
  c, err := client.New(config, options)
  if err != nil {
    return nil, err
  }
  return &client.DelegatingClient{
    Reader: &client.DelegatingReader{
      CacheReader:  cache,
      ClientReader: c,
    },
    Writer:       c,
    StatusClient: c,
  }, nil
}
```

读操作使用上面创建的 Cache，写操作使用 K8s go-client 直连。

## **Controller 初始化**

下面看看 Controller 的启动：

```go
func (r *EDASApplicationReconciler) SetupWithManager(mgr ctrl.Manager) error {
  err := ctrl.NewControllerManagedBy(mgr).
    For(&appsv1alpha1.EDASApplication{}).
    Complete(r)
return err
}
```

使用的是 Builder 模式，NewControllerManagerBy 和 For 方法都是给 Builder 传参，最重要的是最后一个方法 Complete，其逻辑是：

```
func (blder *Builder) Build(r reconcile.Reconciler) (manager.Manager, error) {
...
  // Set the Manager
  if err := blder.doManager(); err != nil {
    return nil, err
  }
  // Set the ControllerManagedBy
  if err := blder.doController(r); err != nil {
    return nil, err
  }
  // Set the Watch
  if err := blder.doWatch(); err != nil {
    return nil, err
  }
...
  return blder.mgr, nil
}
```

主要是看看 doController 和 doWatch 方法：

**doController 方法**

```go
func New(name string, mgr manager.Manager, options Options) (Controller, error) {
  if options.Reconciler == nil {
    return nil, fmt.Errorf("must specify Reconciler")
  }
  if len(name) == 0 {
    return nil, fmt.Errorf("must specify Name for Controller")
  }
  if options.MaxConcurrentReconciles <= 0 {
    options.MaxConcurrentReconciles = 1
  }
  // Inject dependencies into Reconciler
  if err := mgr.SetFields(options.Reconciler); err != nil {
    return nil, err
  }
  // Create controller with dependencies set
  c := &controller.Controller{
    Do:                      options.Reconciler,
    Cache:                   mgr.GetCache(),
    Config:                  mgr.GetConfig(),
    Scheme:                  mgr.GetScheme(),
    Client:                  mgr.GetClient(),
    Recorder:                mgr.GetEventRecorderFor(name),
    Queue:                   workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), name),
    MaxConcurrentReconciles: options.MaxConcurrentReconciles,
    Name:                    name,
  }
  // Add the controller as a Manager components
  return c, mgr.Add(c)
}
```

该方法初始化了一个 Controller，传入了一些很重要的参数：

- Do：Reconcile 逻辑；
- Cache：找 Informer 注册 Watch；
- Client：对 K8s 资源进行 CRUD；
- Queue：Watch 资源的 CUD 事件缓存；
- Recorder：事件收集。

**doWatch 方法**

```go
func (blder *Builder) doWatch() error {
  // Reconcile type
  src := &source.Kind{Type: blder.apiType}
  hdler := &handler.EnqueueRequestForObject{}
  err := blder.ctrl.Watch(src, hdler, blder.predicates...)
  if err != nil {
    return err
  }
  // Watches the managed types
  for _, obj := range blder.managedObjects {
    src := &source.Kind{Type: obj}
    hdler := &handler.EnqueueRequestForOwner{
      OwnerType:    blder.apiType,
      IsController: true,
    }
    if err := blder.ctrl.Watch(src, hdler, blder.predicates...); err != nil {
      return err
    }
  }
  // Do the watch requests
  for _, w := range blder.watchRequest {
    if err := blder.ctrl.Watch(w.src, w.eventhandler, blder.predicates...); err != nil {
      return err
    }
  }
  return nil
}
```

可以看到该方法对本 Controller 负责的 CRD 进行了 watch，同时底下还会 watch 本 CRD 管理的其他资源，这个 managedObjects 可以通过 Controller 初始化 Buidler 的 Owns 方法传入，说到 Watch 我们关心两个逻辑：

- 注册的 handler

```go
type EnqueueRequestForObject struct{}
// Create implements EventHandler
func (e *EnqueueRequestForObject) Create(evt event.CreateEvent, q workqueue.RateLimitingInterface) {
        ...
  q.Add(reconcile.Request{NamespacedName: types.NamespacedName{
    Name:      evt.Meta.GetName(),
    Namespace: evt.Meta.GetNamespace(),
  }})
}
// Update implements EventHandler
func (e *EnqueueRequestForObject) Update(evt event.UpdateEvent, q workqueue.RateLimitingInterface) {
  if evt.MetaOld != nil {
    q.Add(reconcile.Request{NamespacedName: types.NamespacedName{
      Name:      evt.MetaOld.GetName(),
      Namespace: evt.MetaOld.GetNamespace(),
    }})
  } else {
    enqueueLog.Error(nil, "UpdateEvent received with no old metadata", "event", evt)
  }
  if evt.MetaNew != nil {
    q.Add(reconcile.Request{NamespacedName: types.NamespacedName{
      Name:      evt.MetaNew.GetName(),
      Namespace: evt.MetaNew.GetNamespace(),
    }})
  } else {
    enqueueLog.Error(nil, "UpdateEvent received with no new metadata", "event", evt)
  }
}
// Delete implements EventHandler
func (e *EnqueueRequestForObject) Delete(evt event.DeleteEvent, q workqueue.RateLimitingInterface) {
        ...
  q.Add(reconcile.Request{NamespacedName: types.NamespacedName{
    Name:      evt.Meta.GetName(),
    Namespace: evt.Meta.GetNamespace(),
  }})
}
```



可以看到 Kubebuidler 为我们注册的 Handler 就是将发生变更的对象的 NamespacedName 入队列，如果在 Reconcile 逻辑中需要判断创建/更新/删除，需要有自己的判断逻辑。

- 注册的流程

```
// Watch implements controller.Controller
func (c *Controller) Watch(src source.Source, evthdler handler.EventHandler, prct ...predicate.Predicate) error {
  ...
  log.Info("Starting EventSource", "controller", c.Name, "source", src)
  return src.Start(evthdler, c.Queue, prct...)
}
// Start is internal and should be called only by the Controller to register an EventHandler with the Informer
// to enqueue reconcile.Requests.
func (is *Informer) Start(handler handler.EventHandler, queue workqueue.RateLimitingInterface,
  ...
  is.Informer.AddEventHandler(internal.EventHandler{Queue: queue, EventHandler: handler, Predicates: prct})
  return nil
}
```

我们的 Handler 实际注册到 Informer 上面，这样整个逻辑就串起来了，通过 Cache 我们创建了所有 Scheme 里面 GVKs 的 Informers，然后对应 GVK 的 Controller 注册了 Watch Handler 到对应的 Informer，这样一来对应的 GVK 里面的资源有变更都会触发 Handler，将变更事件写到 Controller 的事件队列中，之后触发我们的 Reconcile 方法。

## **Manager 启动**

```go
func (cm *controllerManager) Start(stop <-chan struct{}) error {
  ...
  go cm.startNonLeaderElectionRunnables()
  ...
}
func (cm *controllerManager) startNonLeaderElectionRunnables() {
  ...
  // Start the Cache. Allow the function to start the cache to be mocked out for testing
  if cm.startCache == nil {
    cm.startCache = cm.cache.Start
  }
  go func() {
    if err := cm.startCache(cm.internalStop); err != nil {
      cm.errChan <- err
    }
  }()
        ...
        // Start Controllers
  for _, c := range cm.nonLeaderElectionRunnables {
    ctrl := c
    go func() {
      cm.errChan <- ctrl.Start(cm.internalStop)
    }()
  }
  cm.started = true
}
```

主要就是启动 Cache，Controller，将整个事件流运转起来，我们下面来看看启动逻辑。

## **Cache 启动**

```go
func (ip *specificInformersMap) Start(stop <-chan struct{}) {
  func() {
    ...
    // Start each informer
    for _, informer := range ip.informersByGVK {
      go informer.Informer.Run(stop)
    }
  }()
}
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
        ...
        // informer push resource obj CUD delta to this fifo queue
  fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, s.indexer)
  cfg := &Config{
    Queue:            fifo,
    ListerWatcher:    s.listerWatcher,
    ObjectType:       s.objectType,
    FullResyncPeriod: s.resyncCheckPeriod,
    RetryOnError:     false,
    ShouldResync:     s.processor.shouldResync,
                // handler to process delta
    Process: s.HandleDeltas,
  }
  func() {
    s.startedLock.Lock()
    defer s.startedLock.Unlock()
                // this is internal controller process delta generate by reflector
    s.controller = New(cfg)
    s.controller.(*controller).clock = s.clock
    s.started = true
  }()
        ...
  wg.StartWithChannel(processorStopCh, s.processor.run)
  s.controller.Run(stopCh)
}
func (c *controller) Run(stopCh <-chan struct{}) {
  ...
  r := NewReflector(
    c.config.ListerWatcher,
    c.config.ObjectType,
    c.config.Queue,
    c.config.FullResyncPeriod,
  )
  ...
        // reflector is delta producer
  wg.StartWithChannel(stopCh, r.Run)
        // internal controller's processLoop is comsume logic
  wait.Until(c.processLoop, time.Second, stopCh)
}
```

Cache 的初始化核心是初始化所有的 Informer，Informer 的初始化核心是创建了 reflector 和内部 controller，reflector 负责监听 Api Server 上指定的 GVK，将变更写入 delta 队列中，可以理解为变更事件的生产者，内部 controller 是变更事件的消费者，他会负责更新本地 indexer，以及计算出 CUD 事件推给我们之前注册的 Watch Handler。

**Controller 启动**

```go
// Start implements controller.Controller
func (c *Controller) Start(stop <-chan struct{}) error {
  ...
  for i := 0; i < c.MaxConcurrentReconciles; i++ {
    // Process work items
    go wait.Until(func() {
      for c.processNextWorkItem() {
      }
    }, c.JitterPeriod, stop)
  }
  ...
}
func (c *Controller) processNextWorkItem() bool {
  ...
  obj, shutdown := c.Queue.Get()
  ...
  var req reconcile.Request
  var ok bool
  if req, ok = obj.(reconcile.Request); 
        ...
  // RunInformersAndControllers the syncHandler, passing it the namespace/Name string of the
  // resource to be synced.
  if result, err := c.Do.Reconcile(req); err != nil {
    c.Queue.AddRateLimited(req)
    ...
  } 
        ...
}
```

Controller 的初始化是启动 goroutine 不断地查询队列，如果有变更消息则触发到我们自定义的 Reconcile 逻辑。

## **整体逻辑**

上面我们通过源码阅读已经十分清楚整个流程，如下图所示：

![image-20191029141127411](.\images\20191029141127411.png)

 Kubebuilder 作为脚手架工具已经为我们做了很多，到最后我们只需要实现 Reconcile 方法即可，这里不再赘述。 

## **最佳实践**

### **模式**

\1. 使用 OwnerRefrence 来做资源关联，有两个特性：

- Owner 资源被删除，被 Own 的资源会被级联删除，这利用了 K8s 的 GC；
- 被 Own 的资源对象的事件变更可以触发 Owner 对象的 Reconcile 方法；

\2. 使用 Finalizer 来做资源的清理。

### **注意点**

- 不使用 Finalizer 时，资源被删除无法获取任何信息；
- 对象的 Status 字段变化也会触发 Reconcile 方法；
- Reconcile 逻辑需要幂等。

### **优化**

使用 IndexFunc 来优化资源查询的效率。

# **总结**

通过深入分析，我们可以看到 Kubebuilder 提供的功能对于快速编写 CRD 和 Controller 是十分有帮助的，无论是 Istio、Knative 等知名项目还是各种自定义 Operators，都大量使用了 CRD，将各种组件抽象为 CRD，Kubernetes 变成控制面板将成为一个趋势，希望本文能够帮助大家理解和把握这个趋势。