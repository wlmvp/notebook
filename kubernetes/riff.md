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



## 概念分析

### builder

builder源码地址：https://github.com/projectriff/builder

编译生成builder：

```
[root@izttsn6fen661yz builder]# make
./ci/apply-template.sh builder.toml.tpl > builder.toml
pack create-builder -b builder.toml projectriff/builder
base-cnb: Pulling from cloudfoundry/build
7ddbc47eeb70: Already exists 
c1bbdc448b72: Already exists 
8c3b70e39044: Already exists 
45d437916d57: Already exists 
4399c5188cb9: Pull complete 
b086dd9ab025: Pull complete 
f6247bc87853: Pull complete 
bce1fb13fef8: Pull complete 
Digest: sha256:b30b850f5b4d2e11313f0ec152952eace285ce7a3bc203ca5cdcfa8e5bb232a6
Status: Downloaded newer image for cloudfoundry/build:base-cnb
Downloading from https://storage.googleapis.com/projectriff/command-function-buildpack/io.projectriff.command-0.0.10-BUILD-SNAPSHOT-20191120141213-a86527d33694374e.tgz
9.4 MB/9.4 MB 
Downloading from https://storage.googleapis.com/projectriff/java-function-buildpack/io.projectriff.java-0.2.0-BUILD-SNAPSHOT-20191206161631-b3b925eeb01d41f3.tgz
36.1 MB/36.1 MB
Downloading from https://storage.googleapis.com/projectriff/node-function-buildpack/io.projectriff.node-0.2.0-BUILD-SNAPSHOT-20191205213235-4fe358af155c1fa9.tgz
12 MB/12 MB  
Downloading from https://storage.googleapis.com/cnb-buildpacks/build-system-cnb/org.cloudfoundry.buildsystem-1.0.162.tgz
110 MB/110 MB 
Downloading from https://github.com/cloudfoundry/node-engine-cnb/releases/download/v0.0.126/node-engine-cnb-0.0.126.tgz
5.46 MB/5.46 MB
Downloading from https://github.com/cloudfoundry/npm-cnb/releases/download/v0.0.77/npm-cnb-0.0.77.tgz
5.03 MB/5.03 MB
Downloading from https://storage.googleapis.com/cnb-buildpacks/openjdk-cnb/org.cloudfoundry.openjdk-1.0.70.tgz
641 MB/641 MB 
Successfully created builder image projectriff/builder
Tip: Run pack build <image-name> --builder projectriff/builder to use this builder
```

可以看到生成builder的时候拉取了java-function-buildpack包。

### java-function-buildpack

地址为：https://github.com/projectriff/java-function-buildpack

直接编译，发现打包有点问题，好在build、detect文件已经编译生成，可以手工打包。

```
[root@izttsn6fen661yz java-function-buildpack]# make
go test -v ./...
?   	github.com/projectriff/java-function-buildpack/build	[no test files]
?   	github.com/projectriff/java-function-buildpack/detect	[no test files]
=== RUN   TestBuildpack
Suite: Buildpack
Total: 2 | Focused: 0 | Pending: 0
=== RUN   TestBuildpack/Buildpack
=== RUN   TestBuildpack/Buildpack/id/returns_id
=== RUN   TestBuildpack/Buildpack/passes_with_handler

Passed: 2 | Failed: 0 | Skipped: 0

--- PASS: TestBuildpack (0.00s)
    --- PASS: TestBuildpack/Buildpack (0.00s)
        --- PASS: TestBuildpack/Buildpack/id/returns_id (0.00s)
        --- PASS: TestBuildpack/Buildpack/passes_with_handler (0.00s)
=== RUN   TestFunction
Suite: Function
Total: 4 | Focused: 0 | Pending: 0
=== RUN   TestFunction/Function
=== RUN   TestFunction/Function/returns_true_if_build_plan_exists
=== RUN   TestFunction/Function/returns_false_if_build_plan_does_not_exist
=== RUN   TestFunction/Function/contributes_function_class_to_launch
=== RUN   TestFunction/Function/contributes_function_definition_to_launch

Passed: 4 | Failed: 0 | Skipped: 0

--- PASS: TestFunction (0.00s)
    --- PASS: TestFunction/Function (0.00s)
        --- PASS: TestFunction/Function/returns_true_if_build_plan_exists (0.00s)
        --- PASS: TestFunction/Function/returns_false_if_build_plan_does_not_exist (0.00s)
        --- PASS: TestFunction/Function/contributes_function_class_to_launch (0.00s)
        --- PASS: TestFunction/Function/contributes_function_definition_to_launch (0.00s)
=== RUN   TestInvoker
Suite: Invoker
Total: 3 | Focused: 0 | Pending: 0
=== RUN   TestInvoker/Invoker
=== RUN   TestInvoker/Invoker/returns_true_if_build_plan_exists
=== RUN   TestInvoker/Invoker/returns_false_if_build_plan_does_not_exist
=== RUN   TestInvoker/Invoker/contributes_invoker_to_launch

Passed: 3 | Failed: 0 | Skipped: 0

--- PASS: TestInvoker (0.00s)
    --- PASS: TestInvoker/Invoker (0.00s)
        --- PASS: TestInvoker/Invoker/returns_true_if_build_plan_exists (0.00s)
        --- PASS: TestInvoker/Invoker/returns_false_if_build_plan_does_not_exist (0.00s)
        --- PASS: TestInvoker/Invoker/contributes_invoker_to_launch (0.00s)
=== RUN   TestStreamingHTTPAdapter
Suite: Invoker
Total: 1 | Focused: 0 | Pending: 0
=== RUN   TestStreamingHTTPAdapter/Invoker
=== RUN   TestStreamingHTTPAdapter/Invoker/contributes_invoker_to_launch

Passed: 1 | Failed: 0 | Skipped: 0

--- PASS: TestStreamingHTTPAdapter (0.00s)
    --- PASS: TestStreamingHTTPAdapter/Invoker (0.00s)
        --- PASS: TestStreamingHTTPAdapter/Invoker/contributes_invoker_to_launch (0.00s)
PASS
ok  	github.com/projectriff/java-function-buildpack/java	(cached)
rm -fR artifactory/io/projectriff/java/io.projectriff.java 							&& \
./ci/package.sh						&& \
mkdir artifactory/io/projectriff/java/io.projectriff.java/latest 					&& \
tar -C artifactory/io/projectriff/java/io.projectriff.java/latest -xzf artifactory/io/projectriff/java/io.projectriff.java/*/*.tgz

Java Function Buildpack 0.2.0-BUILD-SNAPSHOT
  Pre-Package with ci/build.sh
  Caching riff Java Invoker 0.2.0+snapshot
    Downloading from https://storage.googleapis.com/projectriff/java-function-invoker/releases/v0.2.0-SNAPSHOT/snapshots/java-function-invoker-0.2.0-SNAPSHOT-6b902e7d36c8a6755c1c493a0bd6179a49287090.jar
    Verifying checksum
  Caching riff Streaming HTTP Adapter 0.0.1
    Downloading from https://storage.googleapis.com/projectriff/streaming-http-adapter/streaming-http-adapter-linux-amd64-0.1.0-snapshot-20190923123008-a6813ca902b44f89.tgz
    Verifying checksum
  Creating package in /tmp/tmp.XYtrFU0giQ
    Adding LICENSE
    Adding NOTICE
    Adding README.md
    Adding bin/build
    Adding bin/detect
    Adding buildpack.toml
    Adding dependency-cache/05ac25e9b12db3b78e9c26008b684e53fe78b12145d1e448608aec465602046c/java-function-invoker-0.2.0-SNAPSHOT-6b902e7d36c8a6755c1c493a0bd6179a49287090.jar
    Adding dependency-cache/05ac25e9b12db3b78e9c26008b684e53fe78b12145d1e448608aec465602046c.toml
    Adding dependency-cache/43583761b3f771c4c821b6956ef4570e34c76704deb5992c608cfba1bd694d73/streaming-http-adapter-linux-amd64-0.1.0-snapshot-20190923123008-a6813ca902b44f89.tgz
    Adding dependency-cache/43583761b3f771c4c821b6956ef4570e34c76704deb5992c608cfba1bd694d73.toml
tar: Dec: Cannot stat: No such file or directory
tar: 23: Cannot stat: No such file or directory
tar: 21\:11\:27: Cannot stat: No such file or directory
tar: 2019: Cannot stat: No such file or directory
tar: +0530-bdb19f01783eb579.tgz: Cannot stat: No such file or directory
tar: Exiting with failure status due to previous errors
make: *** [artifactory/io/projectriff/java/io.projectriff.java] Error 2
[root@izttsn6fen661yz java-function-buildpack]# 
[root@izttsn6fen661yz java-function-buildpack]# cd bin/
[root@izttsn6fen661yz bin]# ll
total 21944
-rwxr-xr-x 1 root root 6832128 Dec 30 14:09 build
-rwxr-xr-x 1 root root 6828032 Dec 30 14:09 detect
-rwxr-xr-x 1 root root 8810496 Dec 30 14:09 package
```

