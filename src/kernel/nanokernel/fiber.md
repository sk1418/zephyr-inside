---
title: Zephys OS nano 内核篇：fiber 服务
date: 2016-09-93 22:53:33
categories: ["Zephyr OS"]
tags: [Zephyr]
---

本文讲解 Zephyr 中的 fiber 相关的 API：
- [_nano_fiber_ready()](#_nano_fiber_ready)
- [_fiber_start()](#_fiber_start)
- [_fiber_start()](#_fiber_start)
- [_nano_fiber_swap()](#_nano_fiber_swap)
- [fiber_abort()](#fiber_abort)
- [fiber_delayed_start()](#fiber_delayed_start)
- [fiber_delayed_start_cancel()](#fiber_delayed_start_cancel)

# _nano_fiber_ready
```
void _nano_fiber_ready(struct tcs *tcs)
{	
	// 获取就绪线程链表的表头数据
	struct tcs *pQ = (struct tcs *)&_nanokernel.fiber;

	// 从表头开始逐渐向后查找链表中的线程，直到所查找线程的优先级比
	// 待插入的线程的优先级低(或者相同)为止
	while (pQ->link && (tcs->prio >= pQ->link->prio)) {
		pQ = pQ->link;
	}

	// 在查找到的线程前插入待插入的线程
	tcs->link = pQ->link;
	pQ->link = tcs;
}
```
内核大总管 _nanokernel　维护了一张由就绪线程形成的就绪链表，这张链表中的线程按优先级由高到低的顺序排列(数字越低，优先级越高)。_nano_fiber_ready() 的作用就是将线程 tcs 按照其优先级插入到这张链表中的对应位置。
> 严格地说，队列指的是只能在队尾加入数据，队首取出数据的一种数据结构。但是为了与后面的等待队列进行类比，我们将这个链表叫做就绪队列。

# _fiber_start
```
ano_thread_id_t _fiber_start(char *pStack,
		unsigned stackSize, /* stack size in bytes */
		nano_fiber_entry_t pEntry,
		int parameter1,
		int parameter2,
		unsigned priority,
		unsigned options)
{
	struct tcs *tcs;
	unsigned int imask;

	// 由于_new_thread()将在线程栈的低地址处存储线程的控制结构，
	// 所以此处直接将 tcs 指向线程栈的低地址处。
	tcs = (struct tcs *) pStack;
	// 新建一个线程
	_new_thread(pStack,
			stackSize,
			NULL,
			(_thread_entry_t)pEntry,
			(void *)parameter1,
			(void *)parameter2,
			(void *)0,
			priority,
			options);

	imask = irq_lock();

	// 将该线程加入到就绪线程链表中
	_nano_fiber_ready(tcs);

	if ((_nanokernel.current->flags & TASK) == TASK) {
		// 如果当前正在运行的上下文是 task，则进行上下文切换，然后由
		// 调度器选择执行优先级最高的线程
		_Swap(imask);
	} else {
		// 如果当前正在运行的上下文是 fiber，不进行上下文切换，
		// 因为 fiber 服务一旦运行，就必须运行完，它是不能被抢占的
		irq_unlock(imask);
	}

	return tcs;
}
```
irq_lock() 和 irq_unlock() 用于屏蔽中断和使能中断，具体信息请参考《Zephyr OS nano 内核篇：中断的使能和屏蔽》。

_Swap()用于进行上下文切换。当调用_Swap()函数后，调度器会保持当前正在执行的上下文的现场信息，然后调用就绪中优先级最高的线程去执行。具体信息请参考《Zephyr OS nano 内核篇：上下文切换》。

内核还为该函数 _fiber_start() 提供了两个别名：
```
FUNC_ALIAS(_fiber_start, fiber_fiber_start, nano_thread_id_t);
FUNC_ALIAS(_fiber_start, task_fiber_start, nano_thread_id_t);
FUNC_ALIAS(_fiber_start, fiber_start, nano_thread_id_t);
```
FUNC_ALIAS  是系统预定义的一个宏，用来为函数取别名，第一个参数是函数本来的名字，第二个参数是函数的别名，第三个参数是函数的返回值。

这样做的原因是为了兼容性考虑，以防今后由于某些原语使 fiber_fiber_start() 和 task_fiber_start() 成为两个实现方法不同的函数。所以我们在调用这类使用了别名的函数时，应尽量根据使用环境选择对应的别名函数。

# fiber_yield
```
void fiber_yield(void)
{
	unsigned int imask = irq_lock();

	if ((_nanokernel.fiber != (struct tcs *)NULL) &&
	    (_nanokernel.current->prio >= _nanokernel.fiber->prio)) {

		// 将本线程加入到就绪线程链表中，此时线程的状态由执行态转变为就绪态
		_nano_fiber_ready(_nanokernel.current);
		_Swap(imask);
	} else {
		irq_unlock(imask);
	}
}
```
如果当前正在执行的线程由于某种原因想主动使用 CPU 的使用权，可以调用该函数。但是 CPU 的使用权释放成功需要满足两个条件：
- 内核中存在就绪线程
- 该线程的优先级比内核中优先级最高的就绪线程的优先级低(或相等)

# _nano_fiber_swap
```

```
