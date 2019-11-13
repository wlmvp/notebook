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

安装Golang（version >= 1.11）

配置GOPATH

开启go module （`export GO111MODULE=on`） 

## 安装kubebuilder

```bash
$ wget https://github.com/kubernetes-sigs/kubebuilder/releases/download/v2.0.1/kubebuilder_2.0.1_linux_amd64.tar.gz
$ tar xzf kubebuilder_2.0.1_linux_amd64.tar.gz
$ mv kubebuilder_2.0.1_linux_amd64 -c /usr/local/kubebuilder
$ export PATH=$PATH:/usr/local/kubebuilder/bin
$ kubebuilder version
Version: version.Version{KubeBuilderVersion:"2.0.1", KubernetesVendor:"1.14.1", GitCommit:"855513fa7eec932f8fcd1c28c02a139c222413af", BuildDate:"2019-09-24T17:37:32Z", GoOs:"unknown", GoArch:"unknown"}

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

# 测试使用

## 开发自己的operator

### 1、创建工程目录

在golang的工作目录src下创建自己的工程，工程域名和模块名称

```bash
[root@node3 src]# mkdir -p cloud.io/etcd-op 
```

### 2、初始化工程依赖

注意开启 export GO111MODULE=on

```bash
[root@node3 src]# cd cloud.io/etcd-op/
[root@node3 etcd-op]# export GO111MODULE=on
[root@node3 etcd-op]# go mod init etcd-op
go: creating new go.mod: module etcd-op
```

### 3、初始化工程

初始化operator工程， 创建脚手架工程

 这一步创建了一个 Go module 工程，引入了必要的依赖，创建了一些模板文件。 

此处会有包下载不了的问题，需科学上网

```bash
[root@node3 etcd-op]# kubebuilder init --domain cloud.io
go mod tidy
此处省略n行...
```

#在实际创建过程中由于golang很多包无妨下载，需要在go.mod中replace,文件内容贴在文章末尾

### **4、创建 API** 

 这一步创建了对应的 CRD 和 Controller 模板文件，经过 1、2 两步，现有的工程结构如图 2 所示： 

```
[root@node3 etcd-op]# kubebuilder create api --group etcd --version v1 --kind EtcdCluster
Create Resource [y/n]
y
Create Controller [y/n]
y
Writing scaffold for you to edit...
此处省略n行...
go build -o bin/manager main.go
```

## 生成的工程结构

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