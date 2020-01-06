# 背景


# Buildpacks简介
## 简介

Buildpacks是可插入的、模块化的工具，通过提供比Dockerfile更高级别的抽象，将源代码转换为容器就绪的构件。通过这样做，他们提供了一种控制的平衡，最小化了最初的生产时间，减少了开发者的操作负担，并支持大规模管理应用程序的企业运营商。

基于从Pivotal和Salesforce Heroku维护产品级构建包（buildpacks）的经验，CNB被构建为提供一个平台到构建包的API契约，该契约获取源代码并输出Docker镜像，这些镜像可以在支持OCI镜像的云平台上运行。

Pivotal公司的工程师兼产品经理Stephen Levine表示：“下一代云原生构建包将帮助开发者和操作人员将应用程序打包成容器，让操作人员能够有效地管理必要的基础设施，以更新应用程序依赖项。我们希望CNB加入CNCF沙箱将进一步提高平台之间的互操作性，并吸引大量贡献者，包括构建包创建者和维护人员。”

Buildpacks最早是由Heroku在2011年构想的。从那以后，他们被Cloud Foundry，以及Gitlab、Knative、Deis(现在的微软)、Dokku和Drie所采用。

Heroku的架构师Joe Kutner表示：“任何人都可以为任何基于Linux的技术创建一个构建包，并与全世界共享。Buildpacks的易用性和灵活性是数百万开发者依赖它们开发关键任务应用的原因。云原生Buildpacks将把这些属性与现代容器标准结合在一起，让开发者能够专注于他们的应用程序，而不是他们的基础设施。”

官网地址：https://buildpacks.io，github地址：https://github.com/buildpacks



## **理解 Cloud Native Buildpacks**

CNB 流程分为四个步骤，每个步骤都有各自的重要目标，最终产出就是 OCI 镜像。CNB 让开发和运维人员能够把创建各种软件的过程中所需的构建、补丁和重新打包的工作自动化成适合机器执行的重复任务。如果 Buildpacks 能够完成容器的构建和管理工作，还需要人工完成么？

1. **检测**：对源码以及其它内容进行检测，查找与其匹配的可用 Buildpacks。假设提供一套 Java 源文件，就会检测到 Java Buildpack 适用于这一输入。
2. **分析**：CNB 会在应用的生命周期中运行多次，在这一步骤里会对前一次的打包内容进行分析，分析过程会对文件的变更进行优化，从而减少构建时间和文件传输。这里会使用多个镜像层来对内容进行组织。
3. **构建**：如果镜像层或者目录需要进行替换，构建过程就会生成新的层。这里会提供缓存来加速构建过程。
4. **导出**：这个步骤中会生成最终镜像并推送到镜像仓库之中。传输、磁盘使用和更新时间都会用镜像层的更新操作来完成。另外 CVE 补丁也可以同时应用到多个镜像之中。

CNB 在 CNCF 生态系统中的旅途才刚刚开始，这其中包含了 Pivotal 客户、Salesforce Heroku 客户以及云原生用户的认可和贡献。很多用户在 Docker 和 Kubernetes 变得炙手可热之前就在 Buildpacks 技术上下了注，现在它们的投资已经成功的应用到了其他生态系统之中。



## 概念

### builder

builder是一个镜像，其中捆绑了有关如何构建应用程序的所有内容和信息，例如buildpack和构建时镜像，以及针对您的应用程序源代码执行buildpack。

builder由Buildpacks、Lifecycle和Stack’s build image组成，如下图所示：

![img](D:\Code\notebook\images\create-builder.svg)

### Buildpack

buildpack是用于检查应用程序源代码并制定计划来构建和运行您的应用程序的工作单元。

典型的构建包是由如下三个文件构成：

- `buildpack.toml` –buildpack的元数据配置描述文件
- `bin/detect` –确定是否应使用buildpack
- `bin/build` –执行buildpack逻辑

还有另一种类型的buildpack，通常称为**meta-buildpack**。它仅包含`buildpack.toml`具有`order`引用其他buildpack 的配置的 文件。这对于组合更复杂的检测策略很有用。

#### detect

平台会针对您的应用程序的源代码顺序测试每个buildpack。第一个认为自己适合您的源代码的组将成为您的应用程序的选定构建包集。检测标准是特定于每个buildpack的，例如，一个NPM buildpack可能会寻找一个`package.json`，而Go buildpack可能会寻找Go源文件。

#### build

通过build过程buildpacks会构成最终的应用镜像。这个过程可能是简单的为镜像设置一些环境变量，创建包含一个二进制的layer（例如：node，Python或ruby），或添加的应用的依赖关系（例如：运行`npm install`，`pip install -r requirements.txt`或`bundle install`）。

### Lifecycle

Lifecycle安排buildpack的执行，然后将生成的工件组合成最终的应用镜像。针对内网环境需重新编译源码使之支持内网的非安全的镜像仓库。

1. **检测**：对源码以及其它内容进行检测，查找与其匹配的可用 Buildpacks。假设提供一套 Java 源文件，就会检测到 Java Buildpack 适用于这一输入。
2. **分析**：CNB 会在应用的生命周期中运行多次，在这一步骤里会对前一次的打包内容进行分析，分析过程会对文件的变更进行优化，从而减少构建时间和文件传输。这里会使用多个镜像层来对内容进行组织。
3. **构建**：如果镜像层或者目录需要进行替换，构建过程就会生成新的层。这里会提供缓存来加速构建过程。
4. **导出**：这个步骤中会生成最终镜像并推送到镜像仓库之中。传输、磁盘使用和更新时间都会用镜像层的更新操作来完成。另外 CVE 补丁也可以同时应用到多个镜像之中。