可以看到编译的过程中下载了java-function-invoker和streaming-http-adapter。

### java-function-invoker

地址：https://github.com/projectriff/java-function-invoker

查看源码，可以看到，java-function-invoker即为一个springboot程序，主要实现就是启动了一个GRPC服务。

```java
package io.projectriff.invoker.main;

import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.projectriff.invoker.rpc.StartFrame;
import io.projectriff.invoker.server.GrpcServerAdapter;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.function.context.FunctionCatalog;
import org.springframework.cloud.function.context.FunctionProperties;
import org.springframework.context.SmartLifecycle;
import org.springframework.context.annotation.Bean;

import java.io.IOException;

/**
 * This is the main entry point for the java function invoker.
 * This sets up an application context with the whole Spring Cloud Function infrastructure (thanks to auto-configuration)
 * setup, pointing to the user function (via correctly set {@link FunctionProperties} ConfigurationProperties.
 * Then exposes a gRPC server adapting this function to the riff RPC protocol (muxing/de-muxing input and output values
 * over a single streaming channel). Marshalling and unmarshalling of byte encoded values is performed by Spring Cloud Function
 * itself, according to the incoming {@code Content-Type} header and the {@link StartFrame#getExpectedContentTypesList() expectedContentType} fields.
 *
 * @author Eric Bottard
 */
@SpringBootApplication
public class EntryPoint {

    @Value("#{systemEnvironment['GRPC_PORT'] ?: 8081}")
    private int grpcPort = 8081;

    public static void main(String[] args) throws InterruptedException {
        SpringApplication.run(EntryPoint.class, args);
        Object o = new Object();
        synchronized (o) {
            o.wait();
        }
    }

    @Bean
    public GrpcServerAdapter adapter(FunctionCatalog functionCatalog, FunctionProperties functionProperties) {
        return new GrpcServerAdapter(
                functionCatalog,
                functionProperties.getDefinition()
        );
    }

    @Bean
    public SmartLifecycle server(GrpcServerAdapter adapter) {
        Server server = ServerBuilder.forPort(grpcPort).addService(adapter).build();
        return new SmartLifecycle() {

            private volatile boolean running;

            @Override
            public void start() {
                try {
                    server.start();
                    running = true;
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
            }

            @Override
            public void stop() {
                server.shutdown();
                running = false;
            }

            @Override
            public boolean isRunning() {
                return running;
            }
        };
    }

}
```

### streaming-http-adapter

可以看到streaming-http-adapter起了一个代理，用于代理8080端口http请求到8081端口的GRPC服务。 

```go
func main() {
	// Lookup the gRPC address where our child process expects to run.
	grpcPort := os.Getenv("GRPC_PORT")
	if grpcPort == "" {
		grpcPort = "8081"
	}

	// Per the http-invoker contract, listen for http traffic on a port defined by PORT
	httpPort := os.Getenv("PORT")
	if httpPort == "" {
		httpPort = "8080"
	}
	proxy, err := proxy.NewProxy(fmt.Sprintf(":%s", grpcPort), fmt.Sprintf(":%s", httpPort))
	if err != nil {
		panic(err)
	}
	go func() {
		if err := proxy.Run(); err != nil {
			log.Fatalf("error running proxy %v", err)
		}
	}()
```
### 应用镜像构建

java函数用例：

直接实现function的用法：

https://github.com/projectriff-samples/java-hello

采用springboot框架的用法：

https://github.com/projectriff-samples/java-boot-uppercase



编译生成镜像：

