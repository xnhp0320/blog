+++
title = 'Systemtap'
date = 2024-01-13T15:49:06+08:00
draft = false
+++


关于Systemtap，网上有很多教程，比较推荐的是[这里](https://sourceware.org/systemtap/langref.pdf)和[这里](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/pdf/systemtap_beginners_guide/red_hat_enterprise_linux-7-systemtap_beginners_guide-en-us.pdf)。但每次读下来，都觉得没讲清楚。这些教程对于一般使用算是够了，但是对一些复杂任务来说还是略浅。以下记录下一些限制，可能有更好的解决的方案，欢迎提建议。

## 字典的局限性
Systemtap的语言比较有特色的是容器只有字典，数组可以被认为是一种特殊的字典，这一点有点类似于lua：

```
a[1]=1234
```
这个不是数组是字典。

但是我找遍了systemtap langref都没有找到初始化的构造方式，这导致我无法构造一些常量数组，类似以下这种：

```
a[] = { 1,2,3,4}
```
这导致systmetap的语言还是一个半成品。对于某些复杂的需求，比如我经常要监控DPDK的轮询线程的某些数据，但是这些线程的tid号有时候并非连续的，我只能用以下这种简单粗暴方式

```
t = tid()
if (t != X1 && t != X2 ...) next
```
的方式进行过滤。另外这个`next`比较类似perl语言，整个systemtap都比较像perl的语法，可能发明人确实受perl的影响比较大。

字典的第二个局限性，在于无法定义nested字典，比如：

```
a[1] = {}
```
就类似于`a[1]`是一个新的字典，Systemtap的解决思路是，key可以多维，比如
```
a[1,2,3] = 1
```
但是这个并不能完全替代nested字典，比如你per-thread收集了一些数据，但是当你要清除per thread的数据时，你发现，你只能一个一个的删，like

```
delete a[1,2,3]
```
而不能一把删掉key == 1的全部数据：
```
delete a[1,...]
```
这块我尚不知道解法。

因此我的解决方案式，使用一个timer，每隔一段时间

```
probe timer.s(1) {
	delete a
}
```

，即把`a`全部清除掉。



## 对Aggregate的理解

Aggregate其实就是一个简单的，定制的数据结构，我一开始以为他会存储数据，但实际上他不会。如果用C++的方式理解Aggregate的工作原理，大意是以下的几行代码：

```c++
template <typename T>
class Aggregate {
	void operator <<< (T v) {
		...
	}
	T max() {
		...
	}
	T count() {
		...
	}
	T min() {
		...
	}
	T max() {
		...
	}
	
}
```

是否瞬间就理解了？其实很简单。就是他在语言里引入了一种原生数据结构，但是不支持元素遍历。这些东西在他的tutorial里，讲的很fancy。其实还不如一开始老老实实讲清楚。

##Probe的并发性
多个Probe在多个核出现时，Systemtap对全局变量的数据类型是如何保护的？文档里没有明确讲清楚，只是在某些地方作了一些暗示，比如在这个[tutorial](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/pdf/systemtap_beginners_guide/red_hat_enterprise_linux-7-systemtap_beginners_guide-en-us.pdf)：

>What about locking? If multiple probes seek conflicting locks on the same global variables, one or more of them will time out, and be aborted. Such events are tallied as “skipped” probes, and a count is displayed at session end. A configurable number of skipped probes can trigger an abort of the session.

这里暗示了，对于global变量的访问是加锁的，但并没有详细的描述是怎么加的锁，是每次访问的时候加锁呢，还是在probe一开始就加锁。要搞清楚这个问题，最简单的方式是直接把编译产生的.c文件打印出来：

```
stap xxx.stp -p3 > xxx.c
```
然后观察他到底如何实现的。不用怕，其实他这个翻译的结果整体上看还是蛮容易看懂的。

在probe点的开头，可以看到如下代码：

```C
static void probe_6341 (struct context * __restrict__ c) {
  __label__ deref_fault;
  __label__ out;
  static const struct stp_probe_lock locks[] = {
    {
      .lock = global_lock(s___global_in_compose),
      .write_p = 1,
      #ifdef STP_TIMING
      .skipped = global_skipped(s___global_in_compose),
      .contention = global_contended(s___global_in_compose),
      #endif
    },
    {
      .lock = global_lock(s___global_stop),
      .write_p = 1,
      #ifdef STP_TIMING
      .skipped = global_skipped(s___global_stop),
      .contention = global_contended(s___global_stop),
      #endif
    },
  };
```

这里似乎对某些全局锁作了引用。这里的`global_lock`似乎是某种引用。结合我自己的的源文件，因为我定义了两个全局变量，一个是`stop`，一个是`in_compose`，那么可以大概猜到，stap为这两个变量生成了两把锁。

为了验证想法，可以去github上搜索代码。github提供了代码的简单搜索，对于一些dirty and quick的活，确实可以直接在github上搜下。在[这里](https://github.com/groleo/systemtap/blob/bd23cd846d787b59183d6d6a91046c1fe4cb3dc3/runtime/linux/common_session_state.h#L51)，我们看到global_lock的定义：
 
```C
#define global_lock(name)	(&global(name ## _lock)) 
```

然后我们能看到如下的一些代码：

```C
    if (c->locked == 0) {
      if (!stp_lock_probe(locks, ARRAY_SIZE(locks)))
        goto out;
      else
        c->locked = 1;
    } else if (unlikely(c->locked == 2)) {
      _stp_error("invalid lock state");
    }
```

此时可以大概看出，原来stap代码对于共享变量的加锁，是一次全部都加。

这里有个小技巧，对于stap生成的代码，其实他对于你stp源代码每一行可能出问题的地方都有对应的一些C代码索引。当你看到如下的代码，你大概就能猜到这段C代码是翻译的哪些stp代码：

```C
c->last_stmt = "identifier 'stop' at compose_output_actions.stp:18:9";
```

因此我猜测，stp对全局变量的访问，是一把加上这个probe里访问的所有变量的锁，然后根据其读写，来加对应的读锁或者写锁（注意前面的locks[]的定义里有一个.write_p的变量）

在github的[这里](https://github.com/groleo/systemtap/blob/bd23cd846d787b59183d6d6a91046c1fe4cb3dc3/runtime/linux/probe_lock.h#L43)，我们可以看到stap的加锁方式：

```C
static unsigned
stp_lock_probe(const struct stp_probe_lock *locks, unsigned num_locks)
{
	unsigned i, retries = 0;
	for (i = 0; i < num_locks; ++i) {
		if (locks[i].write_p)
			while (!write_trylock(locks[i].lock)) {
#if !defined(STAP_SUPPRESS_TIME_LIMITS_ENABLE)
				if (++retries > MAXTRYLOCK)
					goto skip;
#endif
				udelay (TRYLOCKDELAY);
			}
		else
			while (!read_trylock(locks[i].lock)) {
#if !defined(STAP_SUPPRESS_TIME_LIMITS_ENABLE)
				if (++retries > MAXTRYLOCK)
					goto skip;
#endif
				udelay (TRYLOCKDELAY);
			}
	}
	return 1;

skip:
	atomic_inc(skipped_count());
#ifdef STP_TIMING
	atomic_inc(locks[i].skipped);
#endif
	stp_unlock_probe(locks, i);
	return 0;
}
```

这里的信息量略大，我们可以看到：1）确实会区分读写加锁，2）当加锁失败，有两种处理，一是计数，当失败到一定程度，则跳过这个probe，如果没有，则盲等10us（在我看到的实现里，是要等10us），而是直接等待10us，然后继续加锁。

10us并不是一个很短的时间，尤其在数据面里，20~40us基本可以完成一次ovs upcall了，这说明了stap的精度其实也就是在微秒级，而且对于某些特别频繁的probe，几乎是无法直接采集的。当然你可以通过
```
stap --supress-time-limits
```
的方式关闭这个，但是这又会引入新问题，比如某些事件过于频繁，如果把这玩意关了，可能会导致CPU stuck。

这意味着即使在stap的世界里，也没有银弹，你需要了解你的业务代码，通过猜测而编写trace代码，而不是寄希望于某些通用机制，当调包侠。

## trace的开销

我用bpftrace过一个非常短的函数。在火焰图上可以看出，当加上trace的时候，能看到uprobe的函数开销几乎占到了整个函数调用的时间的80%。但是即使是这样一种方式，我还是发现了这个函数偶尔的jitter，当然最终结论和这个函数基本无关。我想说的是，trace肯定有开销，但是你发现jitter达到了ms级别时，哪怕你的函数再短，你也可以用trace去筛查他到底是不是jitter的来源。
