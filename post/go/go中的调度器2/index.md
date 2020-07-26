
（接上...)

## go中的协程

在Go中，goroutine正表达了协程的概念。

### 动态栈

每一个os thread都有一个固定大小(2MB)的栈。一方面2MB的栈对于小的goroutine是浪费，对于go来说一次性创建成百上千的goroutine是普遍的。另一方面固定大小的栈对于更复杂和更深的递归函数调用是不够的。

相反goroutine的栈是动态的。它会从一个很小的栈开始。通常是2KB，根据需要动态伸缩，最大为1GB。

## 调度器的宏观介绍

runtime包中调度器的任务是分发(ready-to-run) goroutine给工作线程。调度器，使用了一种m:n scheduling的技术。它会在n个os thread上多工m个goroutine。因为不需要切换线程context，重新调度一个goroutine比调度一个线程代价低很多。

go调度器中有三个重要概念：

- G: Goroutine
- M: Machine, worker thread,传统意义上的thread routine。
- P: Processor, 是一种人为抽象，表示用于执行go代码的资源。M必须和一个P关联后才能执行Go代码，如果M idle或者进行系统调用则不需要P。P中包含了可运行的G队列。

P的数量由GOMAXPROCS指定，调整GOMAXPROCS会造成STW。

有一个无锁的list表示idle状态的P，当M开始执行Go代码时,必须从list中pop 一个P，当M结束运行代码时，M会将P放回到idle list。

M的两种状态为停驻(parking)和启动(unparking)(或者在进行系统调用)。

### M的自旋(spinning)

M自旋是相对于线程阻塞而言的。m绑定的P本地队列、全局队列、netpoller都没有G可运行时，m进入自旋并尝试从其他地方偷取G，m.spinning和sched.nmspinning表示自旋状态。

自旋有两级：

1. 跟P关联的M,idle时自旋来寻找新的G。
2. 未跟P关联的M，自旋来等待可用的P。

当存在类型2的M时，类型1的M是不阻塞的。

M的数量最大为GOMAXPROCS。未绑定P的阻塞状态M会进入休眠。

当一个新G产生，或者M进入系统调用，或者M从idle转入busy，当最后一个自旋线程发现工作，会唤醒一个新的自旋线程，确保最少有一个自旋状态的M(除非所有P都处于busy)，这样出现可运行G时，可以立即运行。而且避免线程trashing



### 正常运行状态

![](https://raw.githubusercontent.com/againest1/figure_bed/master/blog/3.jpg)

灰色部分的G处于非运行状态，但是是runable的，被放置于runequeues队列中。

为了降低锁竞争，每个P都有独立的本地G队列。

当G被创建，或者一个已经存在的可运行G时, G会被推到当前P的可运行G 列表末尾。当P结束执行G时，它首先尝试从自己的G列表中拿出一个G。

### 系统调用时发生了什么？

![](https://raw.githubusercontent.com/againest1/figure_bed/master/blog/4.jpg)

当发生系统调用时，M0线程阻塞，这个时候，需要通过P把任务队列挂载到其他线程中。使得其他线程可以运行这个P。

调度器需要确保有足够的线程来运行所有的P。图中的M1可能是新创建的，也可能从idle 线程池取出。

而正在执行系统调用M0会持有在做系统调用的G0，因为它理论上仍在执行，只是被系统阻塞了。

当系统调用返回，M0线程必须尝试获取一个P来执行这个返回的G0。正常情况下会从其他线程偷取一个P，如果偷取失败。会把这个goroutine放入全局G队列。再把线程放回到idle线程池，并休眠。

简要描述一下除系统调用外的其他goroutine阻塞调度策略。

- **原子、互斥量或channel操作**，调度器将把当前阻塞的G切换出去，重新调度本地runqueues上的其他G。
- **网络请求和IO操作**，go提供了网络轮询器(NetPoller)来实现IO多路复用，这时M可以执行runqueues上其他的G。
- **goroutine执行sleep**, go程序后台有一个监控线程sysmon，它监控那些长时间运行的G任务，设置一个可以强占的标识，别的G就可以进行强占。只要这个goroutine进行函数调用，那么就会被强占，也会保护现场，重新放入P的本地队列里面等待下次执行。

当某个P的runequeue全部执行完时，会从全局运行队列偷取任务。所有的P也会周期性的检查全局的runqueues。以防止全局队列中的协程饿死。

侧面也可以说明go是多线程的，即使GOMAXPROCS是1.

### 偷取任务

![](https://raw.githubusercontent.com/againest1/figure_bed/master/blog/5.jpg)

当一个P执行完了所有调度给它的G，它会尝试从全局队列中获取G，如果全局队列为空，那么会尝试从另一个P的runqueues中偷取一半的任务。这样确保每个P都在运行任务。