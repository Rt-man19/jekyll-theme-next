**什么是Pod?**

Pod 是kubernetes中的原子调度单位,这意味着kubernetes项目的调度器是统一按照Pod而非容器的资源需求进行计算的。

### Pod的原理

首先关于Pod最重要的一个事实; 它只是一个逻辑概念.
也就是说Kubernetes真正处理的还是宿主机操作系统上的linux容器和Namespace,Cgroups，并不存在一个所谓的Pod边界或者隔离环境。

**Pod是怎么被创建出来的呢?**

Pod,其实是一组共享了某些资源的容器,具体的说Pod里的所有容器共享的是同一个Network Namespace,并且可以声明共享同一个Volume

**问题: 一个有A、B两个容器的Pod,不就等同于一个容器(容器A)共享另外一个容器(容器B)的网络和Volume的玩法吗?**

这好像通过docker run --net --volumes-from这样的命令能实现，比如:
````shell
$ docker run --net=B --volumes-from=B --name=A image-A ...
````

这样确实可行，但是如果真的这样做的话容器B就必须先于容器A启动，这样一个Pod里的多个容器就不是对等关系，而是拓扑关系了.
所以在kubernetes中Pod的实现需要使用一个中间容器,这个容器叫做Infra容器,在这个Pod中,Infra容器永远是第一个被创建的容器,而其他用户定义的容器则通过Join Network Namespace的方式,与Infra容器关联在一起。这样的组织关系可以用下面的图来表达:
![image](928D477C43274F44A705387B2C5DA516)

如上所示,这个Pod里有两个容器A和容器B,还有一个Infra容器.很容易理解，在Kubernetes项目里，Infra容器一定要占用极少的资源，所以它使用的是一个非常特殊的镜像，叫做k8s.gcr.io/pause 这个镜像是一个用汇编语言编写的、永远处于"暂停状态"的容器,解压后大小也只有100~200KB左右

在Infra容器"hold住"Network Namespace后,用户容器就可以加入到Infra容器的Network NameSpace当中了,所以查看这些容器的宿主机上的Namespace文件，他们指向的值一定是完全一样的.

这也就意味着,对于Pod里的容器A和容器B来说:
- 他们可以直接使用localhost进行通信
- 他们看到的网络设备跟Infra容器看到的一样
- 一个Pod只有一个IP地址,也就是这个Pod的Network Namespace对应的IP地址
- 当然其他的所有网络资源都是一个Pod一份，并且被该Pod中的所有容器共享
- Pod的生命周期只跟Infra容器一致,而与容器A和容器B无关

而对于同一个Pod里面的所有用户容器来说,他们进出流量也可以认为都是通过Infra容器来完成的,这一点很重要,因为将来如果要为Kubernetes开发一个网络插件时,应该重点考虑的是如何分配这个Pod的Network Namespace而不是每一个用户容器如何使用网络配置,这是没有意义的
这就意味着,如果你的网络插件需要在容器里安装某些包或者配置才能完成的话,是不可取的: Infra容器镜像的rootfs几乎什么都没有。这也同时意味着网络插件完全不必关心用户容器是否启动，只需要关注如何配置pod,也就是Infra容器的Network Namespace即可。

** 案例: Pod 共享Volume **
````yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
    hostPath:
      path: /data
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
     - name: shared-data
       mountPath: /pod-data
     command: ["/bin/sh"]
     args: ["-c","echo Hello from the debian container > /pod-data/index.html"]
````

**案例1: War包与web服务器**

需求: 有一个Java web应用的War包需要被放在Tomcat的webapps目录下运行起来

假如只用docker来实现这个案例有以下几种解决方案: 
1. 把War包直接放在Tomcat镜像的webapps目录下做成一个新的镜像运行起来。但是这样做，当需要更新War包的内容或者升级tomcat镜像时，都需要重新做一个新的发布镜像，麻烦。
2. 不管War包，永远只发布一个Tomcat容器，不过这个容器的webapps目录必须声明一个hostPath类型的volume,从而把宿主机上的War包挂载到Tomcat容器里运行起来。这样就产生了一个新的问题: 如何让每一台宿主机都预先准备好这个存储有War包的目录?这样看来，只能独立维护一套分布式存储系统了。

***用pod的方式解决***

有了Pod之后，可以把War包和Tomcat分别做成镜像,然后把他们作为一个Pod的两个容器组合在一起
````yaml
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: someimage_withWartag
    name: war
    command: ["cp", "/sample.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: tomcat:7.0
    name: tomcat
    command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001
  volumes:
  - name: app-volume
    emptyDir: {}
````
这个例子中 War包不再是一个普通的容器,而是一个**Init Container**类型的容器

在Pod中,所有InitContainer定义的容器都会比spec.containers定义的用户容器先启动,并且,InitContainer容器会按照顺序逐一启动,而直到他们都启动并且退出了才会启动用户容器

War包容器启动后执行了 "cp /sample.war /app" 把应用的War包拷到/app目录下然后退出
然后/app 目录就挂载了一个叫app-volume的Volume,而tomcat也同样声明了挂载app-volume到自己的webapps目录下，所以等tomcat启动时，它的webapps目录下已经存在了sample.war文件,这个文件就是War包容器启动时拷贝到这个Volume里的，而且这个Volume被两个容器共享

**案例2: 容器日志收集**
场景: 有一个应用，需要不断的把日志输出到容器的/var/log目录中
这时就可以把一个Pod里的Volume挂载到应用容器的/var/log目录中,然后在Pod里同时运行一个**sidecar**容器,也声明挂载同一个Volume到自己的/var/log目录上，接下来sidecar容器只需要不断的从自己的/var/log目录读取日志，转发到MongoDB或者Elasticsearch中存储起来就可以了。

***sidecar容器*** 一种单节点、多容器的应用设计形式,sidecar主张以额外的容器来扩展或增强主容器，这个额外的容器就被称为sidecar容器