---

title: Job和CronJob

categories:

- Container

tags:

---

### Job和CornJob简介和使用
Job和CornJob通常编排对象是离线业务，也叫作Batch Job(计算业务).这种业务在计算完成后就退出了而且此时如果用Deployment编排的话,Pod会在计算结束后退出,然后被Deployment Controller不断的重启.

在最开始Kubernetes项目并不支持对Batch Job的管理，直到v1.4版本之后社区才逐步设计了一个用来描述离线业务的API对象,它的名字就是:Job

##### Job的使用

Job API 对象的定义非常简单:
***job.yaml***
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-pc
        command: ["sh", "-c","echo 'scale=10000;4*a(1)' | bc -l"]
    restartPolicy: Never
  backoffLimit: 4
```
这个Job对象中定义的Pod模板中定义了一个Ubuntu镜像的容器,它运行的程序是
```shell
echo "scale=10000;4*a(1) | bc -l"
```
bc是linux里的计算器;-l表示现在要使用标准数学库;而a(1),则是调用数学库中的arctangent函数,计算atan(1)
```
tan(π/4)=1
4*atan(1)=π
```
这其实就是一个计算π值的容器,通过scale=10000我指定了输出的小数点后的位数是100000。

跟其它的控制器不同,Job对象并不要求定义spec.selector来描述要控制哪些Pod。

现在创建Pod
```shell
$ kubectl create -f job.yaml
```
查看Job对象
```shell
$ kubectl describe jobs/pi
Name:           pi
Namespace:      default
Selector:       controller-uid=1a91618d-0984-11e9-97ce-000c2987c3c2
Labels:         controller-uid=1a91618d-0984-11e9-97ce-000c2987c3c2
                job-name=pi
Annotations:    <none>
Parallelism:    1
Completions:    1
Start Time:     Wed, 26 Dec 2018 22:04:19 -0500
Pods Statuses:  1 Running / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  controller-uid=1a91618d-0984-11e9-97ce-000c2987c3c2
           job-name=pi
  Containers:
   pi:
    Image:      resouer/ubuntu-bc
    Port:       <none>
    Host Port:  <none>
    Command:
      sh
      -c
      echo 'scale=10000; 4*a(1)' | bc -l 
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  3m9s  job-controller  Created pod: pi-97rk
```
Job对象在创建后，它的Pod模板被自动加上了一个controller-uid=<随机字符串>这样的Label，而这个Job对象本身则被自动加上了这个Label对应的Selector,从而保证了Job与它所管理的Pod之间的匹配关系

Job Controller之所以要使用这种携带了UID的Label,就是为了避免不同Job对象所管理的Pod发生重合,需要注意的是,这种自动生成的Label对用户来说并不友好,所以不太适合推广到Deployment等长作业编排对象

接下来等到计算结束后,这个Pod就回进入Completed状态:
```shell
$ kubectl get pods
NAME                                READY     STATUS      RESTARTS   AGE
pi-97rk9                            0/1       Completed   0          4m
```
这也是在Pod模板中定义restartPolicy=Never的原因:**离线计算的Pod永远都不应该被重启**，而且restartPolicy在Job中也只允许被设置为Never和Onfailure。如果设置restartPolicy=Never,那么在Job启动失败时,Job Controller就回不断的尝试创建一个新的Pod:
```shell
$ kubectl get pods
NAME                                READY     STATUS              RESTARTS   AGE
pi-55h89                            0/1       ContainerCreating   0          2s
pi-tqbcz                            0/1       Error               0          5s
```
这个尝试不会无线进行下去,在Job API对象中定义的spec.backoffLimit字段里定义了重试次数为4,这个字段的默认值是6

如果设置restartPolicy=Onfailure,那么Job失败后,Job Controller就不会去尝试创建新的Pod，但是会不断的尝试重启Pod里的容器


此时查看一下这个Pod的日志,就可以看到计算得到的Pi值已经被打印出来了
```shell
$ kubectl logs pi-97rk9
3.141592653589793238462643383279...
```
在一个Job的Pod运行结束后，它会进入Completed状态。在Job的API对象里有一个spec.activeDeadlineSeconds字段可以设置最长运行时间
```yaml
spec:
  backoffLimit: 5
  activeDeadlineSeconds: 100
```
一旦运行超过了100秒,这个job的所有Pod都会被终止,并且可以在Pod的状态里看到终止的原因是: reason: DeadlineExceeded

##### Job Controller对并行作业的控制方法
在Job中,负责并行控制的参数有两个:
1. spec.parallelism, 它定义的是一个Job在任意期间最多可以启动多少个Pod同时运行;
2. spec.completions, 它定义的是Job至少要完成的Pod数目,即Job的最小完成数

例子:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  parallelism: 2
  completions: 4
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-pc
        imagePullPolicy: IfNotPresent
        command: ["sh", "-c","echo 'scale=5000; 4*a(1)' | bc -l"]
      restartPolicy: Never
  backoffLimit: 4
```
这样就指定了这个Job的最大并行数是2，最小的完成数是4
创建这个Job对象
```yaml
$ kubectl create -f job.yaml
```
使用kubectl get查看Job
```shell
$ kubectl get jobs/pi
NAME   COMPLETIONS   DURATION   AGE
pi     0/4           9s         9s
```
COMPLETIONS所描述的就是当前执行完成的Pod的进度

使用kubectl get pods 可以看到这个Job首先创建了两个并行运算的Pod
```shell
[root@k8s-m1 jobs]# kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
pi-b6swl                1/1     Running   0          4s
pi-lftr6                1/1     Running   0          4s
```
每当有一个Pod完成计算进入Completed状态时,就会有一个新的Pod被自动创建出来并且快速的从Pending状态进入ContainerCreating状态:
最终，后面创建的这两个Pod也完成了计算进入了Completed状态,这是由于所有的Pod均已经成功退出这个Job也就执行完成了,所以你会看到它COMPLETIONS变成了4/4
```shell
$ kubectl get pods
pi-b6swl                0/1     Completed   0          29m
pi-fhlxk                0/1     Completed   0          28m
pi-lftr6                0/1     Completed   0          29m
pi-n9v7x                0/1     Completed   0          28m

$ kubectl get job
NAME   COMPLETIONS   DURATION   AGE
pi     4/4           99s        29m
```

##### Job的工作原理











