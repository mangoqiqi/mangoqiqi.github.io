---
layout:     post
title:      k8s调度器
subtitle:   猪齿鱼架构所感
date:       2019-01-10
author:     Hxd
header-img: img/city.jpg
catalog: true
tags: 
  - k8s
  - 调度器
---

## K8S Scheduling
Kubernetes作为一个容器编排调度引擎，资源调度是它的最基本也是最重要的功能。
Kubernetes的资源对象有Pod、Service、Job等等，其中Deployment、DaemonSet等对象本身有定义了Pod运行的默认调度策略，也可以根据资源的不同属性打上不同的标签以制定一些高级的调度策略。

### 使用label selectors调度Pods
Lable是附着在K8S对象（如Pod、Service等）上的键值对。它可以在创建对象的时候指定，也可以在对象创建后随时指定。Kubernetes最终将对labels最终索引和反向索引用来优化查询和watch，在UI和命令行中会对它们排序。通俗的说，就是为K8S对象打上各种标签，方便选择和调度。而Lable就相当于“打标签”的操作，“选择”操作则由label selectors来进行。由于Lable不是唯一的，很多K8S对象可能有相同的Lable，通过label selectors，就可以指定一个资源对象的集合，并进行操作。比如使用Service通过label selector将同一类型的Pod作为一个服务暴露出来，以便其他Pod能通过网络访问到。
首先将一组同一类型的Pod打上相同的标签：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
    labels:
      app: web
      env: dev
spec:
  containers:
  - name: bar
    image: nginx
```
labels中指定了该pod的app=web，env=dev。然后Service通过selectors将含有对应的标签的pod选中。
```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: web
    env: dev
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
```
Lable selector有两种类型，一种是equality-based，就像上面所写的一样，可以使用=、==、!=操作符，可以使用逗号分隔多个表达式。另一种是set-based，可以使用in、notin、!操作符，另外还可以没有操作符，直接写出某个label的key，表示过滤有某个key的object而不管该key的value是何值。使用set-based可以实现更高级的选择策略。

#### Nodelable和Nodeselector

Nodelable是定义在节点Node上的标签，它可以在节点创建后随时添加。就相当于Pod的lables，但是它的对象是节点。除了selector，PodSpec还有一个区域叫做nodeSelector。这个字段用于鉴别Pod是否有资格运行在Node上，只有Node的标签与Pod的Nodeselector匹配才会为Pod分配Node执行。为Node加上标签可以把pod定位到特定节点或节点组上。可以用于确保特定pod仅在具有特定隔离、安全性或监管属性的节点上运行。

```
$ kubectl create -f https://k8s.io/examples/pods/pod-nginx.yaml
pod/nginx created

$ kubectl get po
NAME      READY     STATUS    RESTARTS   AGE
nginx     0/1       Pending   0          8s

$ kubectl get pods -o wide
NAME      READY     STATUS    RESTARTS   AGE       IP        NODE
nginx     0/1       Pending   0          27s       <none>    <none>
```
此时由于没有节点满足以上pod的Nodeselector的要求，所以pod不能被分配，从而一直pending。为该节点添加对应的标签以分配node。
```
$ kubectl label nodes minikube disktype=ssd
node/minikube labeled

$ kubectl get po -o wide
NAME      READY     STATUS              RESTARTS   AGE       IP        NODE
nginx     0/1       ContainerCreating   0          6m        <none>    minikube

$ kubectl get po
NAME      READY     STATUS    RESTARTS   AGE
nginx     1/1       Running   0          10m
```
查看node标签：
```
$kubectl get no -o yaml
...
labels:
      beta.kubernetes.io/arch: amd64
      beta.kubernetes.io/os: linux
      disktype: ssd
      kubernetes.io/hostname: minikube
