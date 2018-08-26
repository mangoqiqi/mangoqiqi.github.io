---
layout:     post
title:      实习工作（一）
subtitle:   记录每周学习的内容
date:       2018-08-26
author:     Hxd
header-img: img/city.jpg
catalog: true
tags: 实习
---


## 星期一

kubectl命令：`kubectl cordon runner` 使runner不分配pod

Choerodon使用：

获得限权角色后：
- 创建了一个应用，编码为c7n，名称猪齿鱼修改器，模板不选择
- 之后会生成一个程序模板
- git克隆下来
- 安装golang开环境
- 在用用户名下的GO目录下创建应用文件夹并把克隆下来的复制过来：`~/go/src/github.com/choerodon/c7n`
- 安装go的依赖包然后进行开发
- 在安装go依赖的时候使用`go install`命令即可
- 这里要注意的是git的分支，不要push错误了，还有要注意commit的格式和方法，这里复习一下：

[IMP] 提升改善正在开发或者已经实现的功能

[FIX] 修正BUG

[REF] 重构一个功能，对功能重写

[ADD] 添加实现新功能

[REM] 删除不需要的文件

整个开发过程主要围绕k8s的api来完成，调用api获取client的数据并处理，其实难度就在git的基础使用、功能的做法和思路，如何写出高效而让别人易懂的代码应该是关注的重点，包括log记录和commit内容等等。还有重要的一点是基本用到的英语单词应该要熟透并表达清楚。

golang语言相关：

函数名首字母大写表示共有的，就像public修饰一样，不然只能在本文件内被调用。

变量格式声明为`var name [类型]`


#### k8s相关的内容

主要组件：

**Etcd**：是一个基于key-value的分布式数据库，类似于Redis，用于服务发现、共享配置以及一致性保障，他它的功能主要是1.基于key-value的存储  2.监听机制  3.key的过期及续约机制，用于监控和服务发现  4.原子CAS和CAD，用于分布式锁和leader选举
其实就把它当作是一个分布式非关系型数据库就可以了

**kube-apiserver**：今天写的go都是围绕着这个api写的，它提供集群管理的 REST API 接口，包括认证授权、数据校验以及集群状态变更等提供其他模块之间的数据交互和通信的枢纽（其他模块通过 API Server 查询或修改数据，只有 API Server 才直接操作 etcd）调用api的方式有三种：1.程序里使用api函数  2.ui界面的工具 3.cli命令行界面

**kube-controller-manage**：Controller Manager由kube-controller-manager和cloud-controller-manager组成，是Kubernetes的大脑，它通过apiserver监控整个集群的状态，并确保集群处于预期的工作状态。分别有很多controller组成

**kube-scheduler**：资源调度，kube-scheduler 负责分配调度 Pod 到集群内的节点上，它监听 kube-apiserver，查询还未分配 Node 的 Pod，然后根据调度策略为这些 Pod 分配节点（更新 Pod 的 NodeName 字段）比如使用`kubectl cordon runner`之后，runner节点上会显示`Ready,SchedulingDisabled`即不可分配pod

**Kubelet**：每个节点上都运行一个 kubelet 服务进程，默认监听 10250 端口，接收并执行 master 发来的指令，管理 Pod 及 Pod 中的容器。每个 kubelet 进程会在 API Server 上注册节点自身信息，定期向 master 节点汇报节点的资源使用情况，并通过 cAdvisor 监控节点和容器的资源。
**Container runtime**容器运行时（Container Runtime）是 Kubernetes 最重要的组件之一，负责真正管理镜像和容器的生命周期。Kubelet 通过 Container Runtime Interface (CRI) 与容器运行时交互，以管理镜像和容器。

**kube-proxy**每台机器上都运行一个 kube-proxy 服务，它监听 API server 中 service 和 endpoint 的变化情况，并通过 iptables 等来为服务配置负载均衡（仅支持 TCP 和 UDP）。
kube-proxy 可以直接运行在物理机上，也可以以 static pod 或者 daemonset 的方式运行，他其实改变的是iptables的转发功能

## 星期二

kibana的使用、创建日志应用并部署、搭建三个节点的k8s集群、通过url定位出错的pod、kunectl命令的用法

首先要创建一个graph，再往里面添加各种图形，需要添加变量，变量字段可以通过https://prometheus.choerodon.com.cn来获取，添加之后再配置规则，主要有两个函数`irate`和`delta`，irate计算的是给定时间窗口内的每秒瞬时增加速率，deta计算范围向量中每个时间系列元素的第一个和最后一个值之间的差异v，返回具有给定增量和等效标签的即时向量。delta被外推以覆盖范围向量选择器中指定的全时间范围，因此即使样本值都是整数，也可以获得非整数结果。比如：`irate(http_server_requests_seconds_count{pod_name="$instance"}[25s])`代表每秒的请求数量，`http_server_requests_seconds_count`这个字段是累积的请求数量，25s的意思是从0~25s内的时间。
再比如`delta(http_server_requests_seconds_count{pod_name="$instance"}[25s])`代表的是请求的差值。

#### 日志管理

主要的架构如图所示:

