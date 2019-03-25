---

title: StatefulSet

categories:

- Container

tags:

---

# deployment的问题

Deployment实际上并不满足所有的应用编排问题.其根本问题原因在于Deployment对应用做了一个简单的假设，它认为一个应用的所有Pod是完全一样的所以他们之间没有相互顺序也无所谓运行在哪台宿主机上。需要的时候Deployment就可以通过Pod模板创建新的Pod不需要的时候Deployment就可以杀掉任意一个Pod。

但是实际场景中并不是所有的应用都可以满足这样的要求，尤其是分布式应用，它的多个实例之间往往有依赖关系，比如主从关系，主备关系。 还有就是数据存储类应用，它的多个实例往往都会在本地磁盘保存一份数据，而这些实例一旦被杀掉，即便重建出来，实例与数据之间的对应关系也已经丢失，从而导致应用失败.

# StatefulSet的工作原理

StatefulSet这个控制器的主要作用之一就是使用Pod模板创建Pod的时候对他们进行编号,按照编号顺序逐一完成创建工作。而当StatefulSet的控制循环发现Pod的实际状态与期望状态不符时，需要新建或者删除Pod进行协调的时候它会严格按照这些Pod编号的顺序逐一完成这些操作

StatefulSet的设计非常容易理解，它把真实世界里的应用状态抽象为两种情况

- **拓扑状态:** 这种情况意味着多个实例之间不是完全对等的关系，这些应用实例必须按照某些顺序启动。比如主节点A要先于节点B启动。并且如果把A、B两个Pod删掉，再重新创建时也必须按照这个顺序才行。并且新创建的Pod必须和原来Pod的网络标识一样,这样原先的访问者才能使用同样的方法访问到这个新Pod
- **存储状态:** 这种状态意味着,应用的多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A第一次读取到的数据，隔了一段时间之后再次读取到的数据应该是同一份，哪怕在这期间Pod A被重新创建过

**StatefulSet的核心功能就是通过某种方式记录这些状态,然后在Pod被重新创建时,能够为新Pod恢复这些状态**

## 拓扑状态

### Headless Service

Service 是kubernetes项目中用来将一组Pod暴露给外界访问的一种机制

**Service被访问的方式**

- **以Service的VIP(Virtual IP)方式** 比如:当我访问10.0.23.1这个Service的IP地址时,10.0.23.1其实就是一个VIP，它会把具体请求转发到该Service所代理的某一个Pod上
- **以Service的DNS方式** 比如: 只要我访问"my-svc.my-namespace.svc.cluster.local"这条DNS记录就能访问到名叫my-svc的Service所代理的某一个Pod

其中Service DNS 的方式下，具体还可以分为两种处理方法:
1. **Normal Service**  这种情况下 访问"my-svc.my-namespace.svc.cluster.local"解析到的正是my-svc这个Service的VIP,后面的流程就跟VIP方式一样了
2. **Headless Service** 这种情况下,访问"my-svc.my-namespace.svc.cluster.local"解析到的直接就是my-svc代理的某一个Pod的IP地址

可以看到他们的区别在于Headless Service不需要分配一个VIP,而是可以直接以DNS记录的方式解析出被代理Pod的IP地址

**StatefulSet使用DNS记录维护Pod的拓扑状态**

***Headless Service 的yaml文件***
svc.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector: 
    app: nginx
```

所谓的Headless Service，仍是一个标准Service的YAML文件,只不过它的clusterIP字段为None。这个Service没有一个VIP作为"头".所以这个Service被创建之后不会被分配一个VIP，而是会以DNS记录的方式暴露出它所代理的Pod。像这个yaml文件的定义中，它所代理的Pod就是携带了app=nginx标签的Pod。
按照这种方式创建了一个Headless Service之后,它所代理的所有Pod的IP地址都会被绑定一个这样格式的DNS记录:

```text
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
这个DNS记录正是kubernetes为Pod分配的唯一的"可解析身份"(Resolvable Identify)
```


nginx-statefulset.yaml
```yaml
apiVersion: apps/v1
kind: statefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
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
        image: nginx:1.9.1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: web