...
```

#### Affinity和Anti-affinity
nodeSelector提供了一种非常简单的方法，可以将pod限制为具有特定标签的节点。而更为强大的表达约束类型则可以由Affinity和Anti-affinity来配置。即亲和性与反亲和性的设置。亲和性和反亲和性包括两种类型：节点（反）亲和性与Pod（反）亲和性。Node affinity与NodeSelector很相似，它允许你根据节点上的标签限制你的pod可以在哪些节点上进行调度。目前有两种类型的节点关联，称为requiredDuringSchedulingIgnoredDuringExecution和 preferredDuringSchedulingIgnoredDuringExecution。可以将它们分别视为“硬规则”和“软规则”，前者指定了要将 pod调度到节点上必须满足的规则，而后者指定调度程序将尝试强制但不保证的首选项。名称中的“IgnoredDuringExecution”部分意味着，类似于nodeSelector工作方式，如果节点上的标签在运行时更改，不再满足pod上的关联性规则，则pod仍将继续在该节点上运行。Pod affinity强调的是同一个节点中Pod之间的亲和力。可以根据已在节点上运行的pod上的标签来约束pod可以调度哪些节点上。比如希望运行该Pod到某个已经运行了Pod标签为app=webserver的节点上，就可以使用Pod affinity来表达这一需求。目前有两种类型Pod亲和力和反亲和力，称为requiredDuringSchedulingIgnoredDuringExecution以及 preferredDuringSchedulingIgnoredDuringExecution，其中表示“硬规则”与“软规则”的要求。类似于Node affinity，IgnoredDuringExecution部分表示如果在Pod运行期间改变了Pod标签导致亲和性不满足以上规则，则pod仍将继续在该节点上运行。无论是Selector还是Affinity，都是基于Pod或者Node的标签来表达约束类型。从而让调度器按照约束规则来调度Pod运行在合理的节点上。

新建node-affinity-pod.yaml为以下内容：
```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```
其中亲和性定义为，该pod只能放置在一个含有键为kubernetes.io/e2e-az-name并且值为e2e-az1或者e2e-az2标签的节点上。此外，在满足该标准的节点中，具有其键为another-node-label-key且值为another-node-label-value的标签的节点应该是优选的。

创建pod：
```
$ kubectl create -f node-affinity-pod.yaml
pod/with-node-affinity created

$ kubectl get po
NAME                 READY     STATUS    RESTARTS   AGE
with-node-affinity   0/1       Pending   0          6s
```
此时由于没有节点满足以上pod亲和性要求，所以pod不能被分配，从而一直pending。为节点添加标签以满足pod亲和性。
```
$ kubectl label nodes minikube kubernetes.io/e2e-az-name=e2e-az1
node/minikube labeled

