## netpoller

*背景介绍*

I/O多路复用模型（I/O Multiplexing）:

**select**

阻塞，直到有FD准备好，FD数量有`FD_SETSIZE`限制

```c
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds,
 struct timeval *timeout);

// readfds,writefds,exceptfds 需要检查的FDs
// nfds 比最大的FD大1（减少内核比较次数）
// timeout 最大等待时间
// fd_set 位掩码表示FD集合
```

**poll**

和select类似，在传递FD方式上不同

```c
int poll(struct pollfd fds[], nfds_t nfds, int timeout);

// fds 需要检查的FDs和事件（events,revents）,结果写入fds
// nfds fds中的FD数量
// timeout 最大等待时间

struct pollfd {
 int fd; /* File descriptor */
 short events; /* Requested events bit mask */
 short revents; /* Returned events bit mask */
};
```

**epoll**

区分`水平触发`和`边缘触发`

```c
// 创建epoll示例：红黑树，就绪队列，返回epoll实例的FD
int epoll_create(int size);
int epoll_create1(int flag); // 与epoll_create1类似，对epfd有一定的控制

// FD注册
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *ev);
// epfd epoll_create返回的FD
// op 操作类型 EPOLL_CTL_ADD EPOLL_CTL_MOD EPOLL_CTL_DEL
// fd 操作对象FD
// ev 关注的事件类型(位掩码)

struct epoll_event {
 uint32_t events; /* epoll events (bit mask) */
 epoll_data_t data; /* User data */
};

typedef union epoll_data {
 void *ptr; /* Pointer to user-defined data */
 int fd; /* File descriptor */
 uint32_t u32; /* 32-bit integer */
 uint64_t u64; /* 64-bit integer */
} epoll_data_t;

// 获取就绪的FD
int epoll_wait(int epfd, struct epoll_event *evlist, int maxevents, int timeout);
// evlist,maxevents指定返回的event的存储空间和数量

// 自动销毁
```

epoll相对于其他I/O多路复用模型的优势：

1.不需要每次调用传递FD（用户态/内核态数据拷贝），epoll_ctl将FD添加到内核数据空间中；

2.epoll_wait的效率更高，不需要对比所有的FD,只需要从就绪队列中获取数据即可;



> 常见I/O模型：阻塞，非阻塞，I/O多路复用，信号驱动，异步I/O



#### 分析实例：File.Read

在unix/linux平台上，netpoller是基于epoll模型来实现的，一下分析也是限定于此；

以简单的文件读取（unix|linux平台）为例，分析从代码层面开始是怎么一步步使用netpoller的。

```
## 用户代码
func main() {
	f, err := os.Open("test.txt")
	if err != nil {
		log.Fatalln(err)
	}
	buf := make([]byte, 10)
	f.Read(buf)
	log.Println(string(buf))
}
```

使用`os.Open`创建一个File实例，核心是获取到文件描述符（pfd）

```go
type File struct {
	*file // os specific
}

// unix平台实现
// ## os/file.go
type file struct {
	pfd         poll.FD
	name        string
	dirinfo     *dirInfo // nil unless directory being read
	nonblock    bool     // whether we set nonblocking mode
	stdoutOrErr bool     // whether this is stdout or stderr
}

/*os/file.go*/
func (f *File) Read(b []byte) (n int, err error) {
	if err := f.checkValid("read"); err != nil {
		return 0, err
	}
	n, e := f.read(b)
	return n, f.wrapErr("read", e)
}
```

```go
/*os/file_unix.go(unix平台)*/
func (f *File) read(b []byte) (n int, err error) {
	n, err = f.pfd.Read(b)
	runtime.KeepAlive(f)
	return n, err
}
```

调用`Read`函数，实际上调用了底层的`FD.Read`，其实现如下：

