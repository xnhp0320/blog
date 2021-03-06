---
title: '对ip_vs中rcu锁和timer之间的竞态关系的分析'
date: 2016-10-01 11:46:42
tags:
---

对内核ip_vs模块中的流表之间的竞态关系做了如下分析。

- RCU锁只能保护在read_lock/unlock之间对cp的引用，而不能保护在读锁之外的区域。因此，如果有人在读锁之外继续使用cp指针，则只能使用引用计数保护。

- 内核2.6.32采用读写锁，因此在读写之间本身是互斥的，所以直接使用inc和dec。而使用RCU锁，读写可能同时出现，因此必须用原子操作互斥。因此get使用atomic\_inc\_not\_zero，put因为跟get配对使用，直接减1，而timer中使用atomic\_cmpxchge。因为原子变量互斥，效果和读写锁类似。因为这种互斥性，当cp从hash表中删除，local\_in和out的勾子函数不可能持有cp。只有使用drop\_randomentry的线程可能会持有cp。因此个人感觉使用RCU锁似乎没有比用读写锁占到更多的便宜？


- timer中，如果引用计数为0，是否一定没有对cp的引用？不一定。可能此时当前这个timer已经被重新激活，在未来某个时候expire，相当于此时cp已经又被一个timer所引用。在当前timer\_expire函数中要避免这一点，因此需要在确定释放cp前，调用del\_timer。内核2.6.32和3.10都这么做了。

- 因而当timer处于pending状态时，是否call\_back一定没有被执行？不是，因为在call\_back刚开始被执行时，有其他线程把timer重新激活了。此时timer处于pending状态，而且timer的callback还在执行。这是timer最复杂的地方。

- 那么是否同一个timer的callback函数会在不同的两个核上运行？不能，因为每次mod_timer时，会检查timer的base—>running\_timer，如果不等于timer，说明timer callback没有运行，那么可以把timer迁移到mod\_timer的核，否则，当timer callback在运行的时候，则只能把timer挂载到之前挂载的核。因此一个timer的callback不可能会在两个核上同时运行。


- timer只有一个，他要么就是pending要么就是expire或者即将调用callback然后expire。当在timer中引用计数为0，并且已经del_timer，此时，是否有其他线程能够重新mod\_timer? 不可能，其他线程获取cp时，要么引用计数大于0（不满足，因为在get时，引用计数不为0才能增加引用计数并返回cp指针），要么mod\_timer时，需要timer是pending状态（比如expire\_now函数），此时timer刚被del\_timer，因此也不满足，因此，此时可以安全的释放cp。


- 上述条件实际上暗示了，并不一定需要获得引用计数才能对timer进行修改。现在解释为何expire\_now函数不需要引用计数的保护。当mod\_timer在read\_lock/unlcok保护区时，cp不会被释放，因此可以安全的访问cp->timer。而此时，如果timer处于pending状态，有两种可能，要么callback没运行，那么就改掉。要么callback正在执行，此时如果mod\_timer会有导致timer被重新激活，并只能在当前核等待callback运行完之后expire，因此只需要保证在callback函数中del\_timer之前完成调用mod\_timer调用便是安全的。实际上，使用del\_timer之后，mod\_timer\_pending一定会失败，因为如果之前mod，timer会被删掉，如果之后执行mod，此时timer已经被删掉了，不是pending状态，同样会失败。


