---
layout:     post
title:      实习工作（五）
subtitle:   记录每周学习的内容
date:       2018-10-20
author:     Hxd
header-img: img/city.jpg
catalog: true
tags: 实习
---

## 星期一
今天的主要任务就是编写之前slaver的api使用接口，首先我们的目标是通过c7n-slaver在集群的每个节点上运行，并提供所需的功能，比如计算所在节点的物理存储（包括nfs）的总量和文件创建的权限，能够请求别的节点的slaver的能力并以此来判断节点的通讯是否互通。首先在一台机器上执行c7n，c7n通过daemonset在每个节点部署c7n-slaver，然后像c7n-slaver发送请求来验证集群的信息是否符合安装c7n的条件。这里需要判断的主要有存储卷和网络的功能，现在要添加能够向MySQL服务发送SQL并且执行的能力。而判断集群节点的存储就是直接使用go语言计算机器的存储，文件夹权限就是通过运行cmd或者sh命令来判断是否有权限在该挂载卷上创建文件。


## 星期二
目标已经确定，但是还要细磨代码。

## 星期三
添加判断集群节点的网络的功能。做法是首先部署一个ingres指向slaver（使用某个域名比如gitlab.a.com），c7n首先发一组域名与值的映射给slaver，slaver保存起来，然后再向slaver发送一个url，让slaver去访问该url，该url访问的也是slaver本身，但他以请求的key为值返回value，c7n的第二次请求和第一次所设置的value值相同，说明通过该域名（gitlab.a.com）可以正常访问。如图：
![](http://pbqgh436d.bkt.clouddn.com/18-10-20/78415340.jpg)

## 星期四
代码完成之后，发现有bug，每次get value的时总是会遇到`default-backends 404`，过一会儿又能验证成功。很迷！经过统计，该变化会搁100秒变化一次，由此可以断定问题出在集群内部的`nginx-ingress-controller`，可能是负载均衡的时候会分配到另一个pod，然后会出现404.但是我又没有部署其他的节点或者是其他的ingress。然后问了同事说，应该是ingress挂掉了才会出现`default-backends 404`.但是404过后又会回到正常状态，这就很其怪了。

## 星期五
继续研究昨天的bug。判断出所建的集群有问题。用于环境是在本机的虚拟机集群上搭建的，nfs也是本集群，所以可能是由于集群有问题造成的。导致某种轮咨。

## 总结
这周深入了解了ingress的作用和nginx-ingress-controller所做的事情。解决该问题还需要深入的了解集群内的组件，毕竟问了几个同事也没能很好的解决。只能自己做排除法了。

## 周天
历经两天的排查，终于得出问题了。原来是使用的domain域名解析捣的鬼：
```
/ # ping gitlab.h.vk.vu
PING gitlab.h.vk.vu (192.168.12.175): 56 data bytes
64 bytes from 192.168.12.175: seq=0 ttl=60 time=2.708 ms
64 bytes from 192.168.12.175: seq=1 ttl=60 time=2.268 ms
64 bytes from 192.168.12.175: seq=2 ttl=60 time=2.726 ms
^C
--- gitlab.h.vk.vu ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 2.268/2.567/2.726 ms
/ # ping gitlab.h.vk.vu
PING gitlab.h.vk.vu (192.168.33.100): 56 data bytes
64 bytes from 192.168.33.100: seq=0 ttl=64 time=0.040 ms
64 bytes from 192.168.33.100: seq=1 ttl=64 time=0.068 ms
^C
--- gitlab.h.vk.vu ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.040/0.054/0.068 ms
```

所以，slaver请求拿value的时候，会隔100次才出现404，就是gitlab.h.vk.vu的dns解析负载均衡搞的鬼。一会儿就会指向192.168.12.175，所以就出现`default-backends 404`.

好了，感谢梓棱哥不辞幸苦跑来加班！