```
在这个yaml文件中，和之前的nginx-deployment(deployment笔记中的例子)的唯一区别,就是多了一个serviceName=nginx字段
这个字段的作用就是告诉StatefulSet控制器,在执行控制循环(control loop)的时候,请使用nginx这个Headless Service来保证Pod的"可解析身份"

```shell
# kubectl create -f svc.yaml
# kubectl get service nginx

NAME    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx   ClusterIP   None         <none>        80/TCP    8s

# kubectl create -f nginx-statefulset.yaml
# kubectl get statefulset web

NAME   READY   AGE
web    2/2     8s

# kubectl get pod -l app=nginx

NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          2m40s
web-1   1/1     Running   0          2m38s
```
可以看到StatefulSet给它所管理的所有的Pod的名字进行了编号,这些编号都是从0开始累加,与StatefulSet的每个Pod实例一一对应,绝不重复。而且当创建这些Pod的时候也是严格按照编号进行的，比如 web-0进入到Running状态并且细分状态(Conditions)成为ready之前，web-1会一直处于Pending状态
```shell
### 查看Pod中容器的Hostname
# kubectl exec web-0 -- sh -c 'hostname'
web-0

# kubectl exec web-1 -- sh -c 'hostname'
web-1
```
可以看到,这两个Pod的hostname与Pod名字是一致的,都被分配了对应的编号,来再看看以DNS的方式访问这个Headless Service
```shell
# kubectl run -i  --tty --image busybox:1.28.4  dns-test --restart=Never --rm /bin/sh
### 这条命令是启动了一个一次性的Pod,因为--rm 意味着Pod退出后就会删除,之后在这个Pod 中尝试用nslookup命令解析一下Pod 对应的Headless Service:
/ # nslookup web-0.nginx

Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.1.43 web-0.nginx.default.svc.cluster.local

/ # nslookup web-1.nginx

Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.1.44 web-1.nginx.default.svc.cluster.local
```
可以看到在这个StatefulSet中这两个新Pod的网络标识(web-0.nginx和web-1.nginx)再次解析到了正确的IP地址.通过这种方式kubernetes就可以成功的将Pod的拓扑状态按照Pod的"名字"+"编号"的方式固定下来，此外Kubernetes还为每一个Pod提供了一个固定并且唯一的访问入口,即:这个Pod对应的DNS记录

## 存储状态

StatefulSet对存储状态的管理机制主要使用的是一种叫做**Persistent Volume Claim**的功能

在往常需要在一个Pod里声明Volume，只要在Pod里加上spec.volumes字段即可，然后就可以在这个字段里定义一个具体类型的Volume了，比如RBD类型:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rbd
spec:
  containers:
  - images: kubernetes/pause
    name: rbd-rw
    volumeMounts:
    - name: rbdpd
      mountPath: /mnt/rbd
  volumes:
  - name: rbdpd
    rbd:
      monitors:
      - '10.16.154.78:6789'
      - '10.16.154.82:6789'
      - '10.16.154.83:6789'
      pool: kube
      image: foo
      fsType: ext4
      readOnly: true
      user: admin
      keyring: /etc/ceph/keyring
      imageformat: "2"
      imagefeatures: "layering"
```

如果不懂得Ceph RBD的使用方法,那么这个Pod的Volumes字段也十有八九看不懂，有暴露公司基础设施秘密的风险。这也是为什么在后来的演化中**Kubernetes项目引入了一组叫做Persistent Volume Claim(PVC)和Persistent Volume(PV)的API对象,大大降低了用户声明和使用持久化Volume的门槛**
PVC就是一种特殊的Volume，只不过一个PVC具体是什么类型的Volume，要跟某个PV绑定之后才知道。

### PVC的使用