```go
/*internal/poll/fd_unix.go*/
func (fd *FD) Read(p []byte) (int, error) {
	if err := fd.readLock(); err != nil {
		return 0, err
	}
	defer fd.readUnlock()
	if len(p) == 0 {
		// If the caller wanted a zero byte read, return immediately
		// without trying (but after acquiring the readLock).
		// Otherwise syscall.Read returns 0, nil which looks like
		// io.EOF.
		// TODO(bradfitz): make it wait for readability? (Issue 15735)
		return 0, nil
	}
	if err := fd.pd.prepareRead(fd.isFile); err != nil {
		return 0, err
	}
	if fd.IsStream && len(p) > maxRW {
		p = p[:maxRW]
	}
	for {
        // 系统调用 -- 读取数据
		n, err := syscall.Read(fd.Sysfd, p)
        // 出现错误 -- 可能是一般错误也可能是数据未准备好的特殊错误（EAGAIN）
		if err != nil {
			n = 0
            // syscall.EAGAIN错误 & 可以使用netpoller
            // 比如：pollable := kind == kindOpenFile || kind == kindPipe || kind == kindNonBlock
            // 那么进入非阻塞的I/O流程
			if err == syscall.EAGAIN && fd.pd.pollable() {
				if err = fd.pd.waitRead(fd.isFile); err == nil {
					continue
				}
			}

			// On MacOS we can see EINTR here if the user
			// pressed ^Z.  See issue #22838.
			if runtime.GOOS == "darwin" && err == syscall.EINTR {
				continue
			}
		}
		err = fd.eofError(n, err)
		return n, err
	}
}
```

```go
/*internal/poll/fd_poll_runtime.go*/
func (pd *pollDesc) waitRead(isFile bool) error {
	return pd.wait('r', isFile)
}

func (pd *pollDesc) wait(mode int, isFile bool) error {
	if pd.runtimeCtx == 0 {
		return errors.New("waiting for unsupported file type")
	}
	res := runtime_pollWait(pd.runtimeCtx, mode)
	return convertErr(res, isFile)
}
```

```go
/*runtime/netpoll.go*/
//go:linkname poll_runtime_pollWait internal/poll.runtime_pollWait
func poll_runtime_pollWait(pd *pollDesc, mode int) int {
	err := netpollcheckerr(pd, int32(mode))
	if err != 0 {
		return err
	}
	// As for now only Solaris and AIX use level-triggered IO.
	if GOOS == "solaris" || GOOS == "aix" {
		netpollarm(pd, mode)
	}
	for !netpollblock(pd, int32(mode), false) {
		err = netpollcheckerr(pd, int32(mode))
		if err != 0 {
			return err
		}
		// Can happen if timeout has fired and unblocked us,
		// but before we had a chance to run, timeout has been reset.
		// Pretend it has not happened and retry.
	}
	return 0
}

// 关键步骤
// 这是一个阻塞调用：如果IO准备好则退出，否则挂起
// returns true if IO is ready, or false if timedout or closed
// waitio - wait only for completed IO, ignore errors
func netpollblock(pd *pollDesc, mode int32, waitio bool) bool {
	gpp := &pd.rg
	if mode == 'w' {
		gpp = &pd.wg
	}
	
    // 如果已经准备好，return true
    // 如果没有准备好，gopark
	// set the gpp semaphore to WAIT
	for {
		old := *gpp
		if old == pdReady {
			*gpp = 0
			return true
		}
		if old != 0 {
			throw("runtime: double wait")
		}
		if atomic.Casuintptr(gpp, 0, pdWait) {
			break
		}
	}

    // waitio -- false
    // 挂起当前goroutine
	// need to recheck error states after setting gpp to WAIT
	// this is necessary because runtime_pollUnblock/runtime_pollSetDeadline/deadlineimpl
	// do the opposite: store to closing/rd/wd, membarrier, load of rg/wg
	if waitio || netpollcheckerr(pd, mode) == 0 {
		gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5)
	}
    // 等待IO准备好，goroutine被唤醒，代码从这里恢复执行
    // 状态置为pdReady
    // 代码退出到系统调用 -- 获取数据
	// be careful to not lose concurrent READY notification
	old := atomic.Xchguintptr(gpp, 0)
	if old > pdWait {
		throw("runtime: corrupted polldesc")
	}
	return old == pdReady
}

```

以上的代码，我们看到一个简单的`File.Read`是怎样一步一步从将当前FD添加到epoll模型，并将当前G挂起；

那么，进行更加深入的分析，我们就要知道一下几个问题：

1. Go是如何初始化epoll?(epoll_ctl)
2. 文件FD是如何添加/删除到epoll中？（epoll_ctl）
3. Go中是怎么获取到IO准备就绪事件的?（epoll_wait）
4. Go是怎么唤醒对应被挂起的goroutine？（重点）

> 需要提前明确的是：Go和epoll实例交互依旧是通过三个固定函数进行的，以系统调用的方式实现；



