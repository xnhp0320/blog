---
title: Maglev一致性哈希（补遗）
date: 2016-06-28 09:41:57
tags:
---

上篇[blog](http://xnhp0320.github.io/2016/06/26/Maglev-consistent-hash/)介绍的一致性哈希算法满足minimal disruption和load balancing两个特性。其实有更简单的方法满足这两个条件。我们知道，构造哈希函数本质是填数组（fill the entry[])，那么要想得到load balancing的数组，只需要后端backend依次填入数组就肯定是balancing的。比如

> entry[0] = B0
> 
> entry[1] = B1
> 
> entry[2] = B2
> 
> entry[3] = B0
> 
> ...

当需要删除一个backend时，将对一个的表项全部remove，然后这些空出来的表项可以被均分给剩下的backends。同理，新增一个backend也可以采用类似方法处理。从这个角度来看permutation table本来也没存在的必要。

我后来写信给Google的人，他们给出一个非常有道理的回复：

>You are correct that with one load balancer, the technique you described would be simple and work optimally.  However, the lookup table would be dependent on history.  Two (or more) load balancers that started at different times or saw events in different orders would end up with different tables, and traffic that moved from one to another would not get the same assignment.  Our technique solves this problem without the need for explicit communication between the load balancers to synchronize backend tables.
>

也就是说，permuation table的意义在于无论backend以什么样的顺序删除，最终在load balancers上的lookup table都是一致的，同时也避免了同步开销。

另外介绍一个[博客](https://blog.acolyer.org/2016/03/21/maglev-a-fast-and-reliable-software-network-load-balancer/)。这个博客对此也做了详细讨论。感慨一下，觉得Google的人真挺nice的，我本来也没太期望他们能回信。