```bash
[root@izttsn6fen661yz java-hello]# pack build --builder projectriff/builder --no-pull -p . your-org/my-function
===> DETECTING
[detector] org.cloudfoundry.openjdk     1.0.70
[detector] org.cloudfoundry.buildsystem 1.0.162
[detector] io.projectriff.java          0.2.0-BUILD-SNAPSHOT
===> RESTORING
[restorer] Cache '/cache': metadata not found, nothing to restore
===> ANALYZING
[analyzer] Warning: Image 'index.docker.io/your-org/my-function:latest' not found
===> BUILDING
[builder] 
[builder] Cloud Foundry OpenJDK Buildpack 1.0.70
[builder]   OpenJDK JDK 11.0.5: Contributing to layer
[builder]     Reusing cached download from buildpack
[builder]     Expanding to /layers/org.cloudfoundry.openjdk/openjdk-jdk
[builder]     Writing JAVA_HOME to build
[builder]     Writing JDK_HOME to build
[builder]   OpenJDK JRE 11.0.5: Contributing to layer
[builder]     Reusing cached download from buildpack
[builder]     Expanding to /layers/org.cloudfoundry.openjdk/openjdk-jre
[builder]     Writing JAVA_HOME to shared
[builder] 
[builder] Cloud Foundry Build System Buildpack 1.0.162
[builder]   Apache Maven 3.6.3: Contributing to layer
[builder]     Reusing cached download from buildpack
[builder]     Expanding to /layers/org.cloudfoundry.buildsystem/maven
[builder]     Linking Cache to /home/cnb/.m2
[builder]   Compiled Application: Contributing to layer
[builder]     Executing /layers/org.cloudfoundry.buildsystem/maven/bin/mvn -Dmaven.test.skip=true package
[builder] [INFO] Scanning for projects...
[builder] [INFO] 
[builder] [INFO] --------------------< io.projectriff.samples:hello >--------------------
[builder] [INFO] Building hello 1.0.0
[builder] [INFO] --------------------------------[ jar ]---------------------------------
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-resources-plugin/2.6/maven-resources-plugin-2.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-resources-plugin/2.6/maven-resources-plugin-2.6.pom (8.1 kB at 4.8 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-plugins/23/maven-plugins-23.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-plugins/23/maven-plugins-23.pom (9.2 kB at 24 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-parent/22/maven-parent-22.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-parent/22/maven-parent-22.pom (30 kB at 53 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/apache/11/apache-11.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/apache/11/apache-11.pom (15 kB at 39 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-resources-plugin/2.6/maven-resources-plugin-2.6.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-resources-plugin/2.6/maven-resources-plugin-2.6.jar (30 kB at 74 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-compiler-plugin/3.1/maven-compiler-plugin-3.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-compiler-plugin/3.1/maven-compiler-plugin-3.1.pom (10 kB at 25 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-plugins/24/maven-plugins-24.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-plugins/24/maven-plugins-24.pom (11 kB at 27 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-parent/23/maven-parent-23.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-parent/23/maven-parent-23.pom (33 kB at 81 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/apache/13/apache-13.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/apache/13/apache-13.pom (14 kB at 37 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-compiler-plugin/3.1/maven-compiler-plugin-3.1.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-compiler-plugin/3.1/maven-compiler-plugin-3.1.jar (43 kB at 106 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-surefire-plugin/2.12.4/maven-surefire-plugin-2.12.4.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-surefire-plugin/2.12.4/maven-surefire-plugin-2.12.4.pom (10 kB at 28 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/surefire/2.12.4/surefire-2.12.4.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/surefire/2.12.4/surefire-2.12.4.pom (14 kB at 37 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-surefire-plugin/2.12.4/maven-surefire-plugin-2.12.4.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-surefire-plugin/2.12.4/maven-surefire-plugin-2.12.4.jar (30 kB at 82 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-jar-plugin/2.4/maven-jar-plugin-2.4.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-jar-plugin/2.4/maven-jar-plugin-2.4.pom (5.8 kB at 16 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-plugins/22/maven-plugins-22.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-plugins/22/maven-plugins-22.pom (13 kB at 34 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-parent/21/maven-parent-21.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-parent/21/maven-parent-21.pom (26 kB at 69 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/apache/10/apache-10.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/apache/10/apache-10.pom (15 kB at 39 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-jar-plugin/2.4/maven-jar-plugin-2.4.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-jar-plugin/2.4/maven-jar-plugin-2.4.jar (34 kB at 92 kB/s)
[builder] [INFO] 
[builder] [INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ hello ---
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-api/2.0.6/maven-plugin-api-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-api/2.0.6/maven-plugin-api-2.0.6.pom (1.5 kB at 4.0 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven/2.0.6/maven-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven/2.0.6/maven-2.0.6.pom (9.0 kB at 24 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-parent/5/maven-parent-5.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-parent/5/maven-parent-5.pom (15 kB at 41 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/apache/3/apache-3.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/apache/3/apache-3.pom (3.4 kB at 9.1 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-project/2.0.6/maven-project-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-project/2.0.6/maven-project-2.0.6.pom (2.6 kB at 7.1 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-settings/2.0.6/maven-settings-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-settings/2.0.6/maven-settings-2.0.6.pom (2.0 kB at 5.2 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-model/2.0.6/maven-model-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-model/2.0.6/maven-model-2.0.6.pom (3.0 kB at 8.2 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.4.1/plexus-utils-1.4.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.4.1/plexus-utils-1.4.1.pom (1.9 kB at 5.1 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/1.0.11/plexus-1.0.11.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/1.0.11/plexus-1.0.11.pom (9.0 kB at 24 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-container-default/1.0-alpha-9-stable-1/plexus-container-default-1.0-alpha-9-stable-1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-container-default/1.0-alpha-9-stable-1/plexus-container-default-1.0-alpha-9-stable-1.pom (3.9 kB at 11 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-containers/1.0.3/plexus-containers-1.0.3.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-containers/1.0.3/plexus-containers-1.0.3.pom (492 B at 1.3 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/1.0.4/plexus-1.0.4.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/1.0.4/plexus-1.0.4.pom (5.7 kB at 15 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/junit/junit/3.8.1/junit-3.8.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/junit/junit/3.8.1/junit-3.8.1.pom (998 B at 2.6 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.0.4/plexus-utils-1.0.4.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.0.4/plexus-utils-1.0.4.pom (6.9 kB at 18 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/classworlds/classworlds/1.1-alpha-2/classworlds-1.1-alpha-2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/classworlds/classworlds/1.1-alpha-2/classworlds-1.1-alpha-2.pom (3.1 kB at 8.4 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-profile/2.0.6/maven-profile-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-profile/2.0.6/maven-profile-2.0.6.pom (2.0 kB at 5.1 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact-manager/2.0.6/maven-artifact-manager-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact-manager/2.0.6/maven-artifact-manager-2.0.6.pom (2.6 kB at 7.0 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-repository-metadata/2.0.6/maven-repository-metadata-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-repository-metadata/2.0.6/maven-repository-metadata-2.0.6.pom (1.9 kB at 4.9 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact/2.0.6/maven-artifact-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact/2.0.6/maven-artifact-2.0.6.pom (1.6 kB at 4.2 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-registry/2.0.6/maven-plugin-registry-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-registry/2.0.6/maven-plugin-registry-2.0.6.pom (1.9 kB at 5.0 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-core/2.0.6/maven-core-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-core/2.0.6/maven-core-2.0.6.pom (6.7 kB at 18 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-parameter-documenter/2.0.6/maven-plugin-parameter-documenter-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-parameter-documenter/2.0.6/maven-plugin-parameter-documenter-2.0.6.pom (1.9 kB at 5.1 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/reporting/maven-reporting-api/2.0.6/maven-reporting-api-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/reporting/maven-reporting-api/2.0.6/maven-reporting-api-2.0.6.pom (1.8 kB at 4.8 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/reporting/maven-reporting/2.0.6/maven-reporting-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/reporting/maven-reporting/2.0.6/maven-reporting-2.0.6.pom (1.4 kB at 4.0 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/doxia/doxia-sink-api/1.0-alpha-7/doxia-sink-api-1.0-alpha-7.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/doxia/doxia-sink-api/1.0-alpha-7/doxia-sink-api-1.0-alpha-7.pom (424 B at 1.2 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/doxia/doxia/1.0-alpha-7/doxia-1.0-alpha-7.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/doxia/doxia/1.0-alpha-7/doxia-1.0-alpha-7.pom (3.9 kB at 11 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-error-diagnostics/2.0.6/maven-error-diagnostics-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-error-diagnostics/2.0.6/maven-error-diagnostics-2.0.6.pom (1.7 kB at 4.7 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/commons-cli/commons-cli/1.0/commons-cli-1.0.pom
Downloaded from central: https://repo.maven.apache.org/maven2/commons-cli/commons-cli/1.0/commons-cli-1.0.pom (2.1 kB at 5.7 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-descriptor/2.0.6/maven-plugin-descriptor-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-descriptor/2.0.6/maven-plugin-descriptor-2.0.6.pom (2.0 kB at 5.4 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interactivity-api/1.0-alpha-4/plexus-interactivity-api-1.0-alpha-4.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interactivity-api/1.0-alpha-4/plexus-interactivity-api-1.0-alpha-4.pom (7.1 kB at 19 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-monitor/2.0.6/maven-monitor-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-monitor/2.0.6/maven-monitor-2.0.6.pom (1.3 kB at 3.4 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/classworlds/classworlds/1.1/classworlds-1.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/classworlds/classworlds/1.1/classworlds-1.1.pom (3.3 kB at 9.0 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/2.0.5/plexus-utils-2.0.5.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/2.0.5/plexus-utils-2.0.5.pom (3.3 kB at 8.9 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/2.0.6/plexus-2.0.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/2.0.6/plexus-2.0.6.pom (17 kB at 45 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-filtering/1.1/maven-filtering-1.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-filtering/1.1/maven-filtering-1.1.pom (5.8 kB at 16 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-components/17/maven-shared-components-17.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-components/17/maven-shared-components-17.pom (8.7 kB at 23 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.5.15/plexus-utils-1.5.15.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.5.15/plexus-utils-1.5.15.pom (6.8 kB at 18 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/2.0.2/plexus-2.0.2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/2.0.2/plexus-2.0.2.pom (12 kB at 31 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interpolation/1.12/plexus-interpolation-1.12.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interpolation/1.12/plexus-interpolation-1.12.pom (889 B at 2.3 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-components/1.1.14/plexus-components-1.1.14.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-components/1.1.14/plexus-components-1.1.14.pom (5.8 kB at 16 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/sonatype/plexus/plexus-build-api/0.0.4/plexus-build-api-0.0.4.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/sonatype/plexus/plexus-build-api/0.0.4/plexus-build-api-0.0.4.pom (2.9 kB at 7.5 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/sonatype/spice/spice-parent/10/spice-parent-10.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/sonatype/spice/spice-parent/10/spice-parent-10.pom (3.0 kB at 7.7 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/sonatype/forge/forge-parent/3/forge-parent-3.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/sonatype/forge/forge-parent/3/forge-parent-3.pom (5.0 kB at 13 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.5.8/plexus-utils-1.5.8.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.5.8/plexus-utils-1.5.8.pom (8.1 kB at 21 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interpolation/1.13/plexus-interpolation-1.13.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interpolation/1.13/plexus-interpolation-1.13.pom (890 B at 2.3 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-components/1.1.15/plexus-components-1.1.15.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-components/1.1.15/plexus-components-1.1.15.pom (2.8 kB at 7.6 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/2.0.3/plexus-2.0.3.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/2.0.3/plexus-2.0.3.pom (15 kB at 40 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-api/2.0.6/maven-plugin-api-2.0.6.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact-manager/2.0.6/maven-artifact-manager-2.0.6.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-profile/2.0.6/maven-profile-2.0.6.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-project/2.0.6/maven-project-2.0.6.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-registry/2.0.6/maven-plugin-registry-2.0.6.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact-manager/2.0.6/maven-artifact-manager-2.0.6.jar (57 kB at 131 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-core/2.0.6/maven-core-2.0.6.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-api/2.0.6/maven-plugin-api-2.0.6.jar (13 kB at 16 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-parameter-documenter/2.0.6/maven-plugin-parameter-documenter-2.0.6.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-registry/2.0.6/maven-plugin-registry-2.0.6.jar (29 kB at 32 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/reporting/maven-reporting-api/2.0.6/maven-reporting-api-2.0.6.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-profile/2.0.6/maven-profile-2.0.6.jar (35 kB at 36 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/doxia/doxia-sink-api/1.0-alpha-7/doxia-sink-api-1.0-alpha-7.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-core/2.0.6/maven-core-2.0.6.jar (152 kB at 146 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-repository-metadata/2.0.6/maven-repository-metadata-2.0.6.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-parameter-documenter/2.0.6/maven-plugin-parameter-documenter-2.0.6.jar (21 kB at 18 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-error-diagnostics/2.0.6/maven-error-diagnostics-2.0.6.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/reporting/maven-reporting-api/2.0.6/maven-reporting-api-2.0.6.jar (9.9 kB at 8.0 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/commons-cli/commons-cli/1.0/commons-cli-1.0.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/doxia/doxia-sink-api/1.0-alpha-7/doxia-sink-api-1.0-alpha-7.jar (5.9 kB at 4.4 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-descriptor/2.0.6/maven-plugin-descriptor-2.0.6.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-project/2.0.6/maven-project-2.0.6.jar (116 kB at 84 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interactivity-api/1.0-alpha-4/plexus-interactivity-api-1.0-alpha-4.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-repository-metadata/2.0.6/maven-repository-metadata-2.0.6.jar (24 kB at 17 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/classworlds/classworlds/1.1/classworlds-1.1.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-error-diagnostics/2.0.6/maven-error-diagnostics-2.0.6.jar (14 kB at 8.8 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact/2.0.6/maven-artifact-2.0.6.jar
Downloaded from central: https://repo.maven.apache.org/maven2/commons-cli/commons-cli/1.0/commons-cli-1.0.jar (30 kB at 19 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-settings/2.0.6/maven-settings-2.0.6.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-descriptor/2.0.6/maven-plugin-descriptor-2.0.6.jar (37 kB at 21 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-model/2.0.6/maven-model-2.0.6.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interactivity-api/1.0-alpha-4/plexus-interactivity-api-1.0-alpha-4.jar (13 kB at 7.6 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-monitor/2.0.6/maven-monitor-2.0.6.jar
Downloaded from central: https://repo.maven.apache.org/maven2/classworlds/classworlds/1.1/classworlds-1.1.jar (38 kB at 21 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-container-default/1.0-alpha-9-stable-1/plexus-container-default-1.0-alpha-9-stable-1.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-settings/2.0.6/maven-settings-2.0.6.jar (49 kB at 25 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/junit/junit/3.8.1/junit-3.8.1.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-monitor/2.0.6/maven-monitor-2.0.6.jar (10 kB at 4.8 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/2.0.5/plexus-utils-2.0.5.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact/2.0.6/maven-artifact-2.0.6.jar (87 kB at 40 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-filtering/1.1/maven-filtering-1.1.jar
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-container-default/1.0-alpha-9-stable-1/plexus-container-default-1.0-alpha-9-stable-1.jar (194 kB at 86 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/sonatype/plexus/plexus-build-api/0.0.4/plexus-build-api-0.0.4.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-model/2.0.6/maven-model-2.0.6.jar (86 kB at 37 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interpolation/1.13/plexus-interpolation-1.13.jar
Downloaded from central: https://repo.maven.apache.org/maven2/junit/junit/3.8.1/junit-3.8.1.jar (121 kB at 49 kB/s)
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-filtering/1.1/maven-filtering-1.1.jar (43 kB at 17 kB/s)
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/sonatype/plexus/plexus-build-api/0.0.4/plexus-build-api-0.0.4.jar (6.8 kB at 2.6 kB/s)
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interpolation/1.13/plexus-interpolation-1.13.jar (61 kB at 23 kB/s)
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/2.0.5/plexus-utils-2.0.5.jar (223 kB at 81 kB/s)
[builder] [WARNING] Using platform encoding (ANSI_X3.4-1968 actually) to copy filtered resources, i.e. build is platform dependent!
[builder] [INFO] skip non existing resourceDirectory /workspace/src/main/resources
[builder] [INFO] 
[builder] [INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ hello ---
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-api/2.0.9/maven-plugin-api-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-api/2.0.9/maven-plugin-api-2.0.9.pom (1.5 kB at 3.8 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven/2.0.9/maven-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven/2.0.9/maven-2.0.9.pom (19 kB at 51 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-parent/8/maven-parent-8.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-parent/8/maven-parent-8.pom (24 kB at 68 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/apache/4/apache-4.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/apache/4/apache-4.pom (4.5 kB at 12 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact/2.0.9/maven-artifact-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact/2.0.9/maven-artifact-2.0.9.pom (1.6 kB at 4.4 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.5.1/plexus-utils-1.5.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.5.1/plexus-utils-1.5.1.pom (2.3 kB at 5.8 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-core/2.0.9/maven-core-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-core/2.0.9/maven-core-2.0.9.pom (7.8 kB at 21 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-settings/2.0.9/maven-settings-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-settings/2.0.9/maven-settings-2.0.9.pom (2.1 kB at 5.1 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-model/2.0.9/maven-model-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-model/2.0.9/maven-model-2.0.9.pom (3.1 kB at 8.2 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-parameter-documenter/2.0.9/maven-plugin-parameter-documenter-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-parameter-documenter/2.0.9/maven-plugin-parameter-documenter-2.0.9.pom (2.0 kB at 5.0 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-profile/2.0.9/maven-profile-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-profile/2.0.9/maven-profile-2.0.9.pom (2.0 kB at 5.3 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-repository-metadata/2.0.9/maven-repository-metadata-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-repository-metadata/2.0.9/maven-repository-metadata-2.0.9.pom (1.9 kB at 5.2 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-error-diagnostics/2.0.9/maven-error-diagnostics-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-error-diagnostics/2.0.9/maven-error-diagnostics-2.0.9.pom (1.7 kB at 4.7 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-project/2.0.9/maven-project-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-project/2.0.9/maven-project-2.0.9.pom (2.7 kB at 7.6 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact-manager/2.0.9/maven-artifact-manager-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact-manager/2.0.9/maven-artifact-manager-2.0.9.pom (2.7 kB at 7.7 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-registry/2.0.9/maven-plugin-registry-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-registry/2.0.9/maven-plugin-registry-2.0.9.pom (2.0 kB at 5.3 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-descriptor/2.0.9/maven-plugin-descriptor-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-descriptor/2.0.9/maven-plugin-descriptor-2.0.9.pom (2.1 kB at 5.3 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-monitor/2.0.9/maven-monitor-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-monitor/2.0.9/maven-monitor-2.0.9.pom (1.3 kB at 3.5 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-toolchain/1.0/maven-toolchain-1.0.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-toolchain/1.0/maven-toolchain-1.0.pom (3.4 kB at 9.6 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-utils/0.1/maven-shared-utils-0.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-utils/0.1/maven-shared-utils-0.1.pom (4.0 kB at 11 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-components/18/maven-shared-components-18.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-components/18/maven-shared-components-18.pom (4.9 kB at 13 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/com/google/code/findbugs/jsr305/2.0.1/jsr305-2.0.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/com/google/code/findbugs/jsr305/2.0.1/jsr305-2.0.1.pom (965 B at 2.6 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-incremental/1.1/maven-shared-incremental-1.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-incremental/1.1/maven-shared-incremental-1.1.pom (4.7 kB at 13 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-components/19/maven-shared-components-19.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-components/19/maven-shared-components-19.pom (6.4 kB at 17 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-api/2.2.1/maven-plugin-api-2.2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-api/2.2.1/maven-plugin-api-2.2.1.pom (1.5 kB at 3.8 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven/2.2.1/maven-2.2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven/2.2.1/maven-2.2.1.pom (22 kB at 57 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-parent/11/maven-parent-11.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-parent/11/maven-parent-11.pom (32 kB at 81 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/apache/5/apache-5.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/apache/5/apache-5.pom (4.1 kB at 11 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-core/2.2.1/maven-core-2.2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-core/2.2.1/maven-core-2.2.1.pom (12 kB at 30 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-settings/2.2.1/maven-settings-2.2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-settings/2.2.1/maven-settings-2.2.1.pom (2.2 kB at 5.7 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-model/2.2.1/maven-model-2.2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-model/2.2.1/maven-model-2.2.1.pom (3.2 kB at 8.6 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interpolation/1.11/plexus-interpolation-1.11.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interpolation/1.11/plexus-interpolation-1.11.pom (889 B at 2.4 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-parameter-documenter/2.2.1/maven-plugin-parameter-documenter-2.2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-parameter-documenter/2.2.1/maven-plugin-parameter-documenter-2.2.1.pom (2.0 kB at 5.2 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/slf4j/slf4j-jdk14/1.5.6/slf4j-jdk14-1.5.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/slf4j/slf4j-jdk14/1.5.6/slf4j-jdk14-1.5.6.pom (1.9 kB at 5.0 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/slf4j/slf4j-parent/1.5.6/slf4j-parent-1.5.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/slf4j/slf4j-parent/1.5.6/slf4j-parent-1.5.6.pom (7.9 kB at 22 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/slf4j/slf4j-api/1.5.6/slf4j-api-1.5.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/slf4j/slf4j-api/1.5.6/slf4j-api-1.5.6.pom (3.0 kB at 8.0 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/slf4j/jcl-over-slf4j/1.5.6/jcl-over-slf4j-1.5.6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/slf4j/jcl-over-slf4j/1.5.6/jcl-over-slf4j-1.5.6.pom (2.2 kB at 6.1 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-profile/2.2.1/maven-profile-2.2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-profile/2.2.1/maven-profile-2.2.1.pom (2.2 kB at 5.9 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact/2.2.1/maven-artifact-2.2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact/2.2.1/maven-artifact-2.2.1.pom (1.6 kB at 4.2 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-repository-metadata/2.2.1/maven-repository-metadata-2.2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-repository-metadata/2.2.1/maven-repository-metadata-2.2.1.pom (1.9 kB at 4.8 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-error-diagnostics/2.2.1/maven-error-diagnostics-2.2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-error-diagnostics/2.2.1/maven-error-diagnostics-2.2.1.pom (1.7 kB at 4.5 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-project/2.2.1/maven-project-2.2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-project/2.2.1/maven-project-2.2.1.pom (2.8 kB at 7.4 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact-manager/2.2.1/maven-artifact-manager-2.2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact-manager/2.2.1/maven-artifact-manager-2.2.1.pom (3.1 kB at 8.2 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/backport-util-concurrent/backport-util-concurrent/3.1/backport-util-concurrent-3.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/backport-util-concurrent/backport-util-concurrent/3.1/backport-util-concurrent-3.1.pom (880 B at 2.3 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-registry/2.2.1/maven-plugin-registry-2.2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-registry/2.2.1/maven-plugin-registry-2.2.1.pom (1.9 kB at 5.0 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-descriptor/2.2.1/maven-plugin-descriptor-2.2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-descriptor/2.2.1/maven-plugin-descriptor-2.2.1.pom (2.1 kB at 5.5 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-monitor/2.2.1/maven-monitor-2.2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-monitor/2.2.1/maven-monitor-2.2.1.pom (1.3 kB at 3.3 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/sonatype/plexus/plexus-sec-dispatcher/1.3/plexus-sec-dispatcher-1.3.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/sonatype/plexus/plexus-sec-dispatcher/1.3/plexus-sec-dispatcher-1.3.pom (3.0 kB at 7.8 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/sonatype/spice/spice-parent/12/spice-parent-12.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/sonatype/spice/spice-parent/12/spice-parent-12.pom (6.8 kB at 18 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/sonatype/forge/forge-parent/4/forge-parent-4.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/sonatype/forge/forge-parent/4/forge-parent-4.pom (8.4 kB at 23 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.5.5/plexus-utils-1.5.5.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.5.5/plexus-utils-1.5.5.pom (5.1 kB at 14 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/sonatype/plexus/plexus-cipher/1.4/plexus-cipher-1.4.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/sonatype/plexus/plexus-cipher/1.4/plexus-cipher-1.4.pom (2.1 kB at 5.4 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-component-annotations/1.5.5/plexus-component-annotations-1.5.5.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-component-annotations/1.5.5/plexus-component-annotations-1.5.5.pom (815 B at 2.1 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-containers/1.5.5/plexus-containers-1.5.5.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-containers/1.5.5/plexus-containers-1.5.5.pom (4.2 kB at 12 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/2.0.7/plexus-2.0.7.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/2.0.7/plexus-2.0.7.pom (17 kB at 48 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compiler-api/2.2/plexus-compiler-api-2.2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compiler-api/2.2/plexus-compiler-api-2.2.pom (865 B at 2.3 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compiler/2.2/plexus-compiler-2.2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compiler/2.2/plexus-compiler-2.2.pom (3.6 kB at 9.6 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-components/1.3.1/plexus-components-1.3.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-components/1.3.1/plexus-components-1.3.1.pom (3.1 kB at 7.9 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/3.3.1/plexus-3.3.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/3.3.1/plexus-3.3.1.pom (20 kB at 52 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/sonatype/spice/spice-parent/17/spice-parent-17.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/sonatype/spice/spice-parent/17/spice-parent-17.pom (6.8 kB at 16 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/sonatype/forge/forge-parent/10/forge-parent-10.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/sonatype/forge/forge-parent/10/forge-parent-10.pom (14 kB at 32 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/3.0.8/plexus-utils-3.0.8.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/3.0.8/plexus-utils-3.0.8.pom (3.1 kB at 8.2 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/3.2/plexus-3.2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/3.2/plexus-3.2.pom (19 kB at 50 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compiler-manager/2.2/plexus-compiler-manager-2.2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compiler-manager/2.2/plexus-compiler-manager-2.2.pom (690 B at 1.8 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compiler-javac/2.2/plexus-compiler-javac-2.2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compiler-javac/2.2/plexus-compiler-javac-2.2.pom (769 B at 2.1 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compilers/2.2/plexus-compilers-2.2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compilers/2.2/plexus-compilers-2.2.pom (1.2 kB at 3.2 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-container-default/1.5.5/plexus-container-default-1.5.5.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-container-default/1.5.5/plexus-container-default-1.5.5.pom (2.8 kB at 7.7 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.4.5/plexus-utils-1.4.5.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.4.5/plexus-utils-1.4.5.pom (2.3 kB at 6.4 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-classworlds/2.2.2/plexus-classworlds-2.2.2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-classworlds/2.2.2/plexus-classworlds-2.2.2.pom (4.0 kB at 11 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/xbean/xbean-reflect/3.4/xbean-reflect-3.4.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/xbean/xbean-reflect/3.4/xbean-reflect-3.4.pom (2.8 kB at 7.5 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/xbean/xbean/3.4/xbean-3.4.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/xbean/xbean/3.4/xbean-3.4.pom (19 kB at 51 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/log4j/log4j/1.2.12/log4j-1.2.12.pom
Downloaded from central: https://repo.maven.apache.org/maven2/log4j/log4j/1.2.12/log4j-1.2.12.pom (145 B at 402 B/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/commons-logging/commons-logging-api/1.1/commons-logging-api-1.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/commons-logging/commons-logging-api/1.1/commons-logging-api-1.1.pom (5.3 kB at 15 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/com/google/collections/google-collections/1.0/google-collections-1.0.pom
Downloaded from central: https://repo.maven.apache.org/maven2/com/google/collections/google-collections/1.0/google-collections-1.0.pom (2.5 kB at 6.8 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/com/google/google/1/google-1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/com/google/google/1/google-1.pom (1.6 kB at 3.9 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/junit/junit/3.8.2/junit-3.8.2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/junit/junit/3.8.2/junit-3.8.2.pom (747 B at 1.9 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-api/2.0.9/maven-plugin-api-2.0.9.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact/2.0.9/maven-artifact-2.0.9.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.5.1/plexus-utils-1.5.1.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-core/2.0.9/maven-core-2.0.9.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-settings/2.0.9/maven-settings-2.0.9.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-settings/2.0.9/maven-settings-2.0.9.jar (49 kB at 139 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-parameter-documenter/2.0.9/maven-plugin-parameter-documenter-2.0.9.jar
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-api/2.0.9/maven-plugin-api-2.0.9.jar (13 kB at 35 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-profile/2.0.9/maven-profile-2.0.9.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact/2.0.9/maven-artifact-2.0.9.jar (89 kB at 214 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-model/2.0.9/maven-model-2.0.9.jar
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/1.5.1/plexus-utils-1.5.1.jar (211 kB at 506 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-repository-metadata/2.0.9/maven-repository-metadata-2.0.9.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-core/2.0.9/maven-core-2.0.9.jar (160 kB at 270 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-error-diagnostics/2.0.9/maven-error-diagnostics-2.0.9.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-parameter-documenter/2.0.9/maven-plugin-parameter-documenter-2.0.9.jar (21 kB at 30 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-project/2.0.9/maven-project-2.0.9.jar
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-profile/2.0.9/maven-profile-2.0.9.jar (35 kB at 48 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-registry/2.0.9/maven-plugin-registry-2.0.9.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-model/2.0.9/maven-model-2.0.9.jar (87 kB at 111 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-descriptor/2.0.9/maven-plugin-descriptor-2.0.9.jar
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-repository-metadata/2.0.9/maven-repository-metadata-2.0.9.jar (25 kB at 31 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact-manager/2.0.9/maven-artifact-manager-2.0.9.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-error-diagnostics/2.0.9/maven-error-diagnostics-2.0.9.jar (14 kB at 14 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-monitor/2.0.9/maven-monitor-2.0.9.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-project/2.0.9/maven-project-2.0.9.jar (122 kB at 118 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-toolchain/1.0/maven-toolchain-1.0.jar
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-registry/2.0.9/maven-plugin-registry-2.0.9.jar (29 kB at 26 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-utils/0.1/maven-shared-utils-0.1.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-plugin-descriptor/2.0.9/maven-plugin-descriptor-2.0.9.jar (37 kB at 32 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/com/google/code/findbugs/jsr305/2.0.1/jsr305-2.0.1.jar
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-artifact-manager/2.0.9/maven-artifact-manager-2.0.9.jar (58 kB at 50 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-incremental/1.1/maven-shared-incremental-1.1.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-monitor/2.0.9/maven-monitor-2.0.9.jar (10 kB at 7.7 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-component-annotations/1.5.5/plexus-component-annotations-1.5.5.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-toolchain/1.0/maven-toolchain-1.0.jar (33 kB at 24 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compiler-api/2.2/plexus-compiler-api-2.2.jar
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/com/google/code/findbugs/jsr305/2.0.1/jsr305-2.0.1.jar (32 kB at 21 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compiler-manager/2.2/plexus-compiler-manager-2.2.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-utils/0.1/maven-shared-utils-0.1.jar (155 kB at 101 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compiler-javac/2.2/plexus-compiler-javac-2.2.jar
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-incremental/1.1/maven-shared-incremental-1.1.jar (14 kB at 8.8 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-container-default/1.5.5/plexus-container-default-1.5.5.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-component-annotations/1.5.5/plexus-component-annotations-1.5.5.jar (4.2 kB at 2.4 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-classworlds/2.2.2/plexus-classworlds-2.2.2.jar
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compiler-api/2.2/plexus-compiler-api-2.2.jar (25 kB at 14 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/xbean/xbean-reflect/3.4/xbean-reflect-3.4.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compiler-manager/2.2/plexus-compiler-manager-2.2.jar (4.6 kB at 2.4 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/log4j/log4j/1.2.12/log4j-1.2.12.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-compiler-javac/2.2/plexus-compiler-javac-2.2.jar (19 kB at 10.0 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/commons-logging/commons-logging-api/1.1/commons-logging-api-1.1.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-container-default/1.5.5/plexus-container-default-1.5.5.jar (217 kB at 112 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/com/google/collections/google-collections/1.0/google-collections-1.0.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/xbean/xbean-reflect/3.4/xbean-reflect-3.4.jar (134 kB at 64 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/junit/junit/3.8.2/junit-3.8.2.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-classworlds/2.2.2/plexus-classworlds-2.2.2.jar (46 kB at 22 kB/s)
Downloaded from central: https://repo.maven.apache.org/maven2/commons-logging/commons-logging-api/1.1/commons-logging-api-1.1.jar (45 kB at 20 kB/s)
Downloaded from central: https://repo.maven.apache.org/maven2/junit/junit/3.8.2/junit-3.8.2.jar (121 kB at 49 kB/s)
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/com/google/collections/google-collections/1.0/google-collections-1.0.jar (640 kB at 256 kB/s)
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/log4j/log4j/1.2.12/log4j-1.2.12.jar (358 kB at 137 kB/s)
[builder] [INFO] Changes detected - recompiling the module!
[builder] [WARNING] File encoding has not been set, using platform encoding ANSI_X3.4-1968, i.e. build is platform dependent!
[builder] [INFO] Compiling 1 source file to /workspace/target/classes
[builder] [INFO] 
[builder] [INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ hello ---
[builder] [INFO] Not copying test resources
[builder] [INFO] 
[builder] [INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ hello ---
[builder] [INFO] Not compiling test sources
[builder] [INFO] 
[builder] [INFO] --- maven-surefire-plugin:2.12.4:test (default-test) @ hello ---
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/surefire-booter/2.12.4/surefire-booter-2.12.4.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/surefire-booter/2.12.4/surefire-booter-2.12.4.pom (3.0 kB at 7.8 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/surefire-api/2.12.4/surefire-api-2.12.4.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/surefire-api/2.12.4/surefire-api-2.12.4.pom (2.5 kB at 6.6 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/maven-surefire-common/2.12.4/maven-surefire-common-2.12.4.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/maven-surefire-common/2.12.4/maven-surefire-common-2.12.4.pom (5.5 kB at 7.6 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugin-tools/maven-plugin-annotations/3.1/maven-plugin-annotations-3.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugin-tools/maven-plugin-annotations/3.1/maven-plugin-annotations-3.1.pom (1.6 kB at 4.2 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugin-tools/maven-plugin-tools/3.1/maven-plugin-tools-3.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugin-tools/maven-plugin-tools/3.1/maven-plugin-tools-3.1.pom (16 kB at 41 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/reporting/maven-reporting-api/2.0.9/maven-reporting-api-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/reporting/maven-reporting-api/2.0.9/maven-reporting-api-2.0.9.pom (1.8 kB at 4.8 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/reporting/maven-reporting/2.0.9/maven-reporting-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/reporting/maven-reporting/2.0.9/maven-reporting-2.0.9.pom (1.5 kB at 3.9 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-toolchain/2.0.9/maven-toolchain-2.0.9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-toolchain/2.0.9/maven-toolchain-2.0.9.pom (3.5 kB at 9.1 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/commons/commons-lang3/3.1/commons-lang3-3.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/commons/commons-lang3/3.1/commons-lang3-3.1.pom (17 kB at 44 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/commons/commons-parent/22/commons-parent-22.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/commons/commons-parent/22/commons-parent-22.pom (42 kB at 107 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/apache/9/apache-9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/apache/9/apache-9.pom (15 kB at 37 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-common-artifact-filters/1.3/maven-common-artifact-filters-1.3.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-common-artifact-filters/1.3/maven-common-artifact-filters-1.3.pom (3.7 kB at 8.8 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-components/12/maven-shared-components-12.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-shared-components/12/maven-shared-components-12.pom (9.3 kB at 23 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-parent/13/maven-parent-13.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-parent/13/maven-parent-13.pom (23 kB at 57 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/apache/6/apache-6.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/apache/6/apache-6.pom (13 kB at 32 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-container-default/1.0-alpha-9/plexus-container-default-1.0-alpha-9.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-container-default/1.0-alpha-9/plexus-container-default-1.0-alpha-9.pom (1.2 kB at 3.1 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/surefire-booter/2.12.4/surefire-booter-2.12.4.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/surefire-api/2.12.4/surefire-api-2.12.4.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/maven-surefire-common/2.12.4/maven-surefire-common-2.12.4.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/commons/commons-lang3/3.1/commons-lang3-3.1.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-common-artifact-filters/1.3/maven-common-artifact-filters-1.3.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/maven-surefire-common/2.12.4/maven-surefire-common-2.12.4.jar (263 kB at 695 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/3.0.8/plexus-utils-3.0.8.jar
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/surefire-booter/2.12.4/surefire-booter-2.12.4.jar (35 kB at 82 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/reporting/maven-reporting-api/2.0.9/maven-reporting-api-2.0.9.jar
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/surefire-api/2.12.4/surefire-api-2.12.4.jar (118 kB at 280 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-toolchain/2.0.9/maven-toolchain-2.0.9.jar
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/shared/maven-common-artifact-filters/1.3/maven-common-artifact-filters-1.3.jar (31 kB at 74 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugin-tools/maven-plugin-annotations/3.1/maven-plugin-annotations-3.1.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/commons/commons-lang3/3.1/commons-lang3-3.1.jar (316 kB at 508 kB/s)
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/3.0.8/plexus-utils-3.0.8.jar (232 kB at 305 kB/s)
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-toolchain/2.0.9/maven-toolchain-2.0.9.jar (38 kB at 47 kB/s)
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/reporting/maven-reporting-api/2.0.9/maven-reporting-api-2.0.9.jar (10 kB at 12 kB/s)
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugin-tools/maven-plugin-annotations/3.1/maven-plugin-annotations-3.1.jar (14 kB at 17 kB/s)
[builder] [INFO] Tests are skipped.
[builder] [INFO] 
[builder] [INFO] --- maven-jar-plugin:2.4:jar (default-jar) @ hello ---
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-archiver/2.5/maven-archiver-2.5.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-archiver/2.5/maven-archiver-2.5.pom (4.5 kB at 11 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-archiver/2.1/plexus-archiver-2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-archiver/2.1/plexus-archiver-2.1.pom (2.8 kB at 6.9 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/3.0/plexus-utils-3.0.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/3.0/plexus-utils-3.0.pom (4.1 kB at 10 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/sonatype/spice/spice-parent/16/spice-parent-16.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/sonatype/spice/spice-parent/16/spice-parent-16.pom (8.4 kB at 21 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/sonatype/forge/forge-parent/5/forge-parent-5.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/sonatype/forge/forge-parent/5/forge-parent-5.pom (8.4 kB at 21 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-io/2.0.2/plexus-io-2.0.2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-io/2.0.2/plexus-io-2.0.2.pom (1.7 kB at 4.4 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-components/1.1.19/plexus-components-1.1.19.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-components/1.1.19/plexus-components-1.1.19.pom (2.7 kB at 6.9 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/3.0.1/plexus-3.0.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus/3.0.1/plexus-3.0.1.pom (19 kB at 48 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interpolation/1.15/plexus-interpolation-1.15.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interpolation/1.15/plexus-interpolation-1.15.pom (1.0 kB at 2.7 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/commons-lang/commons-lang/2.1/commons-lang-2.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/commons-lang/commons-lang/2.1/commons-lang-2.1.pom (9.9 kB at 25 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/classworlds/classworlds/1.1-alpha-2/classworlds-1.1-alpha-2.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-archiver/2.5/maven-archiver-2.5.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interpolation/1.15/plexus-interpolation-1.15.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-archiver/2.1/plexus-archiver-2.1.jar
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-io/2.0.2/plexus-io-2.0.2.jar
Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-archiver/2.1/plexus-archiver-2.1.jar (184 kB at 468 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/commons-lang/commons-lang/2.1/commons-lang-2.1.jar
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-interpolation/1.15/plexus-interpolation-1.15.jar (60 kB at 152 kB/s)
[builder] Downloading from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/3.0/plexus-utils-3.0.jar
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/classworlds/classworlds/1.1-alpha-2/classworlds-1.1-alpha-2.jar (38 kB at 88 kB/s)
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/maven-archiver/2.5/maven-archiver-2.5.jar (22 kB at 49 kB/s)
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-io/2.0.2/plexus-io-2.0.2.jar (58 kB at 126 kB/s)
Downloaded from central: https://repo.maven.apache.org/maven2/commons-lang/commons-lang/2.1/commons-lang-2.1.jar (208 kB at 260 kB/s)
[builder] Downloaded from central: https://repo.maven.apache.org/maven2/org/codehaus/plexus/plexus-utils/3.0/plexus-utils-3.0.jar (226 kB at 282 kB/s)
[builder] [INFO] Building jar: /workspace/target/hello-1.0.0.jar
[builder] [INFO] ------------------------------------------------------------------------
[builder] [INFO] BUILD SUCCESS
[builder] [INFO] ------------------------------------------------------------------------
[builder] [INFO] Total time:  01:17 min
[builder] [INFO] Finished at: 2019-12-30T09:21:18Z
[builder] [INFO] ------------------------------------------------------------------------
[builder]   Removing source code
[builder] 
[builder] Java Function Buildpack 0.2.0-BUILD-SNAPSHOT
[builder]   Java functions.Hello: Contributing to layer
[builder]     Writing SPRING_CLOUD_FUNCTION_FUNCTION_CLASS to launch
[builder]     Writing SPRING_CLOUD_FUNCTION_LOCATION to launch
[builder]   riff Streaming HTTP Adapter 0.0.1: Contributing to layer
[builder]     Reusing cached download from buildpack
[builder]     Expanding to /layers/io.projectriff.java/streaming-http-adapter
[builder]   riff Java Invoker 0.2.0+snapshot: Contributing to layer
[builder]     Reusing cached download from buildpack
[builder]     Expanding to /layers/io.projectriff.java/riff-invoker-java
[builder]   Process types:
[builder]     function:           streaming-http-adapter java -cp /layers/io.projectriff.java/riff-invoker-java $JAVA_OPTS org.springframework.boot.loader.JarLauncher
[builder]     streaming-function: java -cp /layers/io.projectriff.java/riff-invoker-java $JAVA_OPTS org.springframework.boot.loader.JarLauncher
[builder]     web:                streaming-http-adapter java -cp /layers/io.projectriff.java/riff-invoker-java $JAVA_OPTS org.springframework.boot.loader.JarLauncher
===> EXPORTING
[exporter] Adding layer 'app'
[exporter] Adding layer 'config'
[exporter] Adding layer 'launcher'
[exporter] Adding layer 'org.cloudfoundry.openjdk:openjdk-jre'
[exporter] Adding layer 'io.projectriff.java:java-function'
[exporter] Adding layer 'io.projectriff.java:riff-invoker-java'
[exporter] Adding layer 'io.projectriff.java:streaming-http-adapter'
[exporter] *** Images (e046566d1e88):
[exporter]       index.docker.io/your-org/my-function:latest
===> CACHING
[cacher] Caching layer 'org.cloudfoundry.openjdk:openjdk-jdk'
[cacher] Caching layer 'org.cloudfoundry.buildsystem:build-system-cache'
[cacher] Caching layer 'org.cloudfoundry.buildsystem:maven'
Successfully built image your-org/my-function
[root@izttsn6fen661yz java-hello]#
```