<span id=1> 1. 定义一个PV对象，为PVC提供符合条件的Volume</span>

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-volume
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  rbd:
    monitors:
    - '10.16.154.78:6789'
    - '10.16.154.82:6789'
    - '10.16.154.83:6789'
    pool: kube
    image: fool
    fsType: ext4
    readOnly: true
    user: admin
    keyring: /etc/ceph/keyring
    imageformat: "2"
    imagefeatures: "layering"
```
这个PV对象的spec.rbd字段,正是前面的Ceph RBD Volume的详细定义。在这个PV对象中,PV的容量是10GiB。这样再声明PVC的时候，kubernetes就会把这个PV绑定到PVC中


2. 定义一个PVC，声明想要的Volume属性:
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    request:
      sotrage: 1Gi
```
在这个PVC对象中，不需要任何关于Volumes细节的字段,只有描述性和定义。比如storage: 1Gi,表示想要的Volume大小至少是1GiB，accessModes: ReadWriteOnce,表示这个Volume是可读写的，并且最多只能再一个节点中挂载，不能多个节点共享
对于哪种Volume支持哪些accessModes,Kubernetes官方文档中有[详细的列表](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)

3. 在应用的Pod中声明使用这个PVC
```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: pv-pod
spec:
  containers:
  - name: pv-container
    image: nginx
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 80
      name: "http-server"
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: pv-storage
  volumes:
  - name: pv-sotrage
    persistentVolumeClaim:
      claimName: pv-claim
```
在这个Pod的Volume定义中,我们只需要声明它的类型是persistentVolumeCliam，然后指定PVC的名字，完全不必关心Volume本身的定义

**Kubernetes中PVC和PV的设计类似于"接口"和"实现"的思想** 开发者只要知道并会使用"接口"(PVC),而运维人员则负责给"接口"绑定具体的实现，即PV.这种解耦也避免了向开发者暴露过多的存储系统细节而带来的隐患。

### StatefulSet对存储状态的管理

PVC和PV的实现也让StatefulSet对存储状态的管理成为了可能
以"拓扑状态"的例子来说:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
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
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```
相比之前的"拓扑状态"额外添加了volumeClaimTemplates字段.他跟Deployment里的Pod模板作用类似。这个字段说明,凡是被这个StatefulSet管理的Pod，都会声明一个对应的PVC，而这个PVC的定义就来自于volumeClaimTemplates这个模板字段.并且，这个PVC的名字会被分配一个与这个Pod完全一致的编号,他们的命名规则都符合<pvc-name>-<StatefulSet-name>-<编号>。

PVC与PV绑定得以实现的前提是运维人员以及在系统中创建好了符合条件的PV;或者Kubernetes集群运行在公有云上,这样Kubernetes就会通过Dynamic Provisioning的方式 自动创建与PVC匹配的PV

所以当创建StatefulSet之后，kubernetes集群中应该会出现两个PVC

```shell
# kubectl create -f nginx-pv.yaml
# kubectl get pvc -l app=nginx