### Stack

Stack以镜像的形式为buildpack生命周期提供构建时和运行时环境。

> 如果使用的是`pack`CLI，则running `pack suggest-stacks`将显示建议的堆栈列表，这些堆栈可在运行时使用`pack create-builder`，以及每个堆栈的关联构建和运行映像。



Stack是提供给builder使用的，并通过builder的配置文件进行配置：

```toml
[[buildpacks]]
  # ...

[[order]]
  # ...

[stack]
  id = "com.example.stack"
  build-image = "example/build"
  run-image = "example/run"
  run-image-mirrors = ["gcr.io/example/run", "registry.example.com/example/run"]
```

通过提供必需的`[stack]`部分，builder作者可以配置堆栈的ID，build-image和run-image，和任何镜像mirrors。

可以通过如下方式在执行build时配置镜像mirrors。

```bash
$ pack build registry.example.com/example/app
```

在未指定注册表的情况下命名应用时`example/app`，将导致`example/run`被选择为应用的运行镜像。

```bash
$ pack build example/app
```

> 对于本地开发，重写构建器中的运行映像镜像通常会很有帮助。为此， `set-run-image-mirrors`可以使用命令。该命令不会修改构建器，而是配置用户的本地计算机。
>
> 要查看为构建器配置了哪些运行映像，`inspect-builder`可以使用该 命令。`inspect-builder`将输出给定构建器的内置和本地配置的运行映像，以及其他有用信息。输出中的运行图像的顺序表示在期间它们将被匹配的顺序`build`。



要创建自定义stacks，只需创建自定义的build并运行包含以下信息的镜像：

***labels***

| 名称                     | 描述         | 格式 |
| :----------------------- | ------------ | ---- |
| `io.buildpacks.stack.id` | 堆栈的标识符 | 串   |

***environment-variables***

| 名称           | 描述                  |
| -------------- | --------------------- |
| `CNB_STACK_ID` | stack的标识符         |
| `CNB_USER_ID`  | 镜像中指定的用户的UID |
| `CNB_GROUP_ID` | 镜像中指定的用户的GID |



> **注：**该**标识符堆栈**意味着与相同标识符的其他堆的相容性。例如，自定义堆栈可以`io.buildpacks.stacks.bionic`用作其标识符，只要它可用于声明与`io.buildpacks.stacks.bionic`stacks兼容的buildpack 。

## 操作

### build

构建是对应用程序的源代码执行一个或多个构建包以生成可运行的OCI映像的过程。每个buildpack都会检查源代码并提供相关的依赖关系。然后从应用程序的源代码和这些依赖项生成图像。

Buildpack与一个或多个[堆栈](https://buildpacks.io/docs/concepts/components/stack)兼容。堆栈指定**构建映像** 和**运行映像**。在构建过程中，堆栈的构建映像成为执行buildpacks的环境，并且其运行映像成为最终应用程序映像的基础。有关使用堆栈的更多信息，请参见[使用堆栈](https://buildpacks.io/docs/concepts/components/stack)部分。

可以将Buildpacks与特定堆栈的构建映像捆绑在一起，从而生成一个 [构建器](https://buildpacks.io/docs/concepts/components/builder)映像（请注意“ er”结尾）。生成器为分配给定堆栈的构建包提供了最方便的方法。有关使用构建器的更多信息，请参见 [使用构建器使用`create-builder`](https://buildpacks.io/docs/concepts/components/builder)部分。

![img](D:\Code\notebook\images\build.svg)

### Rebase

使用Rebase，应用程序开发人员或操作员可以在堆栈的运行映像已更改时快速更新其映像。通过使用图像层重新基准化，此命令避免了完全重建应用程序的需要。

![变基图](https://buildpacks.io/docs/concepts/operations/rebase.svg)

从本质上讲，图像重定基址是一个简单的过程。通过检查应用程序映像，`rebase`可以确定是否存在应用程序基本映像的更新版本（本地或注册表中）。如果是这样，请`rebase`更新应用程序图像的图层元数据以引用较新的基础图像版本。

example-rebasing-an-app-image

考虑`my-app:my-tag`最初使用默认构建器构建的应用程序图像。该构建器的堆栈具有名为的运行映像`pack/run`。运行以下命令将`my-app:my-tag`使用的最新版本 更新的基础`pack/run`。

```bash
$ pack rebase my-app:my-tag
```

`rebase`具有`--publish`可用于将更新的应用程序映像发布到注册表的标志。

------



# 测试使用

## 简单使用

安装：

```bash
wget https://github.com/buildpacks/pack/releases/download/v0.6.0/pack-v0.6.0-linux.tgz
tar xvf pack-v0.6.0-linux.tgz
rm pack-v0.6.0-linux.tgz
./pack --help
```

使用pack build命令即可将应用代码构建并生成镜像，整个构建过程中应用无需安装JDK，运行Maven或配置构建环境 。

```bash
# clone the repo
git clone https://github.com/buildpacks/samples

# go to the app directory
cd samples/apps/java-maven

# build the app
pack build myapp --builder cnbs/sample-builder:bionic
```

通过docker run运行应用：

```bash
docker run --rm -p 8080:8080 myapp
```

浏览器输入`localhost:8080`即可看到应用的页面。

## Tekton Pipeline