#### 1.初始化

```go
/*internal/poll/fd_poll_runtime.go*/

// 被动初始化(once）
var serverInit sync.Once

func (pd *pollDesc) init(fd *FD) error {
	serverInit.Do(runtime_pollServerInit)
	ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
	if errno != 0 {
		if ctx != 0 {
			runtime_pollUnblock(ctx)
			runtime_pollClose(ctx)
		}
		return syscall.Errno(errno)
	}
	pd.runtimeCtx = ctx
	return nil
}

/*runtime/netpoll.go*/

//go:linkname poll_runtime_pollServerInit internal/poll.runtime_pollServerInit
func poll_runtime_pollServerInit() {
    // 根据不同平台，会有不同的netpoll实现
	netpollinit()
	atomic.Store(&netpollInited, 1)
}
```



```go
/*runtime/netpoll_epoll.go*/

var (
    // 全局的epoll文件描述符
	epfd int32 = -1 // epoll descriptor
)

func netpollinit() {
    // 先调用epollcreate1创建一个epoll实例，flag为_EPOLL_CLOEXEC优化了epfd在竞争和跨线程的使用
	epfd = epollcreate1(_EPOLL_CLOEXEC)
	if epfd >= 0 {
		return
	}
    // epollcreate1如果不成功，则使用epollcreate，size=1024(size参数在之后2.6.8之后已经被忽略)
	epfd = epollcreate(1024)
	if epfd >= 0 {
		closeonexec(epfd)
		return
	}
	println("runtime: epollcreate failed with", -epfd)
	throw("runtime: netpollinit failed")
}
```



> epoll_create1:https://linux.die.net/man/2/epoll_create1

在runtime层，如果有pollDesc被初始化则会被动的进行netpoller的初始化，然后调用平台相关的netpollinit的实现；

首先`epollcreate`和`epollcreate`的实现在汇编中（runtime、sys_linux_amd64.s），我们可以忽略；

同时，在Go中有一个全局的epfd，所以可以说runtime只会创建一个epoll实例来管理所有的IO事件；



#### 2.添加和删除

netpoller初始化后，就涉及到如何向epoll中添加和删除FD，当然我们知道底层肯定是通过系统调用epoll_ctl来实现的；

*添加*

```go
/*runtime/netpoll.go*/
// 参考: func (pd *pollDesc) init(fd *FD) error
ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))

//go:linkname poll_runtime_pollOpen internal/poll.runtime_pollOpen
func poll_runtime_pollOpen(fd uintptr) (*pollDesc, int) {
    // pollDesc链表单独管理，不能被GC释放,复用
	pd := pollcache.alloc()
	lock(&pd.lock)
	if pd.wg != 0 && pd.wg != pdReady {
		throw("runtime: blocked write on free polldesc")
	}
	if pd.rg != 0 && pd.rg != pdReady {
		throw("runtime: blocked read on free polldesc")
	}
    // 初始化
	pd.fd = fd
	pd.closing = false
	pd.rseq++
	pd.rg = 0
	pd.rd = 0
	pd.wseq++
	pd.wg = 0
	pd.wd = 0
	unlock(&pd.lock)

	var errno int32
    //  注册到netpoller,调用平台实现
	errno = netpollopen(fd, pd)
	return pd, int(errno)
}
```

```go
/*runtime/netpoll_epoll.go*/

func netpollopen(fd uintptr, pd *pollDesc) int32 {
	var ev epollevent
    // 普通数据 | 边缘触发 
    // 参考下表
	ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
	*(**pollDesc)(unsafe.Pointer(&ev.data)) = pd
	return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}
```



*删除*

```go
/*internal/poll/fd_poll_runtime.go*/
func (pd *pollDesc) close() {
	if pd.runtimeCtx == 0 {
		return
	}
	runtime_pollClose(pd.runtimeCtx)
	pd.runtimeCtx = 0
}

/*runtime/netpoll.go*/
func poll_runtime_pollClose(pd *pollDesc) {
	if !pd.closing {
		throw("runtime: close polldesc w/o unblock")
	}
	if pd.wg != 0 && pd.wg != pdReady {
		throw("runtime: blocked write on closing polldesc")
	}
	if pd.rg != 0 && pd.rg != pdReady {
		throw("runtime: blocked read on closing polldesc")
	}
    // 从netpoller中删除，调用平台实现
	netpollclose(pd.fd)
    // 释放pollDesc对象
	pollcache.free(pd)
}
```