运行该镜像，并进行请求：

```
 curl localhost:8080 -H 'Content-Type: text/plain' -w '\n' -d hello
```

可以发现生成的镜像对外暴露的是GRPC的接口：

```
[root@izttsn6fen661yz java-hello]# docker run -p 8080:8080 your-org/my-function 

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.2.1.RELEASE)

09:44:20.103 [main] INFO  i.p.invoker.main.EntryPoint - Starting EntryPoint on e5c94695a4f0 with PID 10 (/layers/io.projectriff.java/riff-invoker-java/BOOT-INF/classes started by cnb in /workspace)
09:44:20.111 [main] INFO  i.p.invoker.main.EntryPoint - No active profile set, falling back to default profiles: default
09:44:22.782 [main] INFO  o.s.c.f.d.FunctionDeployerConfiguration - Deploying archive: /workspace
09:44:22.796 [main] INFO  o.s.c.f.d.FunctionArchiveDeployer - Registering function class 'class functions.Hello' of type 'java.util.function.Function<java.lang.String, java.lang.String>' under name 'hello'.
09:44:22.800 [main] INFO  o.s.c.f.d.FunctionDeployerConfiguration - Successfully deployed archive: /workspace
09:44:23.001 [main] INFO  i.p.invoker.main.EntryPoint - Started EntryPoint in 3.733 seconds (JVM running for 5.33)
09:44:33.524 [grpc-default-executor-0] INFO  o.s.c.f.c.c.BeanFactoryAwareFunctionRegistry - Looking up function 'null' with acceptedOutputTypes: [*/*]
io.grpc.StatusException: UNKNOWN: The mapper returned a null value.
	at io.grpc.Status.asException(Status.java:541)
	at io.projectriff.invoker.server.GrpcServerAdapter.lambda$invoke$0(GrpcServerAdapter.java:70)
	at reactor.core.publisher.Flux.lambda$onErrorMap$28(Flux.java:6390)
	at reactor.core.publisher.FluxOnErrorResume$ResumeSubscriber.onError(FluxOnErrorResume.java:88)
	at reactor.core.publisher.FluxMap$MapSubscriber.onError(FluxMap.java:126)
	at reactor.core.publisher.MonoFlatMapMany$FlatMapManyInner.onError(MonoFlatMapMany.java:247)
	at reactor.core.publisher.FluxMap$MapSubscriber.onError(FluxMap.java:126)
	at reactor.core.publisher.FluxMap$MapSubscriber.onError(FluxMap.java:126)
	at reactor.core.publisher.FluxMap$MapSubscriber.onNext(FluxMap.java:106)
	at reactor.core.publisher.FluxMap$MapSubscriber.onNext(FluxMap.java:114)
	at reactor.core.publisher.FluxMap$MapSubscriber.onNext(FluxMap.java:114)
	at reactor.core.publisher.FluxSkip$SkipSubscriber.onNext(FluxSkip.java:80)
	at reactor.core.publisher.FluxGroupBy$UnicastGroupedFlux.drainRegular(FluxGroupBy.java:554)
	at reactor.core.publisher.FluxGroupBy$UnicastGroupedFlux.drain(FluxGroupBy.java:630)
	at reactor.core.publisher.FluxGroupBy$UnicastGroupedFlux.onNext(FluxGroupBy.java:670)
	at reactor.core.publisher.FluxGroupBy$GroupByMain.onNext(FluxGroupBy.java:205)
	at io.projectriff.invoker.server.GrpcServerAdapter$1.onNext(GrpcServerAdapter.java:176)
	at io.projectriff.invoker.server.GrpcServerAdapter$1.onNext(GrpcServerAdapter.java:154)
	at reactor.core.publisher.FluxConcatArray$ConcatArraySubscriber.onNext(FluxConcatArray.java:176)
	at reactor.core.publisher.FluxMap$MapSubscriber.onNext(FluxMap.java:114)
	at reactor.core.publisher.FluxSkip$SkipSubscriber.onNext(FluxSkip.java:80)
	at reactor.core.publisher.FluxSwitchOnFirst$AbstractSwitchOnFirstInner.onNext(FluxSwitchOnFirst.java:184)
	at com.salesforce.reactivegrpc.common.AbstractStreamObserverAndPublisher.drainRegular(AbstractStreamObserverAndPublisher.java:191)
	at com.salesforce.reactivegrpc.common.AbstractStreamObserverAndPublisher.drain(AbstractStreamObserverAndPublisher.java:271)
	at com.salesforce.reactivegrpc.common.AbstractStreamObserverAndPublisher.onNext(AbstractStreamObserverAndPublisher.java:315)
	at io.grpc.stub.ServerCalls$StreamingServerCallHandler$StreamingServerCallListener.onMessage(ServerCalls.java:251)
	at io.grpc.internal.ServerCallImpl$ServerStreamListenerImpl.messagesAvailableInternal(ServerCallImpl.java:309)
	at io.grpc.internal.ServerCallImpl$ServerStreamListenerImpl.messagesAvailable(ServerCallImpl.java:292)
	at io.grpc.internal.ServerImpl$JumpToApplicationThreadServerStreamListener$1MessagesAvailable.runInContext(ServerImpl.java:779)
	at io.grpc.internal.ContextRunnable.run(ContextRunnable.java:37)
	at io.grpc.internal.SerializingExecutor.run(SerializingExecutor.java:123)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(Unknown Source)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(Unknown Source)
	at java.base/java.lang.Thread.run(Unknown Source)
Caused by: java.lang.NullPointerException: The mapper returned a null value.
	at java.base/java.util.Objects.requireNonNull(Unknown Source)
	at reactor.core.publisher.FluxMap$MapSubscriber.onNext(FluxMap.java:100)
	... 25 more
10:08:32.957 [grpc-default-executor-1] INFO  o.s.c.f.c.c.BeanFactoryAwareFunctionRegistry - Looking up function 'null' with acceptedOutputTypes: [application/octet-stream]
io.grpc.StatusException: UNKNOWN: The mapper returned a null value.
	at io.grpc.Status.asException(Status.java:541)
	at io.projectriff.invoker.server.GrpcServerAdapter.lambda$invoke$0(GrpcServerAdapter.java:70)
	at reactor.core.publisher.Flux.lambda$onErrorMap$28(Flux.java:6390)
	at reactor.core.publisher.FluxOnErrorResume$ResumeSubscriber.onError(FluxOnErrorResume.java:88)
	at reactor.core.publisher.FluxMap$MapSubscriber.onError(FluxMap.java:126)
	at reactor.core.publisher.MonoFlatMapMany$FlatMapManyInner.onError(MonoFlatMapMany.java:247)
	at reactor.core.publisher.FluxMap$MapSubscriber.onError(FluxMap.java:126)
	at reactor.core.publisher.FluxMap$MapSubscriber.onError(FluxMap.java:126)
	at reactor.core.publisher.FluxMap$MapSubscriber.onError(FluxMap.java:126)
	at reactor.core.publisher.FluxMap$MapSubscriber.onError(FluxMap.java:126)
	at reactor.core.publisher.FluxMap$MapSubscriber.onNext(FluxMap.java:106)
	at reactor.core.publisher.FluxSkip$SkipSubscriber.onNext(FluxSkip.java:80)
	at reactor.core.publisher.FluxGroupBy$UnicastGroupedFlux.drainRegular(FluxGroupBy.java:554)
	at reactor.core.publisher.FluxGroupBy$UnicastGroupedFlux.drain(FluxGroupBy.java:630)
	at reactor.core.publisher.FluxGroupBy$UnicastGroupedFlux.onNext(FluxGroupBy.java:670)
	at reactor.core.publisher.FluxGroupBy$GroupByMain.onNext(FluxGroupBy.java:205)
	at io.projectriff.invoker.server.GrpcServerAdapter$1.onNext(GrpcServerAdapter.java:176)
	at io.projectriff.invoker.server.GrpcServerAdapter$1.onNext(GrpcServerAdapter.java:154)
	at reactor.core.publisher.FluxConcatArray$ConcatArraySubscriber.onNext(FluxConcatArray.java:176)
	at reactor.core.publisher.FluxMap$MapSubscriber.onNext(FluxMap.java:114)
	at reactor.core.publisher.FluxSkip$SkipSubscriber.onNext(FluxSkip.java:80)
	at reactor.core.publisher.FluxSwitchOnFirst$AbstractSwitchOnFirstInner.onNext(FluxSwitchOnFirst.java:184)
	at com.salesforce.reactivegrpc.common.AbstractStreamObserverAndPublisher.drainRegular(AbstractStreamObserverAndPublisher.java:191)
	at com.salesforce.reactivegrpc.common.AbstractStreamObserverAndPublisher.drain(AbstractStreamObserverAndPublisher.java:271)
	at com.salesforce.reactivegrpc.common.AbstractStreamObserverAndPublisher.onNext(AbstractStreamObserverAndPublisher.java:315)
	at io.grpc.stub.ServerCalls$StreamingServerCallHandler$StreamingServerCallListener.onMessage(ServerCalls.java:251)
	at io.grpc.internal.ServerCallImpl$ServerStreamListenerImpl.messagesAvailableInternal(ServerCallImpl.java:309)
	at io.grpc.internal.ServerCallImpl$ServerStreamListenerImpl.messagesAvailable(ServerCallImpl.java:292)
	at io.grpc.internal.ServerImpl$JumpToApplicationThreadServerStreamListener$1MessagesAvailable.runInContext(ServerImpl.java:779)
	at io.grpc.internal.ContextRunnable.run(ContextRunnable.java:37)
	at io.grpc.internal.SerializingExecutor.run(SerializingExecutor.java:123)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(Unknown Source)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(Unknown Source)
	at java.base/java.lang.Thread.run(Unknown Source)
Caused by: java.lang.NullPointerException: The mapper returned a null value.
	at java.base/java.util.Objects.requireNonNull(Unknown Source)
	at reactor.core.publisher.FluxMap$MapSubscriber.onNext(FluxMap.java:100)
	... 23 more
```



