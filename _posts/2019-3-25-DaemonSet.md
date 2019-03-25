[toc]

### DaemonSet的意义和特点

**DaemonSet**
DaemonSet的主要作用就是让你在kubernetes集群中运行一个Daemon Pod，这个Pod主要有三个特征:
- 这个Pod运行在kubernetes进群里的每一个节点(Node)上
- 每个节点上只有一个这样的Pod实例;
- 当有新的节点加入Kubernetes集群后该Pod会自动的在新节点上被创建出来;而当旧节点被删除后,节点上面的Pod 也会被回收

DaemonSet存在的意义非常重要,简单的几个例子来说明:
1.各种网络插件的Agent组件,都必须运行在每一个节点上,用来处理这个节点上的容器网络
2.各种存储插件的Agent组件，必须运行在每一个节点上,用来处理这个节点上挂载远程存储目录操作容器的Volume目录；
3.各种监控组件和日志组件,必须运行在每一个节点上,负责这个节点上的监控信息和日志收集

跟其他的编排对象不一样,DaemonSet开始运行的时机很多时比整个Kubernetes集群出现的时机都要早，同样举例:
比如某个DaemonSet是一个网络插件Agent组件.这个时候整个Kubernetes集群里可能还没有可用的容器网络,所有Worker节点的状态都是NotReady(NetworkReady=false),这种情况下,普通的Pod肯定不能运行在这个集群上，所有这也就意味着DaemonSet的设计必须要有某种过人之处才行

### DaemonSet工作原理

**API对象的定义**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

这个DaemonSet管理的是一个fluentd-elasticsearch镜像的Pod,这个Pod的功能很实用,通过fluentd将docker容器中的日志转发到ElasticSearch
DeamonSet和Deployment的定义非常相似,只不过它没有replicas字段；它也实用selector选择管理所有携带了name=fluentd-elasticsearch标签的Pod


**DaemonSet保证每个Node有且只有一个被管理的Pod的方法**
这是一个典型的"控制器模型"能够处理的问题

DaemonSet Controller,首先从Etcd里获取所有的Node列表,然后遍历所有的Node。这时就很容易地去检查，当前这个Node是不是有一个携带了name=fluentd-elasticsearch标签的Pod在运行。
检查的结果可能有三种情况:
- 没有这种Pod，这意味着要在这个Node上创建这样一个Pod;
- 有这种Pod，但数量大于1，意味着要把多余的Pod从这个Node上删除掉
- 正好只有一个这种Pod,说明这个节点时正常的

这其中,删除节点上多余的Pod很简单,直接调用Kubernetes API就可以了
在指定的Node节点上创建新的Pod可以用nodeSelector选择Node的名字即可
但是**在kubernetes项目中,nodeSelector是一个将要被弃用的字段了**,有一个功能更完善的字段可以代替它，即**nodeAffinity**,例如：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requireDuringSchedulingIgnoredDuringExecution：
        nodeSelectorTerms:
        - matchExpressions:
          - key: metadata.name
            operator: In
            values:
            - k8s-n1
```
在这个Pod里,声明了spec.affinity字段,然后定义了一个nodeAffinity,其中spec.affinity字段是Pod里调度相关的一个字段。

这里定义的nodeAffinity的含义是:
- requiredDuringSchedulingIgnoredDuringExecution:它的意思是说这个nodeAffinity必须在每次调度的时候予以考虑，同时也意味着可以在某些情况下不考虑这个nodeAffinity;
- 这个Pod将来只允许运行在metadata.name是k8s-n1的节点上

从这个例子中可以看出,nodeAffinity的定义支持更加丰富的语法,这也是它会取代nodeSelector的原因之一

DaemonSet Controller会在创建Pod的时候自动在这个 Pod 的API对象里家伙是哪个这样一个nodeAffinity定义.其中需要绑定的节点名字正是当前正在遍历的这个Node。DaemonSet并不需要修改用户提交的YAML文件里的Pod模板,而是在向kubernetes发起请求之前,直接修改根据补办生成的Pod对象
此外,DaemonSet还会给这个Pod自动加上另外一个与调度相关的字段,叫做tolerations,这个字段意味着这个Pod会"容忍"某些Node的污点
DaemonSet自动加上的toleration如下:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: node.kubernetes.io/unschedulable
    operator: Exists
    effect: NoSchedule
```
Toleration的含义是容忍所有被标记为unschedulable污点的Node，容忍的效果是允许调度。在正常情况下,被标记了unschedulable污点的Node是不会有任何Pod被调度上的(effect: NoSchedule).可是DaemonSet自动的给被管理的Pod加上这个特殊的Toleration就使得这些Pod可以忽略这个限制,从而保证每个节点都会被调度一个Pod。当然如果这个节点有故障的话,Pod可能会启动失败，DaemonSet则会一直重试,直到Pod启动成功.DaemonSet的"过人之处"就是依靠Toleration实现的