NAME        STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
www-web-0   Bound     pvc-15c268c7-b507-11e6-932f-42010a800002   1Gi        RWO           48s
www-web-1   Bound     pvc-15c79307-b507-11e6-932f-42010a800002   1Gi        RWO           48s
```
StatefulSet创建出来的所有Pod都会使用编号的PVC，比如，名叫web-0的Pod的volumes字段它会声明使用名叫www-web-0的PVC从而挂载到这个PVC所绑定的PV

```
### 在Pod的Volume目录写入一个文件
# for i in 0 1; do kubectl exec web-$i -- sh -c 'echo  hello $(hostname) > /usr/share/nginx/html/index.html'; done
```
上面这条命令意思是在每个Pod的Volume目录里写入了一个index.html文件，这个文件内容正是Pod的hostname.这个时候访问http://localhsot 实际访问的就是Pod里的Nginx服务器进程，而它会返回/usr/share/nginx/html/index.html里的内容(提前创建Headless Service)

```shell
# for i in 0 1; do kubectl exec -ti web-$i -- curl localhost; done
hello web-0
hello web-1
```

现在来测试StatefulSet的存储状态
```
删除两个Pod查看数据是否会丢失
# kubectl delete pod -l app=nginx
pod "web-0" deleted
pod "web-1" deleted
```
在删除之后，这两个Pod会被按照编号的顺序重新被创建，然后再次访问http://localhost
```shell
# kubectl exec -ti web-0 -- curl localhost
hello web-0
```
这个请求依然返回hello web-0,也就是说原先名为web-0的Pod绑定的PV 在这个Pod被重建之后，依然同新的名叫web-0的Pod绑定在了一起。这是如何实现的?
分析一下StatefulSet控制器恢复这个Pod的过程就很容易理解了,
首先，当把一个Pod(比如 web-0)删除之后，这个Pod对应的PVC和PV并不会被删除，而这个Volume里已经写入的数据也会保存在远程存储服务器。此时StatefulSet控制器发现实际状态与期望状态不一致，就会把删掉的Pod重新创建出来.注意，在心底Pod对象定义里，它声明使用的PVC的名字还是叫做www-web-0,这个PVC的定义是来自PVC模板(volumeClaimTemplates),这是StatefulSet创建Pod的标准流程。所以在新的web-0 Pod 被创建出来之后Kubernetes为它查找名叫www-web-0的PVC时,就会直接找到旧Pod对应的那个Volume，并且获取到保存在Volume里的数据。

# 总结

梳理一下StatefulSet的工作原理

**首先，statefulSet的控制器直接管理的是Pod**  这是因为StatefulSet里的不通Pod实例不再像ReplicaSet中那样都是完全一致的。比如每个Pod的hostname、名字等都是不同的，携带了编号的。而StatefulSet区分这些实例的方式就是通过在Pod的名字里面加上事先约定好的编号

**其次，Kubernetes通过Headless Service为这些有编号的Pod在DNS服务器中生成带有同样编号的DNS记录** .只要StatefulSet能够保证这些Pod名字里的编号不变,那么Service里类似于web-0.nginx.default.svc.cluster.local这样的DNS记录也就不会变,而且这条记录解析出来的Pod的IP地址则会随着后端Pod的删除和再创建而自动更新，这是Service的功能，不用StatefulSet管理。


**最后，StatefulSet还为每一个Pod分配并创建一个同样编号的PVC**  这样kubernetes就可以通过Persisten Volume机制为这个PVC绑定对应的PV,从而保证了每一个Pod都拥有一个独立的Volume.在这种情况下,即使Pod被删除,它所对应的PVC和PV已然会保留下来,所以当这个Pod被重新创建出来后,kubernetes会为他找到同样编号的PVC，挂载这个PVC对应的Volume,从而获取到之前保存在Volume里的数据

# 实践: MySQL集群

本实践来自[Kubernetes官方文档](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/)

实践需求:
1. 是一个"主从复制"(Master-Slave Replication)的MySQL集群
2. 有一个主节点
3. 有多个从节点
4. 从节点需要能够水平扩展
5. 所有的写操作只能在主节点上执行
6. 读操作可以在所有节点上执行

这个需求可以用下面这张图来描述
![image](https://static001.geekbang.org/resource/image/bb/02/bb2d7f03443392ca40ecde6b1a91c002.png)

在常规环境中,部署这样一个主从模式的MySQL集群的主要难点在于:如何让从节点能够拥有主节点的数据,即:如何配置主从节点的复制同步.

所以在安装好MySQL的Master节点之后需要做的第一步工作就是通过XtraBackup将Master节点的数据备份到指定目录.这一步会在目标目录生成一个备份信息文件，名叫xtrabackup_binlog_info，这个文件一般包含下面两个内容,这些内容将在配置slave节点时使用
```shell
# cat xtrabackup_binlog_info
TheMaster-bin.000001     481
```
第二步: 配置Slave节点.slave节点在第一次启动前,需要先把Master节点的备份数据连同备份信息文件，一起拷贝到自己的数据目录(/var/lib/mysql)下,然后来执行下面一句SQL:
```sql
TheSlave|mysql> CHANGE MASTER TO
                MASTER_HOST='$masterip',
                MASTER_USER='xxx',
                MASTER_PASSWORD='xxx',
                MASTER_LOG_FILE='TheMaster-bin.000001',
                MASTER_LOG_POS='481';