```go
/*runtime/netpoll_epoll.go*/

func netpollclose(fd uintptr) int32 {
	var ev epollevent
    // 汇编实现，系统调用
	return -epollctl(epfd, _EPOLL_CTL_DEL, int32(fd), &ev)
}
```

> pollcache: 是通过链表接口实现的alloc对应LPOP(没有就创建一个)，free对应LPUSH，以达到复用pollDesc对象；

对netpoller中FD的操作仅发生在pollDesc初始化和关闭时，在epoll的实现中，通过epoll_ctl系统调用实现；

![](   http://storage.aaronzz.xyz/docs/epoll_event_type.jpg   )

#### 3.事件获取

```go
/*runtime/netpoll_epoll.go*/

// polls for ready network connections
// returns list of goroutines that become runnable
// block == true 一直阻塞 block == false 不阻塞
func netpoll(block bool) gList {
	if epfd == -1 {
		return gList{}
	}
	waitms := int32(-1)
	if !block {
		waitms = 0
	}
    // 单次最多返回128个事件
	var events [128]epollevent
retry:
    // 系统调用
    // epfd 全局的epoll文件描述符
    // events 返回的事件列表
	n := epollwait(epfd, &events[0], int32(len(events)), waitms)
	if n < 0 {
		if n != -_EINTR {
			println("runtime: epollwait on fd", epfd, "failed with", -n)
			throw("runtime: netpoll failed")
		}
		goto retry
	}
	var toRun gList
	for i := int32(0); i < n; i++ {
		ev := &events[i]
		if ev.events == 0 {
			continue
		}
		var mode int32
        // 读就绪 -- 和添加FD时event类型对应
		if ev.events&(_EPOLLIN|_EPOLLRDHUP|_EPOLLHUP|_EPOLLERR) != 0 {
			mode += 'r'
		}
        // 写就绪 -- 和添加FD时event类型对应
		if ev.events&(_EPOLLOUT|_EPOLLHUP|_EPOLLERR) != 0 {
			mode += 'w'
		}
		if mode != 0 {
			pd := *(**pollDesc)(unsafe.Pointer(&ev.data))
			// 查找可唤醒的goroutines
			netpollready(&toRun, pd, mode)
		}
	}
    // epoll_wait在某些信号下也会返回
	if block && toRun.empty() {
		goto retry
	}
	return toRun
}
```

>1.Go中获取epoll时间只有阻塞和非阻塞两种（非timeout）;
>
>2.events的最大长度为128；

在非阻塞模式下，调用epoll_wait获取关注的event，并唤醒对应的对应goroutine；需要注意的是EPOLLERR也会唤醒，为了让错误能传递出来；

在阻塞模式下，重复非阻塞模式下的流程，知道有goroutine被唤醒为止；

runtime中通过调用netpoll来主动获取已经就绪的epoll_event,那么他的触发时机常见的有一下几个场景：

1.sysmon函数定时触发

sysmon作为调度器很重要的一环，会循环处理调度任务，包括调用netpoll函数；

```go
if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
    atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
    list := netpoll(false) // non-blocking - returns list of goroutines
    if !list.empty() {
        incidlelocked(-1)
        injectglist(&list)
        incidlelocked(1)
    }
}
```

2.P查找可运行的G时

如果P在本地和全局队列中没有找到可用的G，会触发netpoll;

```go
func findrunnable() (gp *g, inheritTime bool)

if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
    if list := netpoll(false); !list.empty() { // non-blocking
        gp := list.pop()
        injectglist(&list)
        casgstatus(gp, _Gwaiting, _Grunnable)
        if trace.enabled {
            traceGoUnpark(gp, 0)
        }
        return gp, false
    }
}
```

3.STW恢复时

STW期间可能已经有IO准备就绪，所以在STW结束后，会立即触发netpoll;

```go
func startTheWorldWithSema(emitTraceEvent bool) int64 {
	_g_ := getg()

	_g_.m.locks++ // disable preemption because it can be holding p in a local var
	if netpollinited() {
		list := netpoll(false) // non-blocking
		injectglist(&list)
	}
	... ...
}
```



#### 4.关联被挂起的G

在`netpoll`函数中会调用`netpollready`来唤醒对应goroutine,那么从epoll的event到goroutine它们是怎么关联起来的呢？

```go
/*runtime/netpoll_epoll.go*/
func netpoll(block bool) gList {
   ...
    if mode != 0 {
        // epoll_event的用户数据部分为pollDesc
        pd := *(**pollDesc)(unsafe.Pointer(&ev.data))
        netpollready(&toRun, pd, mode)
    }
```



```go
/*runtime/netpoll.go*/

func netpollready(toRun *gList, pd *pollDesc, mode int32) {
	var rg, wg *g
	if mode == 'r' || mode == 'r'+'w' {
		rg = netpollunblock(pd, 'r', true)
	}
	if mode == 'w' || mode == 'r'+'w' {
		wg = netpollunblock(pd, 'w', true)
	}
	if rg != nil {
		toRun.push(rg)
	}
	if wg != nil {
		toRun.push(wg)
	}
}

// 查找可唤醒的G
func netpollunblock(pd *pollDesc, mode int32, ioready bool) *g {
	gpp := &pd.rg
	if mode == 'w' {
		gpp = &pd.wg
	}

	for {
        // old保存了关联的G
		old := *gpp
		if old == pdReady {
			return nil
		}
		if old == 0 && !ioready {
			// Only set READY for ioready. runtime_pollWait
			// will check for timeout/cancel before waiting.
			return nil
		}
		var new uintptr
		if ioready {
			new = pdReady
		}
        // 将pdRead保存在rg/wg
		if atomic.Casuintptr(gpp, old, new) {
			if old == pdReady || old == pdWait {
				old = 0
			}
            // 返回关联的G
			return (*g)(unsafe.Pointer(old))
		}
	}
}
```

到此，会有一个疑问为什么`wg`和`rg`分别保存的关联的G，因为从之前的代码中我们看到这个字段也有存pdWait的状态？那么，我们来梳理一下这两个字段在整个流程中的变化；

```go
## 1.pollDesc初始化 wg,rg == 0
/*runtime/netpoll.go*/
// 参考: func (pd *pollDesc) init(fd *FD) error
ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
// 参考 func poll_runtime_pollOpen(fd uintptr) (*pollDesc, int)
pd.rg = 0
pd.wg = 0

## 2.读取数据得到EAGAIN,进行wait wg,rg --> pdWait --> G
/*runtime/netpoll.go*/
// 参考 func netpollblock(pd *pollDesc, mode int32, waitio bool) bool
for {
		old := *gpp
		if old == pdReady {
			*gpp = 0
			return true
		}
		if old != 0 {
			throw("runtime: double wait")
		}
    	// 更新为pdWait状态
		if atomic.Casuintptr(gpp, 0, pdWait) {
			break
		}
	}
	if waitio || netpollcheckerr(pd, mode) == 0 {
        // 挂起
        // lock参数正好为rg或者wg的地址
		gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5)
	}

// gopark定义 
// unlockf为回调函数，传入参数为lock
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int)

// 进过gopark操作后回调
// gp当前goroutine
// gpp rg或者wg的地址
func netpollblockcommit(gp *g, gpp unsafe.Pointer) bool {
    // 关键步骤：将当前G保存到rg或者wg中
	r := atomic.Casuintptr((*uintptr)(gpp), pdWait, uintptr(unsafe.Pointer(gp)))
	if r {
		// Bump the count of goroutines waiting for the poller.
		// The scheduler uses this to decide whether to block
		// waiting for the poller if there is nothing else to do.
		atomic.Xadd(&netpollWaiters, 1)
	}
	return r
}

## 3.之后为等待IO就绪，进入G的唤醒流程
```

所以，runtime之所以可以快速找到对应的G,有一下几个关键步骤：

1. 等待IO就绪，挂起当前G之后将当前G保存到rg或者wg;
2. 添加FD到epoll实例中，用户自定义数据部分为整个epollDesc;
3. IO就绪，返回的epoll_event信息包含epollDesc，同时也就能快速找到对应的rg或者wg;

**总结**

基于netpoller，Go语言让程序员可以用阻塞的思想来编写IO操作的代码，但是底层却自动实现了多路复用的机制；

G对象直接和FD关联，且在整个流程都携带了G的地址信息，可以快速查找并唤醒响应的G;

同时，由于goroutine相较于线程有天生的优势，调度开销小；





















