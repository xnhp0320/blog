---
title: Maglev中的一致性哈希算法
date: 2016-06-26 10:21:13
tags:
---


[Maglev](http://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/44824.pdf)是Gogole在NSDI 2016上公布的其内部使用的软件负载均衡系统。论文关于系统架构已经说得很清楚了，除了其中一致性哈希算法部分，看着略犯晕。本文简要介绍下这个系统中的一致性哈希算法。

## 负载均衡系统
负载均衡系统基本上只要是做互联网的公司都要用到。其原理，我感觉就是个大哈希表：对外网（WAN）只宣告几个虚拟IP地址（VIP），而把客户端的流量通过哈希表做连接跟踪，将流量负载均衡到后端（backend）大量的服务器集群上。基本上各大互联网公司都有一套自己的负载均衡的系统。现在托云计算的福，很多小的公司也可以直接使用其云计算的IaaS功能实现的负载均衡器。

关于负载均衡系统，国内有个著名的开源人物[章文嵩](http://baike.baidu.com/view/3160926.htm)开发了[LVS](http://www.linuxvirtualserver.org/zh/lvs1.html)，想必做网络开发的都基本上耳熟能详。章文嵩在2002年的时候做的LVS，当时是基于内核做的开发。当年的网络开发的潮流就是想要高性能就得玩内核。时过境迁，当DPDK这种技术已经开始普及，bypass-kernel成为新潮流。所以，似乎目前有些公司开始研发全用户态的负载均衡系统，比如[Vortex](https://segmentfault.com/a/1190000004713123)。

Google的Maglev应该做的时候还没有DPDK，但看论文也是bypass-kernel。目前网络已经一坨坨的bypass kernel了，[存储](https://github.com/spdk/spdk)也在路上。所以说，bypass-kernel 已经是 the new sexy了么？


## 一致性哈希

[一致性哈希](http://blog.csdn.net/cywosp/article/details/23397179)在网络、文件系统、中间件中都有用到。其主要解决当后端服务器的个数出现变化的时候，如何通过哈希机制继续保证剩下的服务器之间负载均衡。负载均衡系统的后端服务器集群可能时常面临服务器负载过高而挂起、服务器系统升级、服务器容量升级等原因而产生的服务器数量的变动，这个时候一致性哈希就有了用武之地。比如一个服务器A过载挂了，之前被分配到A的流都需要被重新分配（re-shuffle），一致性哈希需要保证被重新分配后的流能够尽量均衡的分配到其他的服务器上，而不至于造成新的过载。比如新增一个服务器B，一致性哈希需要保证对已有分配结果进行最小限度的修改，同时要保证分配后的均匀性。


## Maglev的一致性哈希算法
首先说说一致性哈希算法的本质：说白了，就是一个哈希函数（注意，不是哈希表)，输入是一个流，输出是这个流所对应的后端服务器。

>Hash(flow1) = 服务器A
>
>Hash(flow2) = 服务器B ...

当有数据包进入，Maglev首先check本地的连接跟踪表，看看该数据包是否属于任何一个已有的连接，如果连接已经建立，则直接将数据包发到该连接对应的服务器上去。如果连接没有建立，此时就需要一致性哈希函数选择一个后端服务器了。

一致性哈希函数可以通过一个数组Entry[]的方式构造，假设数组长度为M，那么对应流F的后端服务器为：

>Entry[Hash(F) % M]

Maglev的一致性哈希算法本质上设计算法让每个后端按照一定的规则去填满数组Entry[]中的empty slot，确保所构造出来的数组Entry[]的元素中，所有后端服务器出现的次数尽可能相等。这个数组Entry[]就是最终所以使用的一致性哈希函数。

为了设计填充规则，Maglev首先给每个后端服务器i设计了一个permutation表。permutation表存储一个0到M-1的随机排列，比如当Entry数组长度M=7时，permutation表内容可以为：

>3 0 4 1 5 2 6 

这个permutation表实际上是用来构建抢占规则的。
对于每个后端服务器i，Maglev通过以下方式生成permutation[i]:

![](https://res.cloudinary.com/xnhp0320/image/upload/v1547279042/blog/WX20190112-154234_2x.png)

实际上，上图中的*skip*，*offset*还有哈希后端服务器名字都只是一种随机化的方法，用于构造permutation表而已。

紧接着，使用以下算法构造Entry表。

![算法描述](https://res.cloudinary.com/xnhp0320/image/upload/v1547279159/blog/WX20190112-154540_2x.png)

这个算法实际上就是，对于每个服务器后端i，通过其permuation[i]表所设立的规则，在entry数组中寻找empty slot，当发现被slot被填时，不停使用next[i]+1的方式跳到entry[permuation[i][next[i]]中probe是否有empty slot，一旦发现有empty slot则把empty slot分配给自己，即算法中entry[c] = i。通过这样的方式，每个服务器都有机会去填一个空位，当最终填满entry[]表时，所构造出entry[]数组一定是均衡的。

假设M=7，即数组entry[]的长度为7，有三个后端服务器B0、B1、B2，其permuation表如下图所示。根据上述算法，首先entry[]表是空的，B0来填，根据其permuation表，首先看看3的位置是不是空，发现是，于是entry[3] = B0。然后B1来填entry[]，于是entry[0]=B1。最后B2来填表，他首先看entry[3]，发现已经填了B0，于是看下一个位置，4，于是entry[4]=B2。

![](https://res.cloudinary.com/xnhp0320/image/upload/v1547279200/blog/WX20190112-154630_2x.png)

上图同时显示了，当B1被remove之后，所构造出来的entry表，可以发现后端服务器出现的次数是完全一样的，达到了均衡的目的。


