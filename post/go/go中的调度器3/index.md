
上一节中列举了一些基本场景来表述概念，我们对场景扩展来看调度器工作细节。我们可以看到

- local runqueues和global runqueues是如何负载均衡。
- M如何寻找G
- M如何在不同的G之间进行切换
- 任务偷取
- M为什么需要自旋
- 调度器抢占

## 场景1: 创建了新的g

![](https://raw.githubusercontent.com/againest1/figure_bed/master/blog/cj1.png)

g1中创建了g2，为了局部性优先加入p1的本地队列末尾。

## 场景2: g完成之后的线程复用

![](https://raw.githubusercontent.com/againest1/figure_bed/master/blog/cj2.png)

当g1运行完成后(goexit)，m1上的切换为g0，g0负责调度时协程的切换(schedule)。从p1的本地队列获取g2，从g0切换到g2，并开始运行g2(execute)。

## 场景3: 本地队列到全局队列的负载均衡

![](https://raw.githubusercontent.com/againest1/figure_bed/master/blog/cj3.png)

假设p的本地队列只能存4个g，g2将要创建6个g，(g3,g4,g5,g6)已加入p1的本地队列，此时p1本地队列满了。

在创建g7时，发现p1队列满，需要执行负载均衡，将队列中的前一半以及新创建的g7转移到全局队列中。实现中并不一定是新的g，如果g是g2之后就执行的，会被保存在本地队列，利用某个老的g替换新g加入全局队列），这些g被转移到全局队列时，会被打乱顺序。所以g3,g4,g7被转移到全局队列。

当g2创建g8时，p1的本地队列未满，所以g8加入到本地队列。

## 场景4: 自旋线程创建

![](https://raw.githubusercontent.com/againest1/figure_bed/master/blog/cj4.png)

创建新g时，运行的g2会尝试唤醒其他idle的p和m来进行自旋。假定g2唤醒了m2, m2绑定了p2，并运行g0，此时p2本地队列没有g，m2为自旋状态。

## 场景5: 全局队列到本地队列的负载均衡

![](https://raw.githubusercontent.com/againest1/figure_bed/master/blog/cj5.png)
自旋线程m2尝试从全局队列(GQ)偷取一批g放入本地队列(findrunnable)。m2从全局队列偷取数量公示：

`n = min(len(GQ) / GOMAXPROCS + 1, len(GQ/2))`

公式保证了不会偷取太多g到本地队列，会给其他p留一些。

此时m2从全局队列中偷取g3，并从g0切换到g3,运行g3。

## 场景6: 不同本地队列之间负载均衡

![](https://raw.githubusercontent.com/againest1/figure_bed/master/blog/cj6.png)

假设g2一直在m1上运行，m2将全局队列中的所有g都偷取并执行完，此时m2进入自旋，全局队列为空，此时m2会从其他p中偷取一半的g，放入本地队列。

## 场景7: 保持线程自旋

![](https://raw.githubusercontent.com/againest1/figure_bed/master/blog/cj7.png)
当前m1,和m2分别运行g2和g8，m3和m4中没有g可运行，处于自旋状态。我们希望有新的g创建时立刻能有m运行它。自旋本质还在运行，会消耗cpu 周期。过多的自旋线程会造成浪费，所以最多有GOMAXPROCS个自旋线程，其余的线程会进行休眠(notesleep)

## 场景8: 阻塞的系统调用

![](https://raw.githubusercontent.com/againest1/figure_bed/master/blog/cj8.png)

g8创建了g9，g8进行了**阻塞的系统调用**，m2和p2立即解绑，p2会执行以下判断：如果p2本地队列有g、全局队列有g或有空闲的m，p2都会立马唤醒1个m和它绑定，否则p2则会加入到空闲P列表，等待m来获取可用的p。

## 场景9: 非阻塞调用

如果进行非阻塞调用(如cgo)，m2和p2会解绑，但m2会记住p，然后g8和m2进入系统调用状态。当g8和m2退出系统调用时，会尝试获取p2，如果无法获取，则获取空闲的p，如果依然没有，g8会被记为可运行状态，并加入到全局队列。

## 场景10: 请求式抢占

Go调度在go1.12实现了请求式抢占，那是因为go调度器的抢占和OS的线程抢占比起来很柔和，不暴力，不会说线程时间片到了，或者更高优先级的任务到了，执行抢占调度。**go的抢占调度柔和到只给goroutine发送1个抢占请求，至于goroutine何时停下来，那就管不到了**。抢占请求需要满足2个条件中的1个：1）G进行系统调用超过20us，2）G运行超过10ms。调度器在启动的时候会启动一个单独的线程sysmon，它负责所有的监控工作，其中1项就是抢占，发现满足抢占条件的G时，就发出抢占请求。