总得来说DaemonSet的工作原理就是：
DaemonSet是一个非常简单的控制器,在它的控制循环中，只需要遍历所有的节点,然后根据节点上是否有被管理Pod的情况来决定删除或者创建一个Pod



### DaemonSet使用方法

***fluentd-elasticsearch.yaml***
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```
***在DaemonSet中一般都应该加上resources字段,防止它占用过多的宿主机资源***
创建这个DaemonSet对象:
```shell
$ kubectl create -f fluentd-elasticsearch.yaml
```
创建成功后就可以看到,如果有N个节点，就会有N个fluentd-elasticsearch Pod在运行.
```shell
$ kubectl get pod -n kube-system -l name=fluentd-elasticsearch
NAME                          READY     STATUS    RESTARTS   AGE
fluentd-elasticsearch-dqfv9   1/1       Running   0          53m
fluentd-elasticsearch-pf9z5   1/1       Running   0          53m
```
此时通过kubectl get 查看Kubernetes集群里的DaemonSet对象:
```shell
$ kubectl get ds -n kube-system fluentd-elasticsearch
NAME                    DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-elasticsearch   2         2         2         2            2           <none>          1h
```
可以发现,DaemonSet和Deployment一样也有DESIRED、CURRENT等多个状态字段.这就意味着,DaemonSet可以像Deployment那样,进行版本管理这个版本可以使用kubectl rollout history看到
```shell
$ kubectl rollout history daemonset fluentd-elasticsearch -n kube-system
daemonsets "fluentd-elasticsearch"
REVISION  CHANGE-CAUSE
1         <none>
```
然后把这个DaemonSet的容器镜像版本升级到v2.2.0:
```shell
$ kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --record -n=kube-system
```
这个时候可以用kubectl rollout status命令查看这个滚动更新的过程:
```shell
$ kubectl rollout status ds/fluentd-elasticsearch -n kube-system
Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 0 out of 2 new pods have been updated...
Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 0 out of 2 new pods have been updated...
Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 1 of 2 updated pods are available...
daemon set "fluentd-elasticsearch" successfully rolled out
```


**DaemonSet的版本维护**
一切节对象
在kubernetes中，任何你觉得需要记录下来的状态都可以被用API对象的方式实现。
在kubernetesv1.7之后添加了一个API对象,名叫ControllerRevision，专门用来记录某种Controller对象的版本
```shell
$ kubectl get controllerrevision -n kube-system -l name=fluentd-elasticsearch
NAME                               CONTROLLER                             REVISION   AGE
fluentd-elasticsearch-64dc6799c9   daemonset.apps/fluentd-elasticsearch   2          1h
```
使用kubectl describe 查看这个ControllerRevision对象:
```shell
$ kubectl describe controllerrevision fluentd-elasticsearch-64dc6799c9 -n kube-system
Name:         fluentd-elasticsearch-64dc6799c9
Namespace:    kube-system
Labels:       controller-revision-hash=2087235575
              name=fluentd-elasticsearch
Annotations:  deprecated.daemonset.template.generation=2
              kubernetes.io/change-cause=kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --record=true --namespace=kube-system
API Version:  apps/v1
Data:
  Spec:
    Template:
      $ Patch:  replace
      Metadata:
        Creation Timestamp:  <nil>
        Labels:
          Name:  fluentd-elasticsearch
      Spec:
        Containers:
          Image:              k8s.gcr.io/fluentd-elasticsearch:v2.2.0
          Image Pull Policy:  IfNotPresent
          Name:               fluentd-elasticsearch
...
Revision:                  2
Events:                    <none>
```
可以看到这个ControllerRevision独享实际上是在Data字段保存了该版本对应的完整的DaemonSet的API对象,并且Annotation字段保存了创建这个对象所使用的kubectl命令

接下来回滚到Revision=1时的状态:
```shell
$ kubectl rollout undo daemonset fluentd-elasticsearch --to-revision=1 -n kube-system
daemonset.extensions/fluentd-elasticsearch rolled back
```
这个kubectl rollout undo 操作实际上相当于读取到了Revision=1的ControllerRevision对象保存的Data字段,这个Data字段保存的信息就是Revision=1时这个DaemonSet的完整API对象.
现在DaemonSetController就可以使用这个历史API对象对现有的DaemonSet做一次PATCH操作(等价于执行一次kubectl apply -f "旧的DaemonSet对象"),从而把这个DaemonSet更新到一个旧版本

这也是为什么在执行完这次回滚后,DaemonSet的Revision并不会从Revision=2退回到1，而是会增加成Revision=3，这是因为一个新的ControllerRevision被创建了出来


