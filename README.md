[合集 \- Go底层基础学习(2\)](https://github.com)[1\.Go语言net/http包源码学习10\-22](https://github.com/MelonTe/p/18493338)2\.Golang的GMP调度模型与源码解析11\-17收起
# 0、引言


我们知道，这当代操作系统中，多线程和多进程模型被广泛的使用以提高系统的并发效率。随着互联网不断的发展，面对如今的高并发场景，为每个任务都创建一个线程是不现实的，使用线程则需要系统不断的在用户态和内核态之间不断的切换，引起不必要的损耗，于是引入了协程。协程存在于用户空间，是一种轻量级的并发执行单元，其创建和上下文的开销更小，如何管理数量众多的协程是一个重要的话题。此篇笔记用于分享笔者学习Go语言协程调度的GMP模型的理解，以及源码的实现。当前使用的Go语言版本为1\.22\.4。


本篇笔记参考了以下文章：


\[[Golang三关\-典藏版] Golang 调度器 GMP 原理与调度全分析 \| Go 技术论坛](https://github.com):[FlowerCloud机场](https://yunbeijia.com)


[Golang GMP 原理](https://github.com)


[Golang\-gopark函数和goready函数原理分析](https://github.com)


# 1、GMP模型拆解


Goroutine调度器的工作是将准备运行的goroutine分配到工作线程上，涉及到的主要概念如下：


## 1\.1、G


G代表的是Goroutine，是Go语言对协程概念的抽象，其有以下的特点：


* 是一个轻量级的线程
* 拥有自己的栈、状态、以及执行的任务函数
* 每一个G会被分配到一个可用的P，并且在M上运行


其结构定义位于runtime/runtime2\.go中：



```
type g struct {
    // ...
    m         *m      
    // ...
    sched     gobuf
    // ...
}

type gobuf struct {
    sp   uintptr
    pc   uintptr
    ret  uintptr
    bp   uintptr // for framepointer-enabled architectures
}

```

在这里，我们核心关注其内嵌了一个m和一个`gobuf`类型的sched。`gobuf`主要用于Gorutine的上下文切换，其保存了G执行过程中的CPU寄存器的状态，使得G在暂停、调度和恢复运行时能够正确地恢复上下文。


G主要有以下几种状态：



```
const (
	_Gidle = iota // 0
	_Grunnable // 1
	_Grunning // 2
	_Gsyscall // 3
	_Gwaiting // 4
    //...
	_Gdead // 6
    //...
	_Gcopystack // 8
    _Gpreempted // 9
	//...
)

```

* `Gidle`：表示这个G刚刚被分配，尚未初始化。
* `Grunnable`：表示这个G在运行队列中，它当前不再执行用户代码，栈未被占用。
* `Grunning`：表示这个G可能在执行用户代码，栈被这个G占用，它不在运行队列中，并且它被分配给了一个M和一个P（g.m和g.m.p是有效的）。
* `Gsyscall`：表示这个G正在执行系统调用，它不在执行用户代码，栈被这个G占用。它不在运行队列中，并且它被分配给了一个M。
* `Gwaiting`：表示这G被堵塞在运行时，它没有执行用户代码，也不在运行队列中，但是它应该被记录在某个地方，以便在必要时将其唤醒。（ready()）gc、channel 通信或者锁操作时经常会进入这种状态。
* `Gdead`：表示这个G当前未使用，它可能是刚被初始化，也可能是已经被销毁。
* `Gcopystack`：表示这个G的栈正在被移动。
* `Gpreempted`：表示这个G因抢占而被挂起，且该G自行停止，等待进一步的恢复。它类似于`Gwaiting`，但是`Gpreempted`还没有一个负责将其状态恢复的管理者，只有某个`suspendG`操作将该G的状态从`Gpreempted`转换为`Gwaiting`，这样调度器才会接管这个G。


在阅读有关调度逻辑的源码的时候，我们可以通过搜索`casgstatus`方法去定位到使得G状态改变的函数，例如：`casgstatus(gp, _Grunning, _Gsyscall)`表示将该G的状态从Grunning变换到Gsyscall，就可以找到对应的函数学习了。


![](https://img2024.cnblogs.com/blog/3542244/202411/3542244-20241117153205143-1345385046.png)


## 1\.2、M


M是Machine，也是Worker Thread，代表的是操作系统的线程。Go运行时在需要时创建或者销毁M，将G安排到M上执行，充分利用多核CPU的能力。其具有以下的特点：


* M是Go与操作系统之间的桥梁，它负责执行分配给它的G。
* M的数量会根据系统资源进行调整。
* M可能会被特定的G通过`LockOSThread`锁定，这种G和M的绑定确保了特定Goroutine可以持续使用同一个线程。


结构定义如下：



```
type m struct{
	g0      *g     // goroutine with scheduling stack
	curg          *g       // current running goroutine
	tls           [tlsSlots]uintptr // thread-local storage (for x86 extern register)
	p             puintptr // attached p for executing go code (nil if not executing go code)
	oldp          puintptr // the p that was attached before executing a syscall
	//...
}

```

每一个M结构体都会有一个名为`g0`的G，它是一个特殊的Goroutine，它并不复杂执行用户的代码，而是负责调度G。g0会分配G绑定到M中执行。`tls`表示的是“Local Thread Storage”，其存储了与当前线程相关的特定信息，而`tls`数组的第一个槽位通常用于存储`g0`的栈指针。


M存在一个状态，名为**“自旋态”**，处在自旋态的M会不断的往全局队列中寻找可运行的G去执行，并且解除自旋态。


## 1\.3、P


P是Processor，代表逻辑处理器，是Goroutine调度的虚拟概念。每个P负责分配执行Goroutine的资源，其具有以下的特点：


* P是G的执行上下文，它具有一个本地队列存储着G，以及对应的任务调度机制，负责在M上执行一个具体的G。
* P的数量由环境变量`GOMAXPROCS`决定，如果其数量大于CPU的物理线程数量时就没有更多的意义了。
* **P是去执行Go代码所必备的资源，M必须绑定了一个P才能去执行Go代码。但是M可以在没有绑定P的情况下执行系统调用或者被阻塞。**



```
type p struct {
	status      uint32
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	m           muintptr
	runnext guintptr
	//...
}

```

* runq存储了这个P具有的goroutine队列，最大长度为256
* runqhead和runqtail分别指向队列的头部和尾部
* runnext存储了下一个可执行的goroutine


P也含有几个状态，如下：



```
const (
	_Pidle = iota
	_Prunning
	_Psyscall
	_Pgcstop
	_Pdead
)

```

* Pidle：表示P没有被运行用户代码或者调度器，通常这个P在空闲P列表中，供调度器使用，但它也可能在其他状态之间转换。P由空闲队列`idle list`或者其他转换其状态的对象拥有，它的`runq`是空的。
* Prunning：表示P被M拥有，并且正在运行用户代码或者调度器。只有拥有此P的M被允许更改P的状态，M可以将P转换为Pidle（当没有工作的时候）、Psyscall（当进入一个系统调用时）、Pgcstop（安顿垃圾回收时）。M还可以将P的所有权交接给另一个M（例如调度一个locked的G）
* Psyscall：表示P没有在运行用户代码，与在系统调用中的M相关但不被其拥有。处于Psyscall状态的P可能会被其他M抢走。将P转换给另一个M是轻量级的，并且P会保持和原始的M的关联性。
* Pgcstop：表示P被暂停以进行STW(Stop The World)（执行垃圾回收）。
* Pdead：表示P不再被使用（GOMAXPROCS减少）。死去的P将会被剥夺资源，但是任然会保留少量的资源例如Trace Buffer，用于后续的跟踪分析需求。


## 1\.4、Schedt


`schedt`是全局goroutine队列的封装



```
type schedt struct {
    // ...
    lock mutex
    // ...
    runq     gQueue
    runqsize int32![](https://img2024.cnblogs.com/blog/3542244/202411/3542244-20241117153220788-1594654379.png)

    // ...
}

```

* lock：是操作全局队列的锁
* runq：存储G的队列
* runqsize：全局G队列的容量


# 2、调度模型的工作流程


我们可以用下图来整体的表示该调度模型的流程：


在接下来的部分，我们将主要探讨GMP调度模型是怎么完成一轮调度的，即是如何完成g0到g再到g0的切换的，期间大致发生了什么。


## 2\.1、G的状态转换


我们刚刚提及到，每一个M都有一个名为`g0`的Goroutine，去负责调度普通的g绑定到M上执行。g0和普通的g之间存在一个转换，当执行普通的g上的代码的时候，就会将执行权交给g，当g执行完代码或者因为原因需要被挂起、退出执行等，就会重新将执行权交给g0。


g0和P是一个协作的关系，P的队列决定了哪些goroutine可以在绑定P时被调用，而g0是执行调度逻辑的关键的goroutine，负责在必要时释放P的资源。


当g0需要将执行权交给g时，会调用一个名为`gogo`的方法，传入g的栈指针，去执行用户的代码。



```
func gogo(buf *gobuf)

```

当需要重新将执行权转交给g0时，都会执行一个名为`mcall`的方法。



```
func mcall(fn func(*g))

```

mcall在go需要进行协程调换时被调用，它传入一个回调函数`fn`，里面携带了当前正在运行的g的指针，它主要做了以下三点的工作：


* 保存当前g的信息，即将PC/SP的信息存储到g\-\>sched中，保证后续可以恢复g的执行现场。
* 将当前M的堆栈从g切换到g0
* 在g0的栈上执行新的函数fn，通常在fn中会进一步安排g的去向，并且调用`schedule`函数，让当前M去寻找另一个可以执行的G。


![](https://img2024.cnblogs.com/blog/3542244/202411/3542244-20241117153231795-1988772537.png)


## 2\.2、调度类型


我们现在知道了，g和g0是通过什么函数进行状态切换的。接下来我们就要来探讨，它们是什么情况下要进行切换，即调度策略有什么。


GMP调度模型一共有4种调度策略，分别为：**主动调度**、**被动调度**、**正常调度**、**抢占调度**。


![](https://img2024.cnblogs.com/blog/3542244/202411/3542244-20241117153246641-1662481636.png)


* 主动调度：提供给用户的方法，当用户调用了runtime.Gosched()方法时，此时当前的g会让出执行权，将g安排进任务队列等待下一次被调度。
* 被动调度：当因不满足某种执行条件，通常为channel读写条件不满足时，会执行gopark()函数，此时的g将会被置为等待状态。
* 正常调度：g正常的执行完毕，转接执行权。
* 抢占调度：存在一个全局监控者moniter，它会每隔一段时间周期去检查是否有G运行太长时间，若发现了，将会通知P去进行和M的解绑，让出P。这里需要全局监控者的存在是因为当G进入到系统调用的时候，这个线程M会陷入僵持，无法主动去检查，需要外援辅助。


## 2\.3、宏观调度流程


接下来我们来关注整体一轮的调度流程，对于g0和g的一轮调度，可以用下图来表示。


![](https://img2024.cnblogs.com/blog/3542244/202411/3542244-20241117153256335-166429556.png)


`schedule`作为每一轮调度的开始，它会寻找到可以执行的G，然后调用`execute`将该g绑定到一个线程M上，然后执行`gogo`方法去真正的运行一个goroutine。当需要转换时，goroutine会在底层执行`mcall`方法，保存栈信息，然后执行回调函数`fn`，即绿框内的方法之一，将执行权重新交给g0。


### 2\.3\.1、schedule()


`schedule()`方法定位于`runtime/proc`中，忽略非主流程部分，源码内容如下：



```
//找到一个是就绪态的G去运行
func schedule() {
	mp := getg().m

	//...

top:
	pp := mp.p.ptr()
	pp.preempt = false

	//如果该M在自旋，但是队列含有G，那么抛出异常。
	if mp.spinning && (pp.runnext != 0 || pp.runqhead != pp.runqtail) {
		throw("schedule: spinning with local work")
	}

	gp, inheritTime, tryWakeP := findRunnable() //阻塞的寻找G

	
    //...

	//当前M将要运转一个G，解除自旋状态
	if mp.spinning {
		resetspinning()
	}

	//...

	execute(gp, inheritTime)
}

```

该方法主要是寻找一个可以运行的G，交给该线程去运行。我们在一开始提到，线程会存在一种名为**“自旋态”**的状态，它会不断的自旋去寻找可以执行的G来执行，成功找到了就解除了自旋态。


这里存在一个点我们值得去注意，处在自旋态的线程它不是在空占用计算资源吗？那么不就是降低了系统的性能吗？


其实这是一个中和的策略，假如每次当出现了一个新的Goroutine需要去执行的时候，我们才创建一个线程M去执行它，然后执行完了又删除掉不去复用，那么就会带来大量的创建销毁的资源消耗。我们希望当有一个新的Goroutine来的时候，能立即有一个M去执行它，就可以将空闲暂时无任务处理的M去自己寻找Goroutine，减少了创建销毁的资源消耗。但是我们也不能有太多的处于自旋态的线程，不然就造就另一个过多消耗的地方了。


我们先跟进一下`resetspinning()`，看看其执行的策略是什么。


#### 1、resetspinning()



```
func resetspinning() {
	gp := getg()
	//...
	gp.m.spinning = false
	nmspinning := sched.nmspinning.Add(-1)
	//...
	wakep()
}



//尝试添加一个P去执行G。该方法被调用当一个G状态为runnable时。
func wakep() {
    //如果自旋的M数量不为0则返回
	if sched.nmspinning.Load() != 0 || !sched.nmspinning.CompareAndSwap(0, 1) {
		return
	}

	// 禁用抢占，直到 pp 的所有权转移到 startm 中的下一个 M，否则在这里的抢占将导致 pp 被卡在等待进入 _Pgcstop 状态。
	mp := acquirem()

	var pp *p
	lock(&sched.lock)
    //尝试从空闲P队列获取一个P
	pp, _ = pidlegetSpinning(0)
	if pp == nil {
		if sched.nmspinning.Add(-1) < 0 {
			throw("wakep: negative nmspinning")
		}
		unlock(&sched.lock)
		releasem(mp)
		return
	}
	
	unlock(&sched.lock)

	startm(pp, true, false)

	releasem(mp)
}

```

在`resetspinning`中，我们先将当前M解除了自旋态，然后尝试去唤醒一个P，即进入到`wakep()`方法中。



```
if sched.nmspinning.Load() != 0 || !sched.nmspinning.CompareAndSwap(0, 1) {
		return
	}

```

在wakep方法内，我们先检查了当前处在自旋的M的数量，假如\>0，则不再去唤醒一个新的P，这是为了防止同一时间内过多的自旋的M空运转消耗CPU资源。



```
pp, _ = pidlegetSpinning(0)
	if pp == nil {
		if sched.nmspinning.Add(-1) < 0 {
			throw("wakep: negative nmspinning")
		}
		unlock(&sched.lock)
		releasem(mp)
		return
	}

```

接着会尝试从空闲P队列中获取一个P，如果没有空闲的P，那么此时会减少自旋线程的数量（这里只是减少了数量，但是具体这个处在自旋的线程接下来去做什么了我也没有明白）并且返回。



```
startm(pp, true, false)

```

假如获取了一个空闲的P，会为这一个P分配一个线程M。


![](https://img2024.cnblogs.com/blog/3542244/202411/3542244-20241117153312376-180377462.png)


#### 2、findRunnable()


findRunnable是一轮调度流程中最核心的方法，它用于找到一个可执行的G。



```
func findRunnable() (gp *g, inheritTime, tryWakeP bool) {
	mp := getg().m
top:
    pp := mp.p.ptr()
	//...
 	
    //每61次调度周期就检查一次全局G队列，防止在特定情况只依赖于本地队列。
	if pp.schedtick%61 == 0 && sched.runqsize > 0 {
		lock(&sched.lock)
		gp := globrunqget(pp, 1)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false, false
		}
	}
    //...
    // local runq
	if gp, inheritTime := runqget(pp); gp != nil {
		return gp, inheritTime, false
	}

	// global runq
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(pp, 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false, false
		}
	}
    
    //在正式的去偷取G之前，用非阻塞的方式检查是否有就绪的网络协程，这是对netpoll的一个优化。
	if netpollinited() && netpollAnyWaiters() && sched.lastpoll.Load() != 0 {
		if list, delta := netpoll(0); !list.empty() { // non-blocking
			gp := list.pop()
			injectglist(&list)
			netpollAdjustWaiters(delta)
			trace := traceAcquire()
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.ok() {
				trace.GoUnpark(gp, 0)
				traceRelease(trace)
			}
			return gp, false, false
		}
	}
    
    //如果当前的M出于自旋状态，或者说处于自旋状态的M的数量小于活跃的P数量的一半时，则进行G窃取。（防止当系统的并行度较低时，自旋的M过多占用CPU资源）
	if mp.spinning || 2*sched.nmspinning.Load() < gomaxprocs-sched.npidle.Load() {
		if !mp.spinning {
			mp.becomeSpinning()
		}

		gp, inheritTime, tnow, w, newWork := stealWork(now)
		if gp != nil {
			// Successfully stole.
			return gp, inheritTime, false
		}
		if newWork {
			// There may be new timer or GC work; restart to
			// discover.
			goto top
		}

		now = tnow
		if w != 0 && (pollUntil == 0 || w < pollUntil) {
			// Earlier timer to wait for.
			pollUntil = w
		}
	}
    
    //...

```

其主要的执行步骤如下：


##### （一）第六十一次调度



```
if pp.schedtick%61 == 0 && sched.runqsize > 0 {
		lock(&sched.lock)
		gp := globrunqget(pp, 1)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false, false
		}
	}

```

首先检查P的调度次数，假如这次是P的第61此次调度，并且全局的G队列长度\>0，就会从全局队列获取一个G。这是为了防止在特定情况下，只运行本地队列的G，忽视了全局队列。


其内部调用的`globrunqget`方法主流程如下：



```
//尝试从G的全局队列获取一批G
func globrunqget(pp *p, max int32) *g {
	assertLockHeld(&sched.lock)
	//检查全局队列是否为空
	if sched.runqsize == 0 {
		return nil
	}

    //计算需要获取的G的数量
	n := sched.runqsize/gomaxprocs + 1
	if n > sched.runqsize {
		n = sched.runqsize
	}
	if max > 0 && n > max {
		n = max
	}
    //确保从队列中获取的G数量不超过当前本地队列的G数量的一半，避免全局队列所有的G都转移到本地队列中导致负载不均衡
	if n > int32(len(pp.runq))/2 {
		n = int32(len(pp.runq)) / 2
	}
	sched.runqsize -= n

	gp := sched.runq.pop()
	n--
	for ; n > 0; n-- {
		gp1 := sched.runq.pop()
		runqput(pp, gp1, false)
	}
	return gp
}

```


```
//计算需要获取的G的数量
	n := sched.runqsize/gomaxprocs + 1
	if n > sched.runqsize {
		n = sched.runqsize
	}
	if max > 0 && n > max {
		n = max
	}
	if n > int32(len(pp.runq))/2 {
		n = int32(len(pp.runq)) / 2
	}

```

n为要从全局G队列获取的G的数量，可以看到它会至少获取一个G，至多获取`runqsize/gomaxprocs+1`个G，它保证了一个P不过多的获取G从而影响负载均衡。并且不允许n一次获取全局G队列一半以上的G，保证负载均衡。



```
gp := sched.runq.pop()
	n--
	for ; n > 0; n-- {
		gp1 := sched.runq.pop()
		runqput(pp, gp1, false)
	}

```

决定好获取多少个G后，第一个G会直接通过指针返回，剩余的则是将其添加到P的本地队列中。


在当前（一）的调用中，函数设置了max值为1，因此只会从全局队列获取1个G返回。




---


虽然在（一）中不会执行`runqput`，但是我们还是来看看是怎么将G添加到P的本地队列的。



```
// runqput尝试将G放到本地队列中
//如果next是False，runqput会将G添加到本地队列的尾部
//如果是True，runqput会将G添加到下一个将被调度的G的槽位
//如果运行队列满了，那么将会把g放回全局队列
func runqput(pp *p, gp *g, next bool) {
    //
	if randomizeScheduler && next && randn(2) == 0 {
		next = false
	}

	if next {
	retryNext:
		oldnext := pp.runnext
		if !pp.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
			goto retryNext
		}
		if oldnext == 0 {
			return
		}
		// Kick the old runnext out to the regular run queue.
		gp = oldnext.ptr()
	}

retry:
	h := atomic.LoadAcq(&pp.runqhead) //加载队列头的位置
	t := pp.runqtail
	if t-h < uint32(len(pp.runq)) { //检查本地队列是否已满
		pp.runq[t%uint32(len(pp.runq))].set(gp) //未满将gp插入runqtail的指定位置
		atomic.StoreRel(&pp.runqtail, t+1) //更新runtail，表示插入的G可供消费
		return
	}
	if runqputslow(pp, gp, h, t) { //如果本地队列已满，则尝试放回全局队列
		return
	}
	// the queue is not full, now the put above must succeed
	goto retry
}

```


```
if randomizeScheduler && next && randn(2) == 0 {
		next = false
	}

```

在第一步中，我们看到即使`next`被设置为true，即要求了该G应该被放置在本地P队列的`runnext`槽位中，**也会有概率地将next置为false**。



```
if next {
	retryNext:
		oldnext := pp.runnext
		if !pp.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
			goto retryNext
		}
		if oldnext == 0 {
			return
		}
		// Kick the old runnext out to the regular run queue.
		gp = oldnext.ptr()
	}

```

假如next仍为true，此时先获取原本P调度器中，runnext槽位的G（oldnext），然后会不断地尝试将新的G替换掉旧的G直到成功为止。当成功之后，在下面的操作流程中会把旧的G放入到P的本地队列中。



```
retry:
	h := atomic.LoadAcq(&pp.runqhead) //加载队列头的位置
	t := pp.runqtail
	if t-h < uint32(len(pp.runq)) { //检查本地队列是否已满
		pp.runq[t%uint32(len(pp.runq))].set(gp) //未满将gp插入runqtail的指定位置
		atomic.StoreRel(&pp.runqtail, t+1) //更新runtail，表示插入的G可供消费
		return
	}
	if runqputslow(pp, gp, h, t) { //如果本地队列已满，则尝试放回全局队列
		return
	}
	// the queue is not full, now the put above must succeed
	goto retry
}

```

在将G加入进P的本地队列的流程中，需要获取队列头部和尾部的坐标，用来判断本地队列是否已满，未满则将G插入进本地队列的尾部中。否则执行`runqputslow`方法，尝试放回全局队列。




---


接下来继续跟进`runqputslow`方法的执行流程。



```
//将G和一批工作（本地队列的G）放入到全局队列
func runqputslow(pp *p, gp *g, h, t uint32) bool {
	var batch [len(pp.runq)/2 + 1]*g //本地队列一半的G

	// First, grab a batch from local queue.
	n := t - h
	n = n / 2
	if n != uint32(len(pp.runq)/2) {
		throw("runqputslow: queue is not full")
	}
	for i := uint32(0); i < n; i++ {
		batch[i] = pp.runq[(h+i)%uint32(len(pp.runq))].ptr()
	}
	if !atomic.CasRel(&pp.runqhead, h, h+n) { // cas-release, commits consume
		return false
	}
	batch[n] = gp

	if randomizeScheduler { //打乱顺序
		for i := uint32(1); i <= n; i++ {
			j := cheaprandn(i + 1)
			batch[i], batch[j] = batch[j], batch[i]
		}
	}

	// Link the goroutines.
	for i := uint32(0); i < n; i++ {
		batch[i].schedlink.set(batch[i+1])
	}
	var q gQueue
	q.head.set(batch[0])
	q.tail.set(batch[n])

	// Now put the batch on global queue.
	lock(&sched.lock)
	globrunqputbatch(&q, int32(n+1))
	unlock(&sched.lock)
	return true
}

```

其执行流程如下：



```
var batch [len(pp.runq)/2 + 1]*g //本地队列一半的G

```

首先创建一个batch数组，是容量为P的本地队列当前含有的G的数量的一半，用于存储将转移的G。



```
n := t - h
	n = n / 2
	if n != uint32(len(pp.runq)/2) {
		throw("runqputslow: queue is not full")
	}
	for i := uint32(0); i < n; i++ {
		batch[i] = pp.runq[(h+i)%uint32(len(pp.runq))].ptr()
	}

```

接着，开始将本地队列一半的G的指针，存储在batch中。



```
if randomizeScheduler { //打乱顺序
		for i := uint32(1); i <= n; i++ {
			j := cheaprandn(i + 1)
			batch[i], batch[j] = batch[j], batch[i]
		}
	}

```

然后会打乱batch中的顺序，保证随机性。



```
// Link the goroutines.
	for i := uint32(0); i < n; i++ {
		batch[i].schedlink.set(batch[i+1])
	}
	var q gQueue
	q.head.set(batch[0])
	q.tail.set(batch[n])

	// Now put the batch on global queue.
	lock(&sched.lock)
	globrunqputbatch(&q, int32(n+1))
	unlock(&sched.lock)
	return true

```

最后一部是将batch中的各个G用指针连接起来，转换为**链表**的形式，并且链接在全局队列中。


`runqput`连接的流程较长，用下图来概括：


![](https://img2024.cnblogs.com/blog/3542244/202411/3542244-20241117153345874-737860941.png)


##### （二）本地队列获取



```
// local runq
	if gp, inheritTime := runqget(pp); gp != nil {
		return gp, inheritTime, false
	}

```

假如不是第61次调用，`findrunnable`会尝试从本地队列中获取一个G用于调度。我们来看runqget方法的执行。



```
// 从本地可运行队列中获取 g。
func runqget(pp *p) (gp *g, inheritTime bool) {
	// 如果有 runnext，则它是下一个要运行的 G。
	next := pp.runnext
    // 如果 runnext 非零且 CAS 操作失败，它只能被另一个 P 窃取，因为其他 P 可以竞争将 runnext 设置为零，但只有当前 P 可以将其设置为非零。
	// 因此，如果 CAS 失败，则无需重试该操作。
	if next != 0 && pp.runnext.cas(next, 0) {
		return next.ptr(), true
	}

	for {
		h := atomic.LoadAcq(&pp.runqhead) // load-acquire, synchronize with other consumers
		t := pp.runqtail
		if t == h {
			return nil, false
		}
		gp := pp.runq[h%uint32(len(pp.runq))].ptr()
		if atomic.CasRel(&pp.runqhead, h, h+1) { // cas-release, commits consume
			return gp, false
		}
	}
}

```

假如可以获取到P的runnext，则返回这一个G，否则就获取本地队列的头部的G。


##### （三）全局队列获取



```
// global runq
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(pp, 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false, false
		}
	}

```

假如无法从本地队列获取到G，则说明了P的本地队列为空，此时会尝试从全局队列获取G。调用了`globrunqget`方法从全局队列获取G，注意此时因为设置了max为0表示不生效，该方法**可能会从全局队列中获取多个G放到P的本地队列内**。关于该方法的具体代码已经在（一）中讲解。


##### （四）网络事件获取



```
    //在正式的去偷取G之前，用非阻塞的方式检查是否有就绪的网络协程，这是对netpoll的一个优化。
	if netpollinited() && netpollAnyWaiters() && sched.lastpoll.Load() != 0 {
		if list, delta := netpoll(0); !list.empty() { // non-blocking
			gp := list.pop()
			injectglist(&list)
			netpollAdjustWaiters(delta)
			trace := traceAcquire()
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.ok() {
				trace.GoUnpark(gp, 0)
				traceRelease(trace)
			}
			return gp, false, false
		}
	}

```

假如本地队列和全局队列都没有G可以获取，此时我们将进入GMP调度模型的一个特殊机制：**WorkStealing**，即从其他的P调度器中偷取其本地队列的G到自己的本地队列中，这是GMP调度模型独有的机制，可以更加充分地利用线程提高系统整体效率。


在此之前，会先尝试用非阻塞的方式获取准备就绪的网络协程，如果有则先执行网络协程。



> 为什么在携程的调度中，还要专门引入对网络协程事件的检测？这一部分不应该解耦吗？



> 这是我自己的一个思考，我认为这应该是Go的运行时的设计原则的一个方面体现。runtime的主要任务是负责`协程调度`和`资源管理`，但是在实际应用中，网络事件的处理通常会和协程调度紧密关联。Go使用`非阻塞网络轮询机制`（netpoll）允许在有网络事件发生时能快速的唤醒和调度相应的协程去处理，在进行了一次本地队列和全局队列的检查后，进行一次网络协程的检查能保证对网络I/O的快速响应。


##### （五）工作窃取



```
	if mp.spinning || 2*sched.nmspinning.Load() < gomaxprocs-sched.npidle.Load() {
		if !mp.spinning {
			mp.becomeSpinning()
		}

		gp, inheritTime, tnow, w, newWork := stealWork(now)
		if gp != nil {
			// Successfully stole.
			return gp, inheritTime, false
		}
		//...
	}

```

当本地队列和全局队列都没有G时，此时会进行工作窃取机制，尝试从其他调度器P中窃取G。



```
if mp.spinning || 2*sched.nmspinning.Load() < gomaxprocs-sched.npidle.Load() {
		if !mp.spinning {
			mp.becomeSpinning()
		}

```

如果当前的**自旋的M的数量＜空闲的P的数量的一半**，就会将当前M设置为自旋态。



```
gp, inheritTime, tnow, w, newWork := stealWork(now)
		if gp != nil {
			// Successfully stole.
			return gp, inheritTime, false
		}

```

调用`stealWork`进行窃取。




---



```
func stealWork(now int64) (gp *g, inheritTime bool, rnow, pollUntil int64, newWork bool) {
	pp := getg().m.p.ptr()

	ranTimer := false

    //最多从其他P窃取4次任务
	const stealTries = 4
	for i := 0; i < stealTries; i++ {
        //在进行最后一次的遍历前，优先检查其他P的Timer队列
		stealTimersOrRunNextG := i == stealTries-1
		//随机生成遍历起点
		for enum := stealOrder.start(cheaprand()); !enum.done(); enum.next() {
			//...
			p2 := allp[enum.position()]
			if pp == p2 {
				continue
			}

			
			//...

			//如果P是非空闲的，则尝试窃取
			if !idlepMask.read(enum.position()) {
				if gp := runqsteal(pp, p2, stealTimersOrRunNextG); gp != nil {
					return gp, false, now, pollUntil, ranTimer
				}
			}
		}
	}

	//如果在所有尝试中均未找到可运行的 Goroutine 或 Timer，则返回 nil，并返回 pollUntil（下一次轮询的时间）。
	return nil, false, now, pollUntil, ranTimer
}

```


```
const stealTries = 4
	for i := 0; i < stealTries; i++ {

```

当前P会尝试从其他的P的本地队列中进行窃取，最多会进行4次。



```
for enum := stealOrder.start(cheaprand()); !enum.done(); enum.next() {
			//...
			p2 := allp[enum.position()]
			if pp == p2 {
				continue
			}

			
			//...

			//如果P是非空闲的，则尝试窃取
			if !idlepMask.read(enum.position()) {
				if gp := runqsteal(pp, p2, stealTimersOrRunNextG); gp != nil {
					return gp, false, now, pollUntil, ranTimer
				}
			}
		}

```

使用`runqsteal`方法进行窃取。




---



```
//从p2偷去一半的工作到p中
func runqsteal(pp, p2 *p, stealRunNextG bool) *g {
	t := pp.runqtail
	n := runqgrab(p2, &pp.runq, t, stealRunNextG)
	if n == 0 {
		return nil
	}
	n--
	gp := pp.runq[(t+n)%uint32(len(pp.runq))].ptr()
	if n == 0 {
		return gp
	}
	h := atomic.LoadAcq(&pp.runqhead) // load-acquire, synchronize with consumers
	if t-h+n >= uint32(len(pp.runq)) {
		throw("runqsteal: runq overflow")
	}
	atomic.StoreRel(&pp.runqtail, t+n) // store-release, makes the item available for consumption
	return gp
}

```

`runqsteal`方法会将p2的本地队列中偷取其一半的G放到p的本地队列中，我们进而跟进`runqgrab`方法；




---



```
func runqgrab(pp *p, batch *[256]guintptr, batchHead uint32, stealRunNextG bool) uint32 {
	for {
		h := atomic.LoadAcq(&pp.runqhead) // load-acquire, synchronize with other consumers
		t := atomic.LoadAcq(&pp.runqtail) // load-acquire, synchronize with the producer
		n := t - h
		n = n - n/2
		if n == 0 {
			if stealRunNextG {
				//尝试偷取P的下一个将要调度的G
				if next := pp.runnext; next != 0 {
                    //如果P正在运行，为了避免产生频繁的任务状态“抖动”，互相抢占任务导致的调度竞争，所以休眠一会，等待P调度完成再尝试获取。
					if pp.status == _Prunning {
						if !osHasLowResTimer {
							usleep(3)
						} else {
							osyield()
						}
					}
                    //尝试窃取任务
					if !pp.runnext.cas(next, 0) {
						continue
					}
                    //窃取成功
					batch[batchHead%uint32(len(batch))] = next
					return 1
				}
			}
			return 0
		}
        //如果n超过队列一半，则由于并发访问导致h和t不一致，要重新开始。
		if n > uint32(len(pp.runq)/2) { // read inconsistent h and t
			continue
		}
        //从runq批量抓取任务
		for i := uint32(0); i < n; i++ {
			g := pp.runq[(h+i)%uint32(len(pp.runq))]
			batch[(batchHead+i)%uint32(len(batch))] = g
		}
		if atomic.CasRel(&pp.runqhead, h, h+n) { // cas-release, commits consume
			return n
		}
	}
}

```

从`n=n-n/2`我们可以得知，是获取一半数量的G。


通过`stealWork->runqsteal->runqgrab`的方法链路，完成了将其他P的本地队列G搬运到当前P的本地队列中的过程。


##### （六）总览


最后，我们用绘图来整体回顾`findRunnable`的执行流程。


![](https://img2024.cnblogs.com/blog/3542244/202411/3542244-20241117153406244-584832358.png)


### 2\.3\.2、execute()


当我们成功的通过`findRunnable()`找到了可以被执行的G的时候，就会对当前的G调用`execute()`方法，开始去调用这个G。



```
func execute(gp *g, inheritTime bool) {
	mp := getg().m


	//绑定G和M
	mp.curg = gp
	gp.m = mp
    //更改G的状态
	casgstatus(gp, _Grunnable, _Grunning)
	gp.waitsince = 0
	gp.preempt = false
	gp.stackguard0 = gp.stack.lo + stackGuard
	if !inheritTime {
        //更新P的调度次数
		mp.p.ptr().schedtick++
	}
	//....
	//执行G的任务
	gogo(&gp.sched)
}

```

可以看到`execute`的主要任务就是将当前的G和M进行绑定，即把G分配给这个线程M，然后调整它的状态为执行态，最后调用`gogo`方法完成对用户方法的运行。


### 2\.3\.3、mcall()


从2\.3\.2小节中我们知道，执行的execute函数完成了g0和g的切换，将对M的执行权交给了g，然后调用了`gogo`方法运行g。当需要重新将M的执行权从g切换到g0的时候，需要执行`mcall()`方法，完成切换。`mcall()`方法的作用我们在2\.1小节中提到过，该方法是通过汇编语言实现的，主要的作用是完成了对g的栈信息的保存、将当前堆栈从g切换到g0、在g0的栈上执行mcall方法中传入的`fn`回调函数。


什么时候调用`mcall()`，就涉及到我们在2\.2小节讲到了调度类型了。接下来我们通过源码一一分析。


#### 1、主动调度


主动调度是提供给用户的让权方法，执行的是runtime包下的`Gosched`方法。



```
func Gosched() {
	checkTimeouts()
	mcall(gosched_m)
}

```

Gosched方法就调用了mcall，并且传入回调函数`gosched_m`。



```
// Gosched continuation on g0.
func gosched_m(gp *g) {
	goschedImpl(gp, false)
}

func goschedImpl(gp *g, preempted bool) {
	//...
	casgstatus(gp, _Grunning, _Grunnable)// 将Goroutine状态从运行中更改为可运行状态
	//...

	dropg()//解绑G和M
	lock(&sched.lock)
	globrunqput(gp)//将G放入到全局队列中，等待下一次的调度
	unlock(&sched.lock)

	//...

	schedule()// 调用调度器，从全局队列或本地队列选择下一个Goroutine运行
}

```

`gosched_m`完成了对G的状态的转换，然后调用`dropg`将M和G解绑，再将G放回到全局队列里面，最终调用schedule进行新一轮的调度。


#### 2、被动调度


当当前G需要被被动调用的时候，就会调用`goprak()`，将其置为阻塞态，等待别人的唤醒。



```
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceReason traceBlockReason, traceskip int) {
	//...
	mcall(park_m)
}

// park continuation on g0.
func park_m(gp *g) {
	mp := getg().m

	trace := traceAcquire()

	casgstatus(gp, _Grunning, _Gwaiting)
	//...

	dropg()

	//...
	schedule()
}

```

`gopark`内部调用了`mcall(park_m)`，`park_m`将G的状态置为waiting，并且将M和G解绑，然后开启新一轮的调度。


进入等待的G需要被动的被其他事件唤醒，此时就会调用`goready`方法。



```
func goready(gp *g, traceskip int) {
	systemstack(func() {
		ready(gp, traceskip, true)
	})
}


//ready 函数的作用是将指定的 Goroutine (gp) 标记为“可运行”状态并将其放入运行队列。它会在 Goroutine 从等待（_Gwaiting）状态转换为可运行（_Grunnable）状态时使用，以确保调度器能够选择并执行它。
// Mark gp ready to run.
func ready(gp *g, traceskip int, next bool) {
	status := readgstatus(gp)

	// Mark runnable.
	mp := acquirem() // 获取当前线程（M），并禁止其被抢占，以避免将 P 错误地保留在本地变量中。
    //确认G的状态
	if status&^_Gscan != _Gwaiting {
		dumpgstatus(gp)
		throw("bad g->status in ready")
	}
	//...
	casgstatus(gp, _Gwaiting, _Grunnable)
	//....
    //将该G放入到当前P的运行队列
	runqput(mp.p.ptr(), gp, next)
    //检查是否有空闲的 P，若有则唤醒，以便它能够处理新加入的可运行 Goroutine。
	wakep()
    //释放当前 M 的锁，以重新允许抢占。
	releasem(mp)
}

```

`ready`方法会将G的状态重新切换成运行态，并且将G放入到P的运行队列里面。从代码中我们可以看到，被唤醒的G并不会立刻执行，而是加入到本地队列中等待下一次被调度。


#### 3、正常调度


假如G被正常的执行完毕，就会调用`goexit1()`方法完成g和g0的切换。



```
func goexit1() {
	//...
	mcall(goexit0)
}


// goexit continuation on g0.
func goexit0(gp *g) {
	gdestroy(gp)
	schedule()
}

```

最终，协程G被销毁，并且开启新一轮的调度。


#### 4、抢占调度


抢占调度最为复杂，因为它需要全局监控者m去检查所有的P是否被长期阻塞，这需要花时间去检索，而不能直接锁定到哪个P需要被抢占。全局监控者会调用`retake()`方法去检查，其流程如下：



```
//retake 函数用于在 Go 的调度器中处理一些调度策略，确保 Goroutine 的执行不被长时间阻塞。它通过检查所有的处理器 (P)，尝试中断过长的系统调用并在合适的条件下重新夺回 P 的控制权。
func retake(now int64) uint32 {
	n := 0
	lock(&allpLock)
	for i := 0; i < len(allp); i++ {
		pp := allp[i]
		if pp == nil {
			continue
		}
		pd := &pp.sysmontick
		s := pp.status
		sysretake := false
		if s == _Prunning || s == _Psyscall {
            //// 如果 `P` 的状态为 `_Prunning` 或 `_Psyscall`，则检查其运行时长。
			t := int64(pp.schedtick)
			if int64(pd.schedtick) != t {
				pd.schedtick = uint32(t)
				pd.schedwhen = now
			} else if pd.schedwhen+forcePreemptNS <= now {
                //超过最大运行时间，抢占P
				preemptone(pp)
				//如果处于系统调用状态，`preemptone()` 无法中断 P，因为没有 M 绑定到 P。
				sysretake = true
			}
		}
		if s == _Psyscall {
			// 如果 `P` 在系统调用中停留超过 1 个监控周期，则尝试收回。
			t := int64(pp.syscalltick)
			if !sysretake && int64(pd.syscalltick) != t {
				pd.syscalltick = uint32(t)
				pd.syscallwhen = now
				continue
			}
            //如果当前P的运行队列为空，切存在至少一个自旋的M，并且未超出等待时间则跳过回收
			if runqempty(pp) && sched.nmspinning.Load()+sched.npidle.Load() > 0 && pd.syscallwhen+10*1000*1000 > now {
				continue
			}
			// 为了获取 `sched.lock`，先释放 `allpLock`
			unlock(&allpLock)
			
            //回收操作...
            handoffp(pp)
		}
	}
	unlock(&allpLock)
	return uint32(n)
}

```


```
for i := 0; i < len(allp); i++ {
		pp := allp[i]
		if pp == nil {
			continue
		}

```

逐一的获取P，进行检查。



```
if s == _Prunning || s == _Psyscall {
            //// 如果 `P` 的状态为 `_Prunning` 或 `_Psyscall`，则检查其运行时长。
			t := int64(pp.schedtick)
			if int64(pd.schedtick) != t {
				pd.schedtick = uint32(t)
				pd.schedwhen = now
			} else if pd.schedwhen+forcePreemptNS <= now {
                //超过最大运行时间，抢占P
				preemptone(pp)
				//如果处于系统调用状态，`preemptone()` 无法中断 P，因为没有 M 绑定到 P。
				sysretake = true
			}
		}

```

当P的运行时间超过最大运行时间的时候，就会调用`preemptone`方法，尝试去抢占P。


值得注意的地方是，`preemptone`方法是设计成**“尽力而为”**的，因为并发的存在，**我们并不能确保它一定能通知到我们需要解绑的G**，因为可能会存在以下状况：


* 当我们尝试去发出抢占通知P上的G需要停止运行的时候，可能在发出通知的过程，这个G就完成运行，调用到下一个G了，我们可能会通知了错误的G。
* 当G进入到系统调用的状态的时候，P和M就会解绑，我们也通知不到G了。
* 就算通知到了目标的G，它也可能在执行newstack，此时会忽略请求。


因此，`preemptone`方法**只会尝试在自己未和M解绑以及m上的g此时不是g0的情况下，将`gp.preempt`置为true，表示发出了通知便返回true。具体的抢占将可能会在未来的某一时刻发生。**



```
if s == _Psyscall {
			// 如果 `P` 在系统调用中停留超过 1 个监控周期，则尝试收回。
			t := int64(pp.syscalltick)
			if !sysretake && int64(pd.syscalltick) != t {
				pd.syscalltick = uint32(t)
				pd.syscallwhen = now
				continue
			}
            //如果当前P的运行队列为空，切存在至少一个自旋的M，并且未超出等待时间则跳过回收
			if runqempty(pp) && sched.nmspinning.Load()+sched.npidle.Load() > 0 && pd.syscallwhen+10*1000*1000 > now {
				continue
			}
			// 为了获取 `sched.lock`，先释放 `allpLock`
			unlock(&allpLock)
			
            //回收操作...
    if atomic.Cas(&pp.status, s, _Pidle) {
        //....
        	handoffp(pp)
    }

		}

```

当满足以下三个条件的时候，就会执行抢占调度：


* p的本地队列有等待执行的G
* 当前没有空闲的p和m
* 执行系统调用的时间超过10ms


此时就会调用抢占调度，先将p的状态置为idle，表示可以被其他的M获取绑定，然后调用`handoffp`方法。



```
func handoffp(pp *p) {
	// handoffp must start an M in any situation where
	// findrunnable would return a G to run on pp.

	// if it has local work, start it straight away
	if !runqempty(pp) || sched.runqsize != 0 {
		startm(pp, false, false)
		return
	}
	// if there's trace work to do, start it straight away
	if (traceEnabled() || traceShuttingDown()) && traceReaderAvailable() != nil {
		startm(pp, false, false)
		return
	}
	// if it has GC work, start it straight away
	if gcBlackenEnabled != 0 && gcMarkWorkAvailable(pp) {
		startm(pp, false, false)
		return
	}
	// no local work, check that there are no spinning/idle M's,
	// otherwise our help is not required
	if sched.nmspinning.Load()+sched.npidle.Load() == 0 && sched.nmspinning.CompareAndSwap(0, 1) { // TODO: fast atomic
		sched.needspinning.Store(0)
		startm(pp, true, false)
		return
	}
	lock(&sched.lock)
	if sched.gcwaiting.Load() {
		pp.status = _Pgcstop
		sched.stopwait--
		if sched.stopwait == 0 {
			notewakeup(&sched.stopnote)
		}
		unlock(&sched.lock)
		return
	}
	if pp.runSafePointFn != 0 && atomic.Cas(&pp.runSafePointFn, 1, 0) {
		sched.safePointFn(pp)
		sched.safePointWait--
		if sched.safePointWait == 0 {
			notewakeup(&sched.safePointNote)
		}
	}
	if sched.runqsize != 0 {
		unlock(&sched.lock)
		startm(pp, false, false)
		return
	}
	// If this is the last running P and nobody is polling network,
	// need to wakeup another M to poll network.
	if sched.npidle.Load() == gomaxprocs-1 && sched.lastpoll.Load() != 0 {
		unlock(&sched.lock)
		startm(pp, false, false)
		return
	}

	// The scheduler lock cannot be held when calling wakeNetPoller below
	// because wakeNetPoller may call wakep which may call startm.
	when := nobarrierWakeTime(pp)
	pidleput(pp, 0)
	unlock(&sched.lock)

	if when != 0 {
		wakeNetPoller(when)
	}
}

```

当我们满足以下情况之一的时候，就会为当前的P新分配一个M进行调度：


* 全局队列不为空或者本地队列不为空，即有可以运行的G。
* 需要有trace去执行。
* 有垃圾回收的工作需要执行。
* 当前时刻没有自旋的线程M并且没有空闲的P（表示当前时刻任务繁忙）。
* 当前P是唯一在运行的P，并且有网络事件等待处理。


当满足五个条件之一的时候，都会进入到`startm()`方法中，为当前的P分配一个M。




---



```
func startm(pp *p, spinning, lockheld bool) {
	mp := acquirem()
	if !lockheld {
		lock(&sched.lock)
	}
	if pp == nil {
		if spinning {
		}
		pp, _ = pidleget(0)
		if pp == nil {
			if !lockheld {
				unlock(&sched.lock)
			}
			releasem(mp)
			return
		}
	}
	nmp := mget()
	if nmp == nil {
		id := mReserveID()
		unlock(&sched.lock)

		var fn func()
		if spinning {
			fn = mspinning
		}
		newm(fn, pp, id)

		if lockheld {
			lock(&sched.lock)
		}
		releasem(mp)
		return
	}
	//...
	releasem(mp)
}

```


```
if pp == nil {
		if spinning {
		}
		pp, _ = pidleget(0)
		if pp == nil {
			if !lockheld {
				unlock(&sched.lock)
			}
			releasem(mp)
			return
		}
	}

```

假如传入的pp是nil，那么会自动设置为空闲p队列中的第一个p，假如仍然为nil表示当前没有空闲的p，会退出方法。



```
nmp := mget()
	if nmp == nil {
		id := mReserveID()
		unlock(&sched.lock)

		var fn func()
		if spinning {
			fn = mspinning
		}
		newm(fn, pp, id)

		if lockheld {
			lock(&sched.lock)
		}
		releasem(mp)
		return
	}

```

然后会尝试获取当前的空闲的m，假如不存在则新创建一个m。


至此，关于GMP模型的节选部分的讲解就完成了，可能有许多我理解的不对的地方欢迎大家讨论，谢谢观看。