![](http://pbqgh436d.bkt.clouddn.com/18-8-21/78054037.jpg)

fluent-bit是记录日志的组件，每个集群都会运行这个服务，日志通过fluentd被转发至es服务器集群（想象它只有一个实体），然后再由可视化平台操作展现数据。

今天主要部署的应用由es和kibana。然后重点了解fluent-bit的用法：

fb的架构：

![](http://pbqgh436d.bkt.clouddn.com/18-8-21/86026463.jpg)
**input**：就是一系列的日志记录输入

**parser**：正则表达式，用来过滤的

**filter**：过滤器

**buffer**：缓冲区

**routing**：分发路由

**output**：输出


### K8s

URI用于pod的资源定位

etcd保存整个集群的状态

apiservrer提供入口

controller负责维护集群的状态

schedule负责资源的调度

kubectl负责维护容器的生命周期，包括Valume（CVI）和网络（CNI）

container runtime负责镜像管理以及pod和容器的真正运行（CRI）

kube-proxy负责为service提供cluster内部的服务发现和负载均衡

kube-dns负责为整个集群提供DNS服务

ingress controller为服务提供外网入口

Heapster提供资源监控

Dashboard提供GUI

Federation提供跨可用区的集群

Fluentd-elasticsearch 提供集群日志采集、存储与查询


## 星期三
#### Prometheus config配置的基本概念

监控服务，用于监控主机资源和容器资源，每个节点运行cadvisor，然后Prometheus监控工具通过node-exporter来收集主机或者node的性能指标，Prometheus首先去读配置文件，里面记录了去哪取指标，如何取等信息。也可以通过扫描label来发现装有node-exporter的node。

```
global:        #全局的配置，优先值比较下面的低
  scrape_interval: 10s   #抓取的时间间隔
  scrape_timeout: 10s    #抓取的超时时间
  evaluation_interval: 10s   #执行的时间间隔
alerting:    #以下先不看
  alertmanagers:
  - static_configs:
    - targets:
      - alertmanager-choerodon-monitoring-369d9:9093
    scheme: http
    path_prefix: /
    timeout: 10s
rule_files:
- /etc/prometheus-rules/*.rules
scrape_configs:             #常用的抓取配置文件
- job_name: kubernetes-pod  #就是一个job的名字
  scrape_interval: 10s      #抓取的时间间隔优先于global
  scrape_timeout: 10s       #抓取的超时时间
  metrics_path: /metrics    #抓取的指标路径
  scheme: http              #协议格式http
  kubernetes_sd_configs:    #kubernetes服务发现
  - api_server: null        #
    role: pod               #指定抓取的对象为pod，可选的值有pod,node,service,ingress,endpoints
    namespaces:             #
      names: []             #
  relabel_configs:          #根据规则替换一些标签数据
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]#是否声明了prometheus_io_scrape，是，则替换掉
    separator: ;
    regex: "true"
    replacement: $1  #替换为上面声明的
    action: keep     #这里理解为，如果声明了scrape，就保持住，就是不需要监控全部的pod
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scheme]#声明替换协议格式
    separator: ;
    regex: (https?)
    target_label: __scheme__
    replacement: $1
    action: replace
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]#声明替换socket地址，比如一个label声明了多个端口，那该去哪个端口那监控数据呢，这里比如指定了某个端口，就用该端口替换掉，从而能通过该地址访问监控数据
    separator: ;                 #值为空，以;分隔
    regex: (.+)(?::\d+);(\d+)    #正则表达，替换端口号
    target_label: __address__
    replacement: $1:$2           #就是替换 _address_ 里的端口号为prometheus_io_port
    action: replace              #替换掉
  - source_labels: [__meta_kubernetes_pod_name]  #pod对象的名字
    separator: ;
    regex: (.*)
    target_label: pod_name
    replacement: $1
    action: replace
  - source_labels: [__metrics_path__]  #替换指标的地址
    separator: ;
    regex: (.*)
    target_label: cluster
    replacement: choerodon-prod
    action: replace
```


## 星期四

修改fluentd的match标签，解决不能全部匹配问题。使用rewrite-tag 插件。
设置revert为 true


## 星期五
修改fluent-bit的out_es插件，使其可以直接定向到elas服务器，从而减少了fluentd组件。主要是通过修改index的指向。

```
[INPUT]
    Name mem
    Tag  mem.local

[OUTPUT]
    Name  stdout
    Match *

[FILTER]
    Name record_modifier
    Match msg
    Record msg $re
    
```


```
{
	"index": {
		"_index": "my_index",    ### ->api
		"_type": "my_type"       #
	}
} {
	"@timestamp": "2018-08-23T06:34:55.679Z",
	"cpu_p": 98.000000,
	"user_p": 81.000000,
	"system_p": 17.000000,
	"cpu0_p_cpu": 98.000000,
	"cpu0_p_user": 81.000000,
	"cpu0_p_system": 17.000000
}

bin/fluent-bit -i mem -o es://192.168.99.100:9200/test_index1/test_type1
```

## 周末

- 学golang
- 看一下grpc

总结下吧，总的来说都是在学习新的知识，导师分配任务的时候感觉摸不着头脑，智商不够用了😂，项目需要用golang，对于主要使用c/c++和java的我来说入门golang还是蛮轻松的，感觉就是C，java，python（切片）的结合体。可以快速开发。

grpc，先说rpc吧，就是本地远程调用服务端的函数资源啥的。grpc就是以[protocol buffer](https://developers.google.com/protocol-buffers/docs/overview)作为序列化组件的rpc，pb是Google家的开源工具--一种与语言无关，平台无关，可扩展的序列化结构化数据的方法，用于通信协议，数据存储等。

grpc有四中通信（调用方式），即简单一元式rpc，服务端流式rpc，客户端流式rpc，双向流式rpc。比如发微信，假设一端是客户端，另一端是服务端，一元式rpc就是客户端发个请求，服务端回应一个请求（比如正常的函数调用）。服务端流式就是，客户端接收到服务端多个消息。客户端流就是客户端发送多个消息，双向流就是他们的结合，而且互不影响