```
其中MASTER_LOG_FILE和MASTER_LOG_POS就是备份对应的二进制日志文件的名称和开始位置(偏移量),也是xtrabackup_binlog_info文件中的两部分内容

第三步: 启动Slave节点
```sql
TheSlave|mysql> START SLAVE;
```

第四步: 在这个集群中添加更多的Slave节点

这一步需要注意,新添加的Slave节点的备份数据,来自已经存在的Slave节点
所以在这一步,需要将Slave节点的数据备份在指定目录.而这个备份操作会自动生成另外一种备份信息文件,xtrabackup_slave_info.同样的,这个文件包含了MASTER_LOG_FILE和MASTER_LOG_POS两个字段
之后就可以在新节点中执行跟前面一样的"CHANGE MASTER TO"和"START SLAVE"指令来初始化并启动新的Slave节点了

通过上面的信息可以了解到,将部署MySQL集群的流程迁移到Kubernetes项目上,需要能够容器化地解决下面的困难点:
- Master节点和Slave节点需要有不同的配置文件(my.cnf)
- Master节点和Slave节点需要能够传输备份信息文件
- 在Slave节点第一次启动之前,需要执行一些初始化操作;



**首先第一点: MASTER和SLAVE需要有不同的配置文件:**
这点很容易处理,只需要给主节点和从节点准备两份不同的MySQL配置文件，就根据Pod的序号挂载进去即可

这样的配置文件信息应该保存在ConfigMap中供Pod使用:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels:
    app: mysql
data:
  master.cnf: |
    #主节点MySQL配置文件
    [mysqld]
    log-bin
  slave.cnf: |
    #从节点MySQL配置文件
    [mysqld]
    super-read-only
```
在这里，定义了master.cnf和slave.cnf两个MySQL的配置文件
- master.cnf开启了log-bin,即:使用二进制日志文件的方式进行主从复制
- slave.cnf开启了super-read-only代表的是从节点会拒绝除了主节点的数据同步操作之外的所有写操作,即它对用户是只读的

接下来需要创建两个Service来供StatefulSet以及用户使用：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
apiVerison: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
```
可以看到,这两个Service都代理了携带app=mysql标签的Pod,也就是所有的MySQL Pod。端口映射都是用Service的3306端口对应Pod的3306端口
不同的是,第一个名叫"mysql"的service是一个Headless Service。它的作用是通过为Pod分配DNS记录来固定它的拓扑状态,比如mysql-0.mysql这样的DNS名字.mysql-0.mysql就是主节点

而第二个名叫"mysql-read"的Service是一个常规的Service
并且，所有用户的读请求都必须访问第二个Service被分配的DNS记录(也可以访问这个Service的VIP)。这样读请求就可以被转发到任意一个MySQL的主节点或者从节点


**第二点: Master节点和Slave节点能够传输备份文件**

**解决这个难点的思路，推荐的做法是先搭框架,再完善细节,其中Pod部分如何定义是完善细节时的重点**

首先来为StatefulSet对象规划一个大致的框架:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
      - name: clone-mysql
      containers:
      - name: mysql
      - name: xtrabackup
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          sotrage: 10Gi
```
在这一步中可以先为StatefulSet定义一些通用的字段,比如它声明要使用的Headless Service的名字是mysql。
这个Stateful Set的replicas值是3，表示它定义的MySQL集群有三个节点,一个Master节点,两个Slave节点.从这一步可以看出,StatefulSet管理的"有状态应用"的多个实例,也都是通过同一份Pod模板创建出来,使用的是同一个Docker镜像.这也就意味着:如果应用要求不同节点的镜像不一样,就不能使用StatefulSet了