$ kubectl get po
NAME                 READY     STATUS    RESTARTS   AGE
with-node-affinity   1/1       Running   0          1m
```
这时候pod可以被调度到节点上。

### 理解DaemonSets和Kubernetes scheduler

在Kubernetes中有一个kube-scheduler组件，该组件运行在master节点上，它主要负责将pod调度到节点上。kube-scheduler监听kube-apiserver中是否有还未调度到node上的pod（即Spec.NodeName为空的Pod），再通过特定的算法为pod指定分派node运行。如果分配失败，则将该pod放置调度队列尾部以重新调度。调度主要分为几个部分：首先是预选过程，过滤不满足Pod要求的节点。然后是优选过程，对通过要求的节点进行优先级排序，最后选择优先级最高的节点分配。其中涉及到的两个关键点是过滤和优先级评定的算法。调度器使用一组规则过滤不符合要求的节点，其中包括设置了资源的request和指定了Nodename或者其他亲和性设置等等。优先级评定将过滤得到的节点列表进行打分，调度器考虑一些整体的优化策略，比如将Deployment控制的多个副本集分配到不同节点上等等。DaemonSet是一种控制器，它确保在一些或全部Node上都运行一个Pod的副本。这些Pod就相当于守护进程一样不期望被终止。当有Node加入集群时，也会为他们新增一个Pod。当有Node从集群移除时，对应的Pod也会被回收。当删除DaemonSet时将会删除它创建的所有Pod。一般情况下，Pod运行在哪个节点上是由Kubernates调度器选择的。但是由Daemon Controller创建的Pod在创建时已经确定了在哪个节点上运行（pod在创建的时候.spec.nodeName字段就指定了， 因此会被scheduler忽略），所以即使调度器还没有启动Daemon Controller创建的Pod仍然可以被分配node。

### 资源的限制影响Pod的调度

当创建一个Pod时，可 以指定每个Container需要和限制多少CPU和内存。requests字段用于说明运行该容器所需的最少资源。当调度器开始调度该Pod时，调度程序确保对于每种资源类型，计划容器的资源请求总和必须小于节点的容量才能分配该节点运行Pod。limits字段用于限制容器运行时所获得的最大资源。如果该容器超出其内存限制，则可能被终止。 如果该容器可以重新启动，kubelet会将它从新启动，就像其他运行时错误一样。如果容器超出了其内存请求，当节点的内存不足时该容器可能被驱逐到其他节点。而对CPU的请求或限制则不会影响容器的生命周期但是会影响Pod的调度。调度器找不到合适的节点运行Pod时，就会产生调度失败事件，调度器会将Pod放置调度队列以循环调度，直到调度完成。

### 运行多个schedulers

以上所说的调度器是Kubernetes的默认调度器，如果默认调度程序不适用于我们的需要，还可以实现自定义的调度器并且多个调度器同时运行。制作完成自定义调度程序镜像后就可以部署一个调度器。想要让Pod受新增的调度器调度，只需要在Pod的配置信息中设置了spec.schedulerName选项，即可指定一个调度器来调度该Pod，若该调度器不存在，则该Pod将不被调度并且永远处于“Pending”状态。如果Pod没有指定spec.schedulerName则会使用默认的调度器。

### 手动调度Pod

使用static pod可以实现不使调度器调度一个pod。静态Pod不通过apiserver，它直接由kubelet监听并重启。当kubelet启动时指定静态pod的配置文件就可以创建一个静态pod。有两种方法创建：kubelet --pod-manifest-path=DIR，这里的DIR要放置static pod的编排文件的目录。 把static pod的编排文件放到此目录下，kubelet可以监听到变化，并根据编排文件创建pod。或者指定--manifest-url=URL，kubelet会从这个URL下载编排文件，并创建pod。当使用kubectl命令直接删除静态pod时，过了一段时间又被kubelet重启，这是因为启动时kubelet会去监听指定的静态pod的配置文件与etcd中pod的信息是否符合，若pod被删除了即etcd中不存在该pod的信息，kubelet就会重新启动该静态pod并将信息写入etcd。想要完全删除静态pod，应该移除static pod的编排文件即可。
使用静态pod是一种手动调度Pod的方法，但是更为安全可靠的方式是使用DaemonSet。DeamonSet中可以指定Pod的nodeName，从而绕过调度器直接部署在对应的节点上。

### 配置Kubernetes scheduler

如果需要配置一些高级的调度策略以满足我们的需要，可以修改默认调度程序的配置文件。kube-scheduler在启动的时候可以通过--policy-config-file参数来指定调度策略文件，我们可以根据我们自己的需要来组装Predicates和Priority函数。选择不同的过滤函数和优先级函数。调整控制优先级函数的权重和过滤函数的顺序都会影响调度结果。
官方的Policy文件如下：
```
{
    "kind" : "Policy",
    "apiVersion" : "v1",
    "predicates" : [
        {"name" : "PodFitsHostPorts"},
        {"name" : "PodFitsResources"},
        {"name" : "NoDiskConflict"},
        {"name" : "NoVolumeZoneConflict"},
        {"name" : "MatchNodeSelector"},
        {"name" : "HostName"}
    ],
    "priorities" : [
        {"name" : "LeastRequestedPriority", "weight" : 1},
        {"name" : "BalancedResourceAllocation", "weight" : 1},
        {"name" : "ServiceSpreadingPriority", "weight" : 1},
        {"name" : "EqualPriority", "weight" : 1}
    ]
}
```
其中predicates区域是调度的预选阶段所需要的过滤算法。priorities区域是优选阶段的评分算法。