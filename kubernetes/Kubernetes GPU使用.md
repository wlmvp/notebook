# 背景
随着人工智能、机器学习、深度学习等技术的迅速发展，GPU近年来也得到了快速的发展，基于GPU的大数据运算越来越普及。GPU 可以大大加快深度学习任务的运行速度，同时，GPU资源又是十分昂贵的，需要尽可能提高 GPU 资源的利用率，改善其开发和部署平台对于提高机器学习应用的应用开发、测试以及上线效率提升非常关键。

Kubernetes作为容器管理平台，已经成为容器编排领域的事实标准，通过kubernetes技术可实现容器的有效管理，资源的合理分配，任务的高效调度等，为上层应用提供强有力的支撑。

GPU高性能计算和容器编排已成为AI重要支撑技术，需利用Kubernetes 将 GPU 资源聚合成资源池来实现统一管理，使得Kubernetes集群中可以使用GPU的运算能力，并借用容器交付深度学习的运行时环境。

# 简介

在kubernetes的发展规划中，GPU资源有这非常重要的地位。用户能够为其工作任务请求GPU资源，就像请求CPU和内存一样，而Kubernetes将负责调度容器到具有GPU资源的节点上。目前Kubernetes对NVIDIA和AMD两个厂商的GPU进行了实验性的支持。其中对NVIDIA GPU的支持从1.6版本开始，对AMD GPU的支持从1.9版本开始，并且都在快速发展。

## Device Plugin 

Kubernetes从1.8版本开始，引入了 DevicePlugins 模型， 为设备提供商提供了一种基于插件的、无需修改kublet核心代码的外部设备启动方式，由第三方设备厂商开发插件，实现Kubernetes Device Plugins的相关接口即可。 

目前关注度比较高的Device Plugins实现有：

- Nvidia提供的GPU插件：[NVIDIA device plugin for Kubernetes](https://github.com/NVIDIA/k8s-device-plugin)
- 高性能低延迟RDMA卡插件：[RDMA device plugin for Kubernetes](https://github.com/hustcat/k8s-rdma-device-plugin.git)
- 低延迟Solarflare万兆网卡驱动：[Solarflare Device Plugin](https://github.com/vikaschoudhary16/sfc-device-plugin)

从1.8开始,官方建议调用 GPUs 的方法是通过使用 device plugins，具体说明详见官方文档https://kubernetes.io/docs/concepts/cluster-administration/device-plugins。

为了启用 GPU支持，在1.10之前, 该`DevicePlugins` feature gate 需要通过kubelet服务开启 `--feature-gates="DevicePlugins=true"`特性开关。但在 1.10及以后，默认启动，不再需要手动设置。

Device 插件实际上是一个 gPRC 接口，需要实现 ListAndWatch() 和 Allocate() 等方法，并监听 gRPC Server 的 Unix Socket ，在 /var/lib/kubelet/device-plugins/ 目录中。 

使用NVIDIA GPU device plugin之前，需要有如下配置：

- **1、配置docker runtime**

vim /lib/systemd/system/docker.service

修改docker启动参数，配置docker默认使用NVIDIA运行时：

```
--add-runtime nvidia=/usr/bin/nvidia-container-runtime
--default-runtime=nvidia
```

或者配置文件`/etc/docker/daemon.json`:

```
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

重启服务：

```
systemctl daemon-reload
systemctl restart docker.service
```

- **2、安装NVIDIA驱动**

安装的NVIDIA驱动版本为361.93及以上。

```
# Install official NVIDIA driver package
sudo apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/7fa2af80.pub
sudo sh -c 'echo "deb http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64 /" > /etc/apt/sources.list.d/cuda.list'
sudo apt-get update && sudo apt-get install -y --no-install-recommends linux-headers-generic dkms cuda-drivers
```

- **3、安装NVIDIA GPU device plugin**

直接执行yaml文件进行安装

```
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/1.0.0-beta4/nvidia-device-plugin.yml
```

具体的yaml配置内容如下，可以看到安装device plugin即为在每个计算节点上以DaemonSet方式启动一个设备插件容器供kubelet调用。

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: nvidia-device-plugin-ds
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      # This annotation is deprecated. Kept here for backward compatibility
      # See https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ""
      labels:
        name: nvidia-device-plugin-ds
    spec:
      tolerations:
      # This toleration is deprecated. Kept here for backward compatibility
      # See https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
      - key: CriticalAddonsOnly
        operator: Exists
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      # Mark this pod as a critical add-on; when enabled, the critical add-on
      # scheduler reserves resources for critical add-on pods so that they can
      # be rescheduled after a failure.
      # See https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
      priorityClassName: "system-node-critical"
      containers:
      - image: nvidia/k8s-device-plugin:1.0.0-beta4
        name: nvidia-device-plugin-ctr
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
        volumeMounts:
          - name: device-plugin
            mountPath: /var/lib/kubelet/device-plugins
      volumes:
        - name: device-plugin
          hostPath:
            path: /var/lib/kubelet/device-plugins
```

## 在容器中使用GPU资源

GPU资源在kubernetes中的名称为nvidia.com/gpu（NVIDIA类型）或amd.com/gpu（AMD类型），可以对容器进行GPU资源请求的设置。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
    - name: cuda-container
      image: nvidia/cuda:9.0-devel
      resources:
        limits:
          nvidia.com/gpu: 2 # requesting 2 GPUs
    - name: digits-container
      image: nvidia/digits:6.0
      resources:
        limits:
          nvidia.com/gpu: 2 # requesting 2 GPUs
```

目前对GPU资源的使用配置有如下限制：

- GPU资源请求只能在limits字段进行设置，不支持设置requests，系统默认requests的值等于和limits的值。

- 在多个容器之间或者在多个pod之间不能共享GPU资源
- 每个容器只能请求整数个（1个或多个）GPU资源