除了一些基本的字段外，作为一个有存储状态的MySQL集群，StatefulSet还需要管理存储状态,所以需要通过volumeClaimTemplates(PVC模板)来为每个Pod定义PVC.比如:volumeClaimTemplates.spce.resources.requests.storage指定了存储的大小为10GiB;ReadWriteOnce指定了改存储的属性为可读写,并且一个PV只允许挂载在一个宿主机上。


然后来重新设计一下这个StatefulSet的Pod模板也就是Template字段.
由于在这个例子中StatefulSet管理的Pod都来自于同一个镜像,这个时候就要分析:
1. 如果这个Pod是Master节点,我们需要怎么做;
2. 如果这个Pod是Slave节点,我们又需要怎么做;



**第一步: 从ConfigMap中获取MySQL的Pod的对应的配置文件**

首先,需要先做一个初始化操作来判断节点的角色是Master节点还是Slave节点,为Pod分配对应的配置文件。此外MySQL还要求集群的每个节点都有一个唯一的ID文件,名叫server-id.cnf
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
        - bash
        - "-c"
        - /
        set -ex
        # 从Pod的序列号生成server-id
        [[`hostname` =~ -([0-9]+)$ ]] || exit 1
        ordinal=${BASH_REMATCH[1]}
        echo [mysqld]> /mnt/conf.d/server-id.cnf
        # 由于server-id=0是MySQL保留的,所以需要添加一个偏移量来避开0
        echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
        # 如果Pod的序号是0则说明是Master节点,从ConfigMap中拷贝Master的配置文件
        # 否则拷贝Slave的配置文件
        if [[ $ordinal -eq 0 ]]; then
          cp /mnt/config-map/master.cnf /mnt/conf.d/
        else
          cp /mnt/config-map/slave.cnf /mnt/conf.d/
        fi
      volumeMounts:
      - name: conf
        mountPath: /mnt/conf.d
      - name: config-map
        mountPath: /mnt/config-map
```
这个名为init-mysql的InitContainer的配置中，它从Pod的hostname里读取到了Pod的序号,以此作为MySQL节点的Server-id.然后init-mysql通过这个序号判断当前Pod到底是Master节点(即序号为0)，还是Slave节点(序号不为0)，从而把对应的配置文件从/mnt/config-map目录拷贝到/mnt/conf.d/目录下.
其中文件拷贝的源目录是/mnt/config-map,正是ConfigMap在这个Pod的Volume
```yaml
...
# template.spec
volumes:
- name: conf
  emptyDir: {}
- name: config-map
  configMap:
    name: mysql
```
而文件拷贝的目录，即容器中的/mnt/conf.d目录对应的则是一个名叫conf的emptyDir类型的Volume 基于Pod Volume共享的原理,当InitContainer复制完配置文件退出后，后面启动的MySQL容器只要直接声明挂载这个名叫conf的Volume，它所需要的.cnf配置文件就已经出现在里面了


**第二步: 在Slave Pod启动之前,从Master或者其他Slave Pod里拷贝数据库到自己的目录下**

为了实现这个操作，还需要定义第二个InitContainer
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
      ...
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          #拷贝操作只需要在第一次启动时进行，如果数据已经存在,则跳过
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # master节点不需要做这个操作
          [[ `hostname` =~ -([0-9]+)$ ]] exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # 使用ncat指令,远程的从前一个节点拷贝数据到本地
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # 执行--prepare,这样拷贝来的数据就可以用作恢复了
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
```
这个名为clone-mysql的InitContainer里。使用的是xtrabackup镜像。在它的启动命令中首先做了一个判断：当初始化所需的目录(/var/lib/mysql)已经存在和当前Pod是Master节点的时候不需要做拷贝操作.

