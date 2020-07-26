
goroutine生命周期中包含了各种状态的变换，弄清楚了这些状态及状态切换的原来，对搞清楚go调度器会非常有帮助。

![](https://raw.githubusercontent.com/againest1/figure_bed/master/blog/15.jpg)
Gidle在go调度器没有真正被用到所以忽略。runtime创建goroutine，就会将其状态设置为Grunnable并加入到任务队列中。所以从Grunnable开始介绍

## 1. Grunnable

goroutine在下列几种情况会设置为Grunnable状态：

### 1.1 创建goroutine

在go中，包括用户入口main.mian在内的所有goroutine都是通过runtime.newproc->runtime.newproc1创建的，前者是对后者的一层封装。go关键字最终会被编译器映射为对runtime.newproc的调用。当runtime.newproc1完整资源分配及初始化后，新任务的状态会被置为Grunnable，然后被添加到当前P的本地任务队列中。

```go
func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) {
  // --snip--
  // 获取当前g所在的p，从p中创建一个新g(newg)
  _p_ := _g_.m.p.ptr()
  newg := gfget(_p_)
  // --snip--
  // 设置Goroutine状态为Grunnable
  casgstatus(newg, _Gdead, _Grunnable)
  // --snip--
  // // 新创建的g添加到run队列中
  runqput(_p_, newg, true)
  // --snip--
}
```

### 1.2 阻塞任务唤醒

当某个阻塞任务(Gwaiting)的等待条件满足而被唤醒时。（如g1项channel写入数据将唤醒等待接收的)，g1通过调用runtime.ready将g2状态重新置为Grunnable并添加到任务队列中。关于groutine阻塞，还有更详细的介绍。

```go
func ready(gp *g, traceskip int, next bool) {
  // --snip--
  // 获取current g
  _g_ := getg()
  // 将状态从Gwaiting转换至Grunnable
  casgstatus(gp, _Gwaiting, _Grunnable)
  // 添加到运行队列中
  runqput(_g_.m.p.ptr(), gp, next)
  // --snip--
}
```

### 1.3 其他

另外的路径是从Grunning和Gsyscall状态转换到Grunnable，后面再介绍。总之处于Grunnable的任务一定是在某个任务队列中，随时等待被调度执行。

## Grunning

所有状态为Grunnable的任务都可能通过findrunnable函数被调度器(P&M)获取，进而通过execute将其状态切换到Grunning，最后调用runtime.gogo加载context并执行。

```go
// One round of scheduler: find a runnable goroutine and execute it.
// Never returns.
func schedule() {
  // --snip--
  // 挑一个可运行的g，并执行
  if gp == nil {
    gp, inheritTime = findrunnable() // blocks until work is available
  }
  // --snip--
  execute(gp, inheritTime)
}
```

```go
// Schedules gp to run on the current M.
func execute(gp *g, inheritTime bool) {
  // 将当前g的M切换到新的g上
  _g_ := getg()
  _g_.m.curg = gp
  gp.m = _g_.m
  // 将Grunnable状态变更为Grunning
  casgstatus(gp, _Grunnable, _Grunning)
  // --snip--
  // 真正执行goroutine
  gogo(&gp.sched)
}
```

go采取的是一种协作式调度方案，一个正在运行的任务，需要通过yield的方式显式的让出处理器。

在Go1.2之后，runtime也支持一定程度的任务抢占--当系统线程sysmon发现某个任务执行时间过长或者runtime判断需要进行垃圾收集时，会将任务置为“可被抢占”的，当该任务下一次函数调用时，就会让出处理器并重新切换到Grunnable状态。

## Gsyscall

Go运行时为了保证高的并发性能，当会在任务执行OS系统调用前，先调用runtime.entersyscall函数将自己的状态置为Gsyscall(如果系统调用是阻塞式的或者执行过久，则将当前M与P分离)，当系统调用返回后，执行线程调用runtime.exitsyscall尝试重新获取P，如果成功且当前任务没有被抢占，则将状态切换回Grunning并继续执行；否则将状态置为Grunnable，等待再次被调度执行。

```go
func reentersyscall(pc, sp uintptr) {
	_g_ := getg()
	// --snip--
	casgstatus(_g_, _Grunning, _Gsyscall)
	// --snip--
  // 将m和p分离
	pp := _g_.m.p.ptr()
	pp.m = 0
	_g_.m.oldp.set(pp)
	_g_.m.p = 0
	// --snip--
}
```

```go
func exitsyscall() {
  _g_ := getg()
  // --snip--
  // 如果P还存在则重新获取P
  if exitsyscallfast(oldp) {
    // --snip--
    casgstatus(_g_, _Gsyscall, _Grunning)
    // --snip--
    return
  }
  // --snip--
  // 如果P不存在则Gsyscall->Grunnable
  mcall(exitsyscall0)
  // --snip--
}
```



## Gwaiting

当一个任务需要的资源或运行条件不能被满足时，需要调用runtime.park函数进入该状态，之后除非等待条件满足，否则任务将一直处于等待状态不能执行。除了之前举过的channel的例子外，Go的定时器，网络io操作，原子，信号量都可能引起任务的阻塞。

```go
// park continuation on g0.
func park_m(gp *g) {
  // --snip--
  casgstatus(gp, _Grunning, _Gwaiting)
	// --snip--
}
```

```go
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
	// --snip--
  mcall(park_m)
}
```

runtime.park中lock是goroutine阻塞时需要释放的锁，(比如channel)，reason是阻塞的原因，方便gdb调试。当所有任务处于Gwaiting状态时，也就表示当前程序进入了死锁状态，那么runtime回检测到这种情况，并输出所有Gwaiting任务的backtrace信息。

## Gdead

当一个任务执行结束后，会调用runtime.goexit结束。将状态置为Gdead，并进入当前P的gFree列表。