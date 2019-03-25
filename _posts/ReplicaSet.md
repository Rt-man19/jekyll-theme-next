ReplicaSet是下一代Replication Controller,ReplicaSet和Replication Controller的唯一区别就是选择器的支持。
ReplicaSet支持基于集合的选择器，而Replication Controller只支持基于等式的选择器。

ReplicaSet的结构非常简单，下例说明:
```yaml
apiVersion: v1
kind: ReplicaSet
metadata:
  name: nginx-set
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx: 1.7.9
```
这个YAML文件中可以看到，一个ReplicaSet对象其实就是由副本数目定义和一个Pod模板组成，不难看出ReplicaSet是Deployment的一个子集，更重要的是**Deployment控制器操作的正是这样的ReplicaSet对象,而不是Pod对象**所以，当Deployment所管理的Pod，它的ownerReference是ReplicaSet


**ReplicaSet与Deployment的关系**

***案例***

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  tempalte: 
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx: 1.7.9
        ports:
        - containerPort: 80
```
可以看出,这个yaml定义的Pod的副本数量是3(spec.replicas=3)
Deployment，ReplicaSet，以及Pod的关系可以用下面一张图表示:
![image](85048CCDF6554904BD71E3E0E563C8B8)

这张图就可以清楚的看到，一个定义了replicas=3的Deployment与它的ReplicaSet以及Pod的关系实际上是一种层层控制的关系

其中,ReplicaSet负责通过"控制器模式",保证系统中Pod的个数永远等于指定的个数,这也正是Deployment只允许容器的restrtPolicy=Always的主要原因:只有在容器保证自己始终是Running状态的前提下,ReplicaSet调整Pod的个数才有意义。在此基础上Deployment同样通过"控制器模型"来操作ReplicaSet的个数和属性进而实现"水平扩展/伸缩"和滚动更新着两个编排动作。

这其中，水平扩展/伸缩功能很容易实现,Deployment只需要修改它所控制的ReplicaSet的Pod的副本个数就可以了。用户想要执行这个操作指令也非常简单，kubectl scale:
```shell
kube scale deployment nginx-deployment --replicas=4
deployment.apps/nginx-deployment scaled
```

**滚动更新**

滚动更新就是将一个集群中正在运行的多个Pod版本，交替地逐一升级的过程
通过上面的例子来描述滚动更新的过程:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  tempalte: 
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx: 1.7.9
        ports:
        - containerPort: 80
```
1.创建这个nginx-deployment:
```shell
kubectl create -f nginx-deployment.yaml --record
```
***--record的作用是记录下每次操作所执行的命令***

2.查看nginx-deployment的创建状态信息:
```shell
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         0         0            0           1s
```
DESIGED: 用户期望的Pod副本数量(spec.replicas)
CURRENT: 当前处于Running状态的Pod的数量
UP-TP-DATE: 当前处于最新版本的Pod的数量，最新版本指的是Pod的spec部分与Deployment里Pod模板里定义的完全一致
AVAILABLE: 当前已经可用的Pod的个数,即:是Running状态又是最新版本并且已经处于Ready状态的Pod的个数,这个字段描述的才是用户期望的状态


kubernetes还提供了一条指令可以实时查看Deployment对象的状态变化,这条指令就是kubectl rollout status
```shell
kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment.apps/nginx-deployment successfully rolled out
###继续等待一会就可以看到这个Deployment的3个Pod进入到了AVAILABLE状态:
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           20s
###这时可以尝试查看一下这个Deployment所控制的ReplicaSet:
kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-76bf4969df   3         3         3       20s
```
如上所示,在用户提交了一个Deployment对象后,Deployment Controller就回立即创建一个Pod副本个数为3的ReplicaSet，这个ReplicaSet的名字，则是由Deployment的名字和一个随机字符串共同组成

这个随机字符串叫做pod-template-hash 在这个例子中就是76bf4969df,ReplicaSet会把这个随机字符串加到它所控制的所有Pod的标签里,从而保障这些Pod不会与集群中的其他Pod混淆

ReplicaSet的DESIRED/CURRENT/READY字段的含义和Deployment中是一致的,Deployment只是在ReplicaSet的基础上添加了UP-TO-DATE这个跟版本有关的状态字段

这个时候如果修改deployment的Pod模板，滚动更新就会被自动触发
修改deployment有很多种方法。比如，可以直接使用kubectl edit指令编辑Etcd里的API对象