然后clone-mysql会使用Linux自带的ncat指令,向DNS记录为"mysql-<当前序列号-1>.mysql"的Pod,也就是当前Pod的前一个Pod发起传输请求,并且直接用xbstream指令将受到备份数据保存在/var/lib/mysql目录下
3307是一个特殊端口,运行着一个专门负责备份MySQL数据的辅助进程

而且这个容器里的/var/lib/mysql目录实际上正是一个名为data的PVC，这个PVC在之前的框架中定义过了。这样就可以保证，及时宿主机宕机了数据库的数据也不会丢失.后面启动的MySQL容器就可以把这个Volume挂载到自己的/var/lib/mysql 目录下,直接使用里面的备份数据进行恢复操作

不过clone-mysql容器还要对/var/lib/mysql目录执行依据xtrabackup --prepare 操作,目的是上拷贝来的数据进入一致性状态,这样这些数据才能被用作数据恢复

**第三点: 如何在Slave节点的MySQL容器第一次执行时执行初始化SQL**

接下来开始定义MySQL容器,启动MySQL服务。由于StatefulSet；里的所有Pod都来自于同一个Pod模板,所以还要区分主节点和从节点,它们的启动命令有什么不同

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
      ...
      - name: clone-mysql
      ...
      ...
      containers
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql
          
          # 从备份信息文件里读取 MASTER_LOG_FILE 和 MASTER_LOG_POS 这两个字段的值，用来拼装集群初始化 SQL
          if [[ -f xtrabackup_slave_info ]]; then
            # 如果 xtrabackup_slave_info 文件存在，说明这个备份数据来自于另一个 Slave 节点。这种情况下，XtraBackup 工具在备份的时候，就已经在这个文件里自动生成了 "CHANGE MASTER TO" SQL 语句。所以，我们只需要把这个文件重命名为 change_master_to.sql.in，后面直接使用即可
            mv xtrabackup_slave_info change_master_to.sql.in
            # 所以，也就用不着 xtrabackup_binlog_info 了
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # 如果只存在 xtrabackup_binlog_inf 文件，那说明备份来自于 Master 节点，我们就需要解析这个备份信息文件，读取所需的两个字段的值
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm xtrabackup_binlog_info
            # 把两个字段的值拼装成 SQL，写入 change_master_to.sql.in 文件
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi
          
          # 如果 change_master_to.sql.in，就意味着需要做集群初始化工作
          if [[ -f change_master_to.sql.in ]]; then
            # 但一定要先等 MySQL 容器启动之后才能进行下一步连接 MySQL 的操作
            echo "Waiting for mysqld to be ready (accepting connections)"
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done
            
            echo "Initializing replication from clone position"
            # 将文件 change_master_to.sql.in 改个名字，防止这个 Container 重启的时候，因为又找到了 change_master_to.sql.in，从而重复执行一遍这个初始化流程
            mv change_master_to.sql.in change_master_to.sql.orig
            # 使用 change_master_to.sql.orig 的内容，也是就是前面拼装的 SQL，组成一个完整的初始化和启动 Slave 的 SQL 语句
            mysql -h 127.0.0.1 <<EOF
          $(<change_master_to.sql.orig),
            MASTER_HOST='mysql-0.mysql',
            MASTER_USER='root',
            MASTER_PASSWORD='',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
          fi
          
          # 使用 ncat 监听 3307 端口。它的作用是，在收到传输请求的时候，直接执行 "xtrabackup --backup" 命令，备份 MySQL 的数据并发送给请求者
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
```
在这个名叫xtrabackup的sidecar容器的启动命令中,其实实现了两部分工作，第一部分工作当然是mysql节点的初始化工作，这个初始化需要使用的SQL是sidecar容器拼装出来、保存在一个名为change_master_to_sql.in的文件里的,具体过程如下:

sidecar 容器首先会判断当前pod的/var/lib/mysql目录下是否有xtrabackup_slave_info这个备份信息文件
- 如果有，则说明这个目录下的备份数据是由一个slave节点生成的,这种情况下xtrabackup工具在备份的时候就已经在这个文件里自动生成了"CHANGE MASTER TO"SQL语句。所以，只需要把这个文件重命名为change_master_to.sql.in。后面直接使用即可
- 如果没有xtrabackup_slave_info文件、但是存在xtrabackup_binlog_info文件,那就说明备份文件来自于Master节点,这种情况下,sidecar容器就需要解析这个备份信息文件,读取MASTER_LOG_FILE和MASTER_LOG_POS这两个字段的值，用他们拼装出初始化SQL语句，然后把这句SQL写入到change_master_to.sql.in文件中

最后,sidecar容器就可以执行初始化了.从上面的叙述中可以看到,只要这个change_master_to.sql.in文件存在,那就说明接下来需要进行集群初始化操作

所以这个时候sidecar容器只需要读取并执行change_master_to.sql.in里面的"CHANGE MASTER TO"指令,再执行依据START SLAVE命令,一个Slave节点就被成功启动了

上述初始化操作完成后还要删掉前面用到的这些备份信息文件,否则下次这个容器重启时,就会发现这些文件存在,所以又会重新执行一次数据恢复和集群初始化操作。

在MySQL执行完上述初始化操作后，sidecar容器的第二个工作就是启动一个数据传输服务

具体做法是：sidecar容器会使用ncat命令启动一个工作在3307端口上的网络发送服务。一旦受到数据传输请求是,sidecar容器就回调用xtrabackup --backup指令备份当前MySQL的数据，然后把这些备份数据返回给请求者,这就是为什么在InitContainer里定义数据拷贝的时候访问的是上一个MySQL节点的3307端口

再解决了这三个问题之后,再定义Pod里的MySQL容器就很简单了
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
      ...
      - name: clone-mysql
      ...
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # 通过 TCP 连接的方式进行健康检查
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
```
**所有完整的文件内容**

