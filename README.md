# System Info of JVM

This is sample project to display the CPU/Memory resources for JVM of Tomcat.


## Sample

```
docker run -v $PWD:/usr/local/tomcat/webapps/system-info -p 8080:8080 tomcat:9-jre8
```


本系列文章记录了企业客户在应用Kubernetes时的一些常见问题

第一篇：Java应用资源限制的迷思

第二篇：利用LXCFS提升容器资源可见性

第三篇：解决服务依赖

随着容器技术的成熟，越来越多的企业客户在企业中选择Docker和Kubernetes作为应用平台的基础。然而在实践过程中，还会遇到很多具体问题。本系列文章会记录阿里云容器服务团队在支持客户中的一些心得体会和最佳实践。我们也欢迎您通过邮件和钉钉群和我们联系，分享您的思路和遇到的问题。

在对Java应用容器化部署的过程中，有些同学反映：自己设置了容器的资源限制，但是Java应用容器在运行中还是会莫名奇妙地被OOM Killer干掉。

这背后一个非常常见的原因是：没有正确设置容器的资源限制以及对应的JVM的堆空间大小。

我们拿一个tomcat应用为例，其实例代码和Kubernetes部署文件可以从Github中获得。

git clone https://github.com/denverdino/system-info
cd system-info`

下面是一个Kubernetes的Pod的定义描述：

Pod中的app是一个初始化容器，负责把一个JSP应用拷贝到 tomcat 容器的 “webapps”目录下。注： 镜像中JSP应用index.jsp用于显示JVM和系统资源信息。
tomcat 容器会保持运行，而且我们限制了容器最大的内存用量为256MB内存。
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  initContainers:
  - image: registry.cn-hangzhou.aliyuncs.com/denverdino/system-info
    name: app
    imagePullPolicy: IfNotPresent
    command:
      - "cp"
      - "-r"
      - "/system-info"
      - "/app"
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: tomcat:9-jre8
    name: tomcat
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - mountPath: /usr/local/tomcat/webapps
      name: app-volume
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "256Mi"
        cpu: "500m"
      limits:
        memory: "256Mi"
        cpu: "500m"
  volumes:
  - name: app-volume
    emptyDir: {}
我们执行如下命令来部署、测试应用

$ kubectl create -f test.yaml
pod "test" created
$ kubectl get pods test
NAME      READY     STATUS    RESTARTS   AGE
test      1/1       Running   0          28s
$ kubectl exec test curl http://localhost:8080/system-info/

...
我们可以看到HTML格式的系统CPU/Memory等信息，我们也可以用 html2text 命令将其转化成为文本格式。

注意：本文是在一个 2C 4G的节点上进行的测试，在不同环境中测试输出的结果会有所不同


$ kubectl exec test curl http://localhost:8080/system-info/ | html2text

Java version     Oracle Corporation 1.8.0_162
Operating system Linux 4.9.64
Server           Apache Tomcat/9.0.6
Memory           Used 29 of 57 MB, Max 878 MB
Physica Memory   3951 MB
CPU Cores        2
                                          **** Memory MXBean ****
Heap Memory Usage     init = 65011712(63488K) used = 19873704(19407K) committed
                      = 65536000(64000K) max = 921174016(899584K)
Non-Heap Memory Usage init = 2555904(2496K) used = 32944912(32172K) committed =
                      33882112(33088K) max = -1(-1K)
                      
我们可以发现，容器中看到的系统内存是 3951MB，而JVM Heap Size最大是 878MB。
纳尼？！我们不是设置容器资源的容量为256MB了吗？如果这样，当应用内存的用量超出了256MB，JVM还没对其进行GC，而JVM进程就会被系统直接OOM干掉了。

问题的根源在于：

对于JVM而言，如果没有设置Heap Size，就会按照宿主机环境的内存大小缺省设置自己的最大堆大小。
Docker容器利用CGroup对进程使用的资源进行限制，而在容器中的JVM依然会利用宿主机环境的内存大小和CPU核数进行缺省设置，这导致了JVM Heap的错误计算。
类似，JVM缺省的GC、JIT编译线程数量取决于宿主机CPU核数。如果我们在一个节点上运行多个Java应用，即使我们设置了CPU的限制，应用之间依然有可能因为GC线程抢占切换，导致应用性能收到影响。

了解了问题的根源，我们就可以非常简单地解决问题了

解决思路：开启CGroup资源感知
Java社区也关注到这个问题，并在JavaSE8u131+和JDK9 支持了对容器资源限制的自动感知能力 https://blogs.oracle.com/java-platform-group/java-se-support-for-docker-cpu-and-memory-limits

其用法就是添加如下参数

java -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap …
我们在上文示例的tomcat容器添加环境变量 “JAVA_OPTS”参数


apiVersion: v1
kind: Pod
metadata:
  name: cgrouptest
spec:
  initContainers:
  - image: registry.cn-hangzhou.aliyuncs.com/denverdino/system-info
    name: app
    imagePullPolicy: IfNotPresent
    command:
      - "cp"
      - "-r"
      - "/system-info"
      - "/app"
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: tomcat:9-jre8
    name: tomcat
    imagePullPolicy: IfNotPresent
    env:
    - name: JAVA_OPTS
      value: "-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap"
    volumeMounts:
    - mountPath: /usr/local/tomcat/webapps
      name: app-volume
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "256Mi"
        cpu: "500m"
      limits:
        memory: "256Mi"
        cpu: "500m"
  volumes:
  - name: app-volume
    emptyDir: {}
我们部署一个新的Pod，并重复相应的测试

$ kubectl create -f cgroup_test.yaml
pod "cgrouptest" created

$ kubectl exec cgrouptest curl http://localhost:8080/system-info/ | html2txt
Java version     Oracle Corporation 1.8.0_162
Operating system Linux 4.9.64
Server           Apache Tomcat/9.0.6
Memory           Used 23 of 44 MB, Max 112 MB
Physica Memory   3951 MB
CPU Cores        2
                                          **** Memory MXBean ****
Heap Memory Usage     init = 8388608(8192K) used = 25280928(24688K) committed =
                      46661632(45568K) max = 117440512(114688K)
Non-Heap Memory Usage init = 2555904(2496K) used = 31970840(31221K) committed =
                      32768000(32000K) max = -1(-1K)


我们看到JVM最大的Heap大小变成了112MB，这很不错，这样就能保证我们的应用不会轻易被OOM了。
随后问题又来了，为什么我们设置了容器最大内存限制是256MB，而JVM只给Heap设置了112MB的最大值呢？

这就涉及到JVM的内存管理的细节了，JVM中的内存消耗包含Heap和Non-Heap两类；类似Class的元信息，JIT编译过的代码，线程堆栈(thread stack)，GC需要的内存空间等都属于Non-Heap内存，所以JVM还会根据CGroup的资源限制预留出部分内存给Non Heap，来保障系统的稳定。（在上面的示例中我们可以看到，tomcat启动后Non Heap占用了近32MB的内存）

在最新的JDK 10中，又对JVM在容器中运行做了进一步的优化和增强。

容器内部感知CGroup资源限制
如果无法利用JDK 8/9的新特性，比如还在使用JDK6的老应用，我们还可以在容器内部利用脚本来获取容器的CGroup资源限制，并通过设置JVM的Heap大小。

Docker1.7开始将容器cgroup信息挂载到容器中，所以应用可以从 /sys/fs/cgroup/memory/memory.limit_in_bytes 等文件获取内存、 CPU等设置，在容器的应用启动命令中根据Cgroup配置正确的资源设置 -Xmx, -XX:ParallelGCThreads等参数

在 https://yq.aliyun.com/articles/18037 一文中已经有相应的示例和代码，本文不再赘述

总结
本文分析了Java应用在容器使用中一个常见Heap设置的问题。容器与虚拟机不同，其资源限制通过CGroup来实现。而容器内部进程如果不感知CGroup的限制，就进行内存、CPU分配可能导致资源冲突和问题。

我们可以非常简单地利用JVM的新特性和自定义脚本来正确设置资源限制。这个可以解决绝大多数资源限制的问题。