```shell
kubectl edit deployment/nginx-deployment
... 
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1 # 1.7.9 -> 1.9.1
        ports:
        - containerPort: 80
...
deployment.extensions/nginx-deployment edited
```
kubectl edit指令会直接打开nginx-deployment的API对象然后就可以修改这里的Pod模板部分了。这里将nginx版本升级到了1.9.1

kubectl edit 指令编辑完成后，保存退出，kubernetes就回立即出发滚动更新的过程,你可以通过kubectl rollout status指令查看nginx-deployment的状态变化

```shell
kubectl rollout status deployment/nginx-deployment

Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment.extensions/nginx-deployment successfully rolled out
```
用户可以使用kubectl describe deployment nginx-deployment查看Event，可以看到滚动更新的过程


**滚动更新的特点:**

1. 如果新版本的Pod有问题无法启动，那么滚动更新将会停止
2. DeploymentController会确保在任何时间内，永远只有指定比例的Pod处于离线状态以确保服务的连续性。同时还确保在任何时间内，只有指定比例的Pod会被创建，这两个比例的值默认都是25%

***更改在一次滚动中被创建或删除的Pod的比例***

在spec.strategy.rollingUpdate.maxUnavailable,spec.strategy.rollingUpdate.maxSurge中更改,需要指定spec.strategy中指定type为RollingUpdate,maxSurge指定的是除了DESIRED之外在一次滚动中，Deployment控制器还可以创建多少新Pod，maxUnavailable指定的是在一次滚动中Deployment控制器可以删除多少旧Pod。可以指定具体数字，也可以指定一个百分比



### Deployment对应用进行版本控制的具体原理

Deployment实际上是一个**两层控制器**首先，它通过ReplicaSet的个数来描述应用的版本;然后通过ReplicaSet的属性(如:replicas的值)来保障Pod的副本数量

随着应用版本的不断增加，Kubernetes中还是会以同一个Deployment保存很多很多不同的ReplicaSet，在Deployment对象中有一个字段，spec.revisionHistoryLimit，就是kubernetes保留的"历史版本"个数，如果把它设置为0，那么就不能做回滚操作了

Kubernetes项目对Deployment的设计实际上是代替我们完成了对应用的抽象，是我们可以使用这个Deployment对象来描述应用,使用kubectl rollout命令控制应用的版本。

---

**使用案例说明原理:**

以之前创建好的ReplicaSet为例，这次使用kubectl set image指令直接修改nginx-deployment所使用的镜像,这样就可以不必像kubectl edit 那样打开编辑器。
这次，我把镜像名字修改成了错误的名字,比如nginx:1.91 这样Deployment 就会出现一个升级失败的版本

```shell
kube set image deployment/nginx-deployment nginx=nginx:1.91
deployment.extensions/nginx-deployment image updated
```
这个镜像在dockerhub中并不存在，所以会立刻停止报错。
检查ReplicaSet的状态：
```shell
[root@k8s-m1 ~]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-76bf4969df   0         0         0       24h
nginx-deployment-779fcd779f   3         3         3       22h
nginx-deployment-79dccf98ff   1         1         0       87s
```
可以发现，新版本的ReplicaSet(hash=79dccf98ff)的水平扩展已经停止,而此时已经创建了一个Pod但是没有进入READY状态,这是因为Pod拉取不到有效的镜像

现在把Deployment之前的三个Pod回滚到上一个版本,使用kubectl rollout undo deployment/nginx-deployment
```shell
kubectl rollout undo deployment/nginx-deployment
deployment.extensions/nginx-deployment rolled back
```
这条命令在具体实现上就是 Deployment的控制器让这个旧的ReplicaSet(hash=779fcd779f)再次扩展到三个Pod,而让新的ReplicaSet(hash=79dccf98ff)重新收缩到0个Pod。
***关于kubectl rollout 指令***

- kubectl rollout history: 查看每次Deployment变更对应的版本.
- kubectl rollout history --revision 版本号: 查看某个版本的更新明细
- kubectl rollout undo deployment/nginx-deployment --revision=2: 回滚到第二版本
- kubectl rollout pause: 让一个Deployment进入暂停状态,在暂停状态下，对Deployment 进行的而更改都不会触发"滚动更新",也不会创建新的ReplicaSet
- kubectl rollout resume :从暂停状态中恢复回来
注:**在rollout pause和rollout resume期间，只会触发一次"滚动更新"

---

在实际使用过程中，应用发布的流程往往千差万别，也可能有很多的定制化需求,比如应用可能有会话黏连(session sticky)，这就意味着"滚动更新" 的时候，哪个Pod能下线是不能随便选择的。这种场景光靠Deployment自己很难应付了，这种需求可以使用自定义控制器来实现一个更强大的Deployment Controller