## 安装riff

helm安装文件：https://github.com/projectriff/charts

```
[root@izttsn6fen661yz charts]# make
mkdir -p repository
mkdir -p uncharted
./charts/package.sh cert-manager 0.5.0-snapshot
Successfully packaged chart and saved it to: repository/cert-manager-0.5.0-snapshot.tgz
./charts/unpackage.sh cert-manager
./charts/fetch-istio.sh istio 1.3.3
./charts/package.sh istio 0.5.0-snapshot
Successfully packaged chart and saved it to: repository/istio-0.5.0-snapshot.tgz
./charts/unpackage.sh istio
patching file /root/projectriff/charts/uncharted/istio.yaml
./charts/fetch-kafka.sh kafka 0.20.5
.........................................................
./charts/package.sh riff 0.5.0-snapshot
Successfully packaged chart and saved it to: repository/riff-0.5.0-snapshot.tgz
[root@izttsn6fen661yz charts]# ll
total 48
drwxr-xr-x 14 root root  4096 Jan  1 13:55 build
drwxr-xr-x 14 root root  4096 Jan  1 09:55 charts
-rw-r--r--  1 root root 11359 Dec 31 17:16 LICENSE
-rw-r--r--  1 root root  1658 Jan  1 11:04 Makefile
drwxr-xr-x  2 root root  4096 Dec 31 17:16 overlays
-rw-r--r--  1 root root  6241 Dec 31 17:16 README.md
drwxr-xr-x  2 root root  4096 Jan  1 13:55 repository
drwxr-xr-x  2 root root  4096 Jan  1 13:55 uncharted
-rw-r--r--  1 root root    14 Dec 31 17:16 VERSION
[root@izttsn6fen661yz charts]# 
[root@izttsn6fen661yz charts]# 
[root@izttsn6fen661yz charts]# cd repository/
[root@izttsn6fen661yz repository]# ll
total 452
-rw-r--r-- 1 root root  58478 Jan  1 13:52 cert-manager-0.5.0-snapshot.tgz
-rw-r--r-- 1 root root  78477 Jan  1 13:52 istio-0.5.0-snapshot.tgz
-rw-r--r-- 1 root root  25530 Jan  1 13:53 kafka-0.5.0-snapshot.tgz
-rw-r--r-- 1 root root  59422 Jan  1 13:53 keda-0.5.0-snapshot.tgz
-rw-r--r-- 1 root root  12264 Jan  1 13:54 knative-0.5.0-snapshot.tgz
-rw-r--r-- 1 root root   1867 Jan  1 13:54 kpack-0.5.0-snapshot.tgz
-rw-r--r-- 1 root root 164985 Jan  1 13:55 riff-0.5.0-snapshot.tgz
-rw-r--r-- 1 root root   3615 Jan  1 13:55 riff-build-0.5.0-snapshot.tgz
-rw-r--r-- 1 root root    680 Jan  1 13:54 riff-builders-0.5.0-snapshot.tgz
-rw-r--r-- 1 root root   9971 Jan  1 13:55 riff-core-runtime-0.5.0-snapshot.tgz
-rw-r--r-- 1 root root  10288 Jan  1 13:55 riff-knative-runtime-0.5.0-snapshot.tgz
-rw-r--r-- 1 root root  11484 Jan  1 13:55 riff-streaming-runtime-0.5.0-snapshot.tgz
```