***mysql-configmap.yaml***
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels:
    app: mysql
data:
  master.cnf: |
    # Apply this config only on the master.
    [mysqld]
    log-bin
  slave.cnf: |
    # Apply this config only on slaves.
    [mysqld]
    super-read-only
```
***rook-storage.yaml***
```yaml
apiVersion: ceph.rook.io/v1beta1
kind: Pool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: ceph.rook.io/block
parameters:
  pool: replicapool
  clusterNamespace: rook-ceph
```
***mysql-svc.yaml***
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
apiVerison: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
```
***mysql-statefulset.yaml***
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Generate mysql server-id from pod ordinal index.
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # Add an offset to avoid reserved server-id=0 value.
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # Copy appropriate conf.d files from config-map to emptyDir.
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Skip the clone if data already exists.
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Skip the clone on master (ordinal index 0).
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # Clone data from previous peer.
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # Prepare the backup.
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # Check we can execute queries over TCP (skip-networking is off).
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql

          # Determine binlog position of cloned data, if any.
          if [[ -f xtrabackup_slave_info ]]; then
            # XtraBackup already generated a partial "CHANGE MASTER TO" query
            # because we're cloning from an existing slave.
            mv xtrabackup_slave_info change_master_to.sql.in
            # Ignore xtrabackup_binlog_info in this case (it's useless).
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # We're cloning directly from master. Parse binlog position.
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm xtrabackup_binlog_info
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi

          # Check if we need to complete a clone by starting replication.
          if [[ -f change_master_to.sql.in ]]; then
            echo "Waiting for mysqld to be ready (accepting connections)"
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done

            echo "Initializing replication from clone position"
            # In case of container restart, attempt this at-most-once.
            mv change_master_to.sql.in change_master_to.sql.orig
            mysql -h 127.0.0.1 <<EOF
          $(<change_master_to.sql.orig),
            MASTER_HOST='mysql-0.mysql',
            MASTER_USER='root',
            MASTER_PASSWORD='',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
          fi

          # Start a server to send backups when requested by peers.
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

