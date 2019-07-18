### sync

sync包实现了一些基础的同步原语；更高级的同步机制官方建议使用channel来实现；

同时包含atomic包，实现数据的原子操作；

> 以下原语对象在参数传递时，切忌不可被拷贝：XXX must not be copied after first use.

#### Mutex

锁是sync包的核心概念，其他原语的实现到多都是基于Mutex的封装，golang在Mutex之前抽象了`Locker`接口;

```go
// A Locker represents an object that can be locked and unlocked.
type Locker interface {
	Lock()
	Unlock()
}
```

Mutex则是 `Locker`一种基本实现；

golang中Mutex是根据`自旋锁`实现，并在此基础上增加了优化策略来解决过度等待和饥饿问题；

> 自旋锁的一种简单表示（来自一个大佬）

```
for {atomic.CAS == true?break:continue}
```

自旋锁的基本描述中是需要基于atomic操作来保证排他性，不停的进行CAS尝试，成功也就表示锁成功，CAS操作的对象的值在0/1之间不停切换；

*Mutex定义*

```go
// A Mutex is a mutual exclusion lock.
// The zero value for a Mutex is an unlocked mutex.
//
// A Mutex must not be copied after first use.
type Mutex struct {
  // 状态字段：/唤醒状态/模式状态/lock状态
	state int32
  // goroutine阻塞唤醒状态量？
	sema  uint32
}
```

*相关概念*

被锁状态（mutexLocked）：锁处于锁住状态；

唤醒状态（mutexWoken状态）：Unlock操作后唤醒某个goroutine，并发状态下，防止并发唤醒多个goroutine;

正常模式：等待队列中的goroutine根据FIFO的顺序依次竞争获取锁；此模式下，新加的goroutine在等待队列队列非空的情况下仍尝试获取锁（4次自旋尝试等待）获取到锁；

饥饿模式（mutexStarving状态）：

触发条件是某个goroutine的等待时长超过1ms；

新加的goroutine也不会尝试去获取锁，不自旋等待；

Unlock操作直接交给等待队列的第一个goroutine；

这种模式是为了保证公平性，保证在队尾的goroutine也有可能获取到锁；



> 省略了部分并发检查的逻辑