以上安装成功需安装依赖：

https://github.com/k14s/ytt

https://github.com/mikefarah/yq

https://github.com/projectriff/k8s-manifest-scanner

使用helm(需使用helm2，helm3会有报错)生成安装yaml文件：

```bash
[root@izttsn6fen661yz charts]# cd build/
[root@izttsn6fen661yz build]# ll
total 48
drwxr-xr-x 3 root root 4096 Jan  1 13:52 cert-manager
drwxr-xr-x 6 root root 4096 Jan  1 13:52 istio
drwxr-xr-x 4 root root 4096 Jan  1 13:53 kafka
drwxr-xr-x 3 root root 4096 Jan  1 13:53 keda
drwxr-xr-x 3 root root 4096 Jan  1 13:54 knative
drwxr-xr-x 3 root root 4096 Jan  1 13:54 kpack
drwxr-xr-x 4 root root 4096 Jan  1 13:55 riff
drwxr-xr-x 3 root root 4096 Jan  1 13:55 riff-build
drwxr-xr-x 3 root root 4096 Jan  1 13:54 riff-builders
drwxr-xr-x 3 root root 4096 Jan  1 13:55 riff-core-runtime
drwxr-xr-x 3 root root 4096 Jan  1 13:55 riff-knative-runtime
drwxr-xr-x 3 root root 4096 Jan  1 13:55 riff-streaming-runtime
[root@izttsn6fen661yz build]# helm template ./riff --set tags.core-runtime=true --set tags.knative-runtime=true --set tags.streaming-runtime=false --set knative.enabled=false  > riff.yaml
```

### 

helm安装文件：https://github.com/projectriff/charts



# 测试使用

## 简单使用

## 

创建默认的镜像仓库及认证：

如使用dockerhub官方仓库，命令如下

```
riff credential apply my-creds --docker-hub $DOCKER_ID --set-default-image-prefix
```

如使用私有仓库

```
riff credential apply my-creds --registry http://dockerhub.icbc:5000 -registry-user admin --default-image-prefix dockerhub.icbc:5000 
```

之后输入仓库的用户密码就可以了。