![](http://ww4.sinaimg.cn/large/006tNc79gy1g53tk5345fj315a0dcq4l.jpg)

![](http://ww1.sinaimg.cn/large/006tNc79gy1g53u5gw9f0j30wk07wdgc.jpg)

> 这里不仅仅是对Lock状态的操作需要CAS，Mutex的所有状态更新都要保证CAS，如果CAS失败则要考虑Mutex状态已经被其他goroutine更新，代码中通过`old = m.state`来获取最新的状态

*Lock*

> 可对照源码阅读：src/sync/mutex.go

```go
func (m *Mutex) Lock() {
		// 第一次尝试获取锁，成功则直接退出
  	atomic.CompareAndSwapInt32
  
  	// 初始化阻塞策略的相关变量
  	// waitStartTime 获取锁所等待的时长
  	var waitStartTime int64
  	// starving 是否为饥饿模式
  	starving := false
    // awoke 是否为唤醒状态
  	awoke := false
  	// iter 自旋次数
  	iter := 0
  	// old 当前状态 = m.state
  	old := m.state
    for {
      if 满足自旋条件（被锁状态&非饥饿模式&自旋次数不超过限制） {
        if 处于非 && CAS操作成功 {
          已进入唤醒状态（当前goroutine抢占成功）
        }
      }
      自旋一定时间
      自旋次数++
      old取最新值
      continue
    }
  	// new为下一个状态
  	new := old
    if 非饥饿模式 {
      new := mutexLocked(尝试去获取锁)
    }
    if 当前状态为Locked或者饥饿模式 {
				new += 1 << mutexWaiterShift
    }
  	if starving && old&mutexLocked != 0 {
      // 下一个状态进入饥饿模式
			new |= mutexStarving
		}
    if awoke {
      // 重置唤醒状态标志
			new &^= mutexWoken
    }
  	// state未被其他goroutine更新
  	if atomic.CompareAndSwapInt32(&m.state, old, new) {
      	// 通过old判断是不是获取锁成功，成功就退出了
      	if old&(mutexLocked|mutexStarving) == 0 {
					break // locked the mutex with CAS
				}
        // 进入等待队列,非第一次则放入到等待队列首部（保证公平性）
        // 等待时长超过starvationThresholdNs（1ms）
        starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
      	// 更新当前状态
      	old = m.state
        if 非饥饿模式 {
						delta := int32(mutexLocked - 1<<mutexWaiterShift)
          if !starving || old>>mutexWaiterShift == 1 {
            // Exit starvation mode.
            // Critical to do it here and consider wait time.
            // Starvation mode is so inefficient, that two goroutines
            // can go lock-step infinitely once they switch mutex
            // to starvation mode.
            delta -= mutexStarving
          }
        }
      	awoke = true
				iter = 0
    }
}
```



*Unlock*

```go
func (m *Mutex) Unlock() {
    // 释放锁标志
  	new := atomic.AddInt32(&m.state, -mutexLocked)
    // 重复Unlock检查
  	if (new+mutexLocked)&mutexLocked == 0 {
      throw("sync: unlock of unlocked mutex")
    }
  	if 非饥饿模式 {
       // 检查已唤醒其他goroutine
       // 随便唤醒一个
       runtime_Semrelease(&m.sema, false)
    } else {
       // 唤醒阻塞队列中的第一个
       runtime_Semrelease(&m.sema, true)
    }
}
```

总计一下，Mutex设计中几个要点：

1.新加goroutine首次获取锁失败放在队首，之后Lock失败则放入队尾（新增的goroutine正在CPU执行，把锁给他们会有很大的优势）；

2.任何一个goroutine尝试获取锁的时长超过1ms，则进入饥饿模式；饥饿模式下，新加goroutine不会自旋等待，不会尝试获取锁；Unlock之后换新队首的goroutine；

#### RWMutex

读写锁，基于Mutex实现，如果只是使用Lock/Unlock，和Mutex是等效的；

实现上主要依赖的是Mutex和一些状态量，使得锁可以被任意数量的Reader或者单个Writer获取；

Lock:不仅要等待m.Lock()，还要判断readerCount是不是0；

RLock:只需要判断有没有Writer在等待（readerCount为特殊值）；

> 这个很重要：只要有Writer被阻塞，新到的Reader也会被阻塞，直到Unlock;

```go
// If a goroutine holds a RWMutex for reading and another goroutine might
// call Lock, no goroutine should expect to be able to acquire a read lock
// until the initial read lock is released. In particular, this prohibits
// recursive read locking. This is to ensure that the lock eventually becomes
// available; a blocked Lock call excludes new readers from acquiring the
// lock.
type RWMutex struct {
	w           Mutex  // held if there are pending writers
	writerSem   uint32 // semaphore for writers to wait for completing readers
	readerSem   uint32 // semaphore for readers to wait for completing writers
	readerCount int32  // number of pending readers
	readerWait  int32  // number of departing readers
}

// 示例
func Case9() {
	var rm sync.RWMutex
	rm.RLock()
	println("read locked 1.")
	go func() {
		rm.Lock()
		println("locked.")
	}()
	go func() {
		time.Sleep(time.Second)
		println("2 read locked.")
		rm.RLock()
		println("read locked 2.")
	}()
	time.Sleep(time.Second * 10)
}
/*
read locked 1.
2 read locked.
*/

```

#### Once

基于Mutex和一个标志字段done来实现，在`Do()`被调用一次之后done被置为1，之后不在触发f的执行；

```go
type Once struct {
	m    Mutex
	done uint32
}

// 示例
func Case1() {
	var once sync.Once
	f := func() {
		println("1")
	}
	once.Do(f)
	once.Do(f)
}
```

#### WaitGroup

对于并发执行多个任务的场景，WaitGroup可用于等待所有任务执行结束；

```go
// A WaitGroup waits for a collection of goroutines to finish.
// The main goroutine calls Add to set the number of
// goroutines to wait for. Then each of the goroutines
// runs and calls Done when finished. At the same time,
// Wait can be used to block until all goroutines have finished.
//
// A WaitGroup must not be copied after first use.
type WaitGroup struct {
	noCopy noCopy

	// 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
	// 64-bit atomic operations require 64-bit alignment, but 32-bit
	// compilers do not ensure it. So we allocate 12 bytes and then use
	// the aligned 8 bytes in them as state, and the other 4 as storage
	// for the sema.
	state1 [3]uint32
}

// 示例
func Case7() {
	var wg sync.WaitGroup
	wg.Add(2)
	go func() {
		time.Sleep(time.Second)
		println("done 1")
		wg.Done() // wg.Add(-1)
	}()
	go func() {
		println("done 2")
		wg.Done()
	}()
	wg.Wait()
	println("all done.")
}
```

#### Cond

Cond实现了类似广播/单播的同步场景；

广播：多个goroutine阻塞等待，广播导致这些goroutine都被唤醒；

单播：多个goroutine阻塞等待，单播导致这些goroutine中的某一个被唤醒（一般是FIFO）；

```go
// Cond implements a condition variable, a rendezvous point
// for goroutines waiting for or announcing the occurrence
// of an event.
//
// Each Cond has an associated Locker L (often a *Mutex or *RWMutex),
// which must be held when changing the condition and
// when calling the Wait method.
//
// A Cond must not be copied after first use.
type Cond struct {
	noCopy noCopy

	// L is held while observing or changing the condition
	L Locker

	notify  notifyList
	checker copyChecker
}

// 示例
func Case2() {
	cond := sync.NewCond(&sync.Mutex{})
	go func() {
		for {
      // 必须要先获取锁
			cond.L.Lock()
			cond.Wait()
			println(1)
			cond.L.Unlock()
		}
	}()
	go func() {
		for {
			cond.L.Lock()
			cond.Wait()
			println(2)
			cond.L.Unlock()
		}
	}()
	go func() {
		for {
      // 可以不用获取锁
			cond.Signal()
			// cond.Broadcast()
			time.Sleep(time.Second)
		}
	}()
	select {}
}
```

####Pool

Pool实现了一个对象池，用于共享一些临时对象，避免频繁创建小对象给GC和内存带来压力；

结合bytes.Buffer更能实现共享的内存池，应对一般的高并发场景；

```go
// An appropriate use of a Pool is to manage a group of temporary items
// silently shared among and potentially reused by concurrent independent
// clients of a package. Pool provides a way to amortize allocation overhead
// across many clients.
//
// An example of good use of a Pool is in the fmt package, which maintains a
// dynamically-sized store of temporary output buffers. The store scales under
// load (when many goroutines are actively printing) and shrinks when
// quiescent.
//
// On the other hand, a free list maintained as part of a short-lived object is
// not a suitable use for a Pool, since the overhead does not amortize well in
// that scenario. It is more efficient to have such objects implement their own
// free list.
//
// A Pool must not be copied after first use.
type Pool struct {
	noCopy noCopy

	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
	localSize uintptr        // size of the local array

	// New optionally specifies a function to generate
	// a value when Get would otherwise return nil.
	// It may not be changed concurrently with calls to Get.
	New func() interface{}
}

// 示例
func Case8() {
	pool := sync.Pool{
		New: func() interface{} {
			return bytes.Buffer{}
		},
	}
	buf := bytes.Buffer{}
	buf.WriteString("abc")
	buf.Reset()
	pool.Put(buf)
	rec := pool.Get().(bytes.Buffer)
	println(rec.String())
}

```

#### Map

并发安全的Map

空间换时间， 通过冗余的两个数据结构(read、dirty),实现加锁对性能的影响

动态调整，miss次数多了之后，将dirty数据提升为read

优先从read读取、更新、删除，因为对read的读取不需要锁

> TODO

#### sync/atomic

原子操作，具体实现主要在汇编代码；

使用LOCK汇编指令通过锁总线/MESI协议实现缓存刷新；

包括Load，Store，Add，Cas，Swap操作

```go
// atomic.AddXXX()
TEXT runtime∕internal∕atomic·Xadd(SB), NOSPLIT, $0-12
	MOVL	ptr+0(FP), BX
	MOVL	delta+4(FP), AX
	MOVL	AX, CX
	LOCK
	XADDL	AX, 0(BX)
	ADDL	CX, AX
	MOVL	AX, ret+8(FP)
	RET

// atomic.StoreXXX()
TEXT runtime∕internal∕atomic·Store(SB), NOSPLIT, $0-8
	MOVL	ptr+0(FP), BX
	MOVL	val+4(FP), AX
	XCHGL	AX, 0(BX)
	RET

// bool Cas(int32 *val, int32 old, int32 new)
// Atomically:
//	if(*val == old){
//		*val = new;
//		return 1;
//	}else
//		return 0;
TEXT runtime∕internal∕atomic·Cas(SB), NOSPLIT, $0-13
	MOVL	ptr+0(FP), BX
	MOVL	old+4(FP), AX
	MOVL	new+8(FP), CX
	LOCK
	CMPXCHGL	CX, 0(BX)
	SETEQ	ret+12(FP)
	RET
```









