#Timer

*案例分析*

```go
func FailureCase() {
	i := 0
	go func() {
		for {
			i = i + 1
			time.Sleep(time.Second)
      if i > 5 {
        break
      }
		}
	}()
	for {
		exit := false
		select {
    // 避免使用for
		case <-time.Tick(time.Millisecond):
			if i > 5 {
				exit = true
			}
		}
		if exit {
			break
		}
	}
}
```



> 问题：如果FailureCase被频繁调用？

 

> 结论：容器CPU使用率峰值翻倍（或者更高）而且居高不下！

[pprof分析](file:///Users/codoon/Documents/工作文档/pprof002.svg)

![](http://ww3.sinaimg.cn/large/006tNc79gy1g383ykop09j31gi0u04a9.jpg)



### time包

###### 标准常量定义

```
ANSIC       = "Mon Jan _2 15:04:05 2006"
UnixDate    = "Mon Jan _2 15:04:05 MST 2006"
RubyDate    = "Mon Jan 02 15:04:05 -0700 2006"
RFC822      = "02 Jan 06 15:04 MST"
RFC822Z     = "02 Jan 06 15:04 -0700" // RFC822 with numeric zone
RFC850      = "Monday, 02-Jan-06 15:04:05 MST"
RFC1123     = "Mon, 02 Jan 2006 15:04:05 MST"
RFC1123Z    = "Mon, 02 Jan 2006 15:04:05 -0700" // RFC1123 with numeric zone
RFC3339     = "2006-01-02T15:04:05Z07:00"
RFC3339Nano = "2006-01-02T15:04:05.999999999Z07:00"
Kitchen     = "3:04PM"
// Handy time stamps.
Stamp      = "Jan _2 15:04:05"
StampMilli = "Jan _2 15:04:05.000"
StampMicro = "Jan _2 15:04:05.000000"
StampNano  = "Jan _2 15:04:05.000000000"
```



```
stdDay                                         // "2"
stdUnderDay                                    // "_2"
stdZeroDay                                     // "02"
stdHour                  = iota + stdNeedClock // "15"
stdHour12                                      // "3"
stdZeroHour12                                  // "03"
stdMinute                                      // "4"
stdZeroMinute                                  // "04"
stdSecond                                      // "5"
stdZeroSecond                                  // "05"
stdLongYear              = iota + stdNeedDate  // "2006"
stdYear                                        // "06"
stdPM                    = iota + stdNeedClock // "PM"
```



> "2006-01-02 15:04:05"(format.go: nextStdChunk)



###### 时间的表示

系统时钟：

CLOCK_REALTIME（wall clock）：当前时间（系统展示的时间，可同步，可修改）

CLOCK_MONOTONIC（monotonic time）：单调时间，系统启动后每个计时器中断+1

```go
type Time struct {
	// wall and ext encode the wall time seconds, wall time nanoseconds,
	// and optional monotonic clock reading in nanoseconds.
	//
	// From high to low bit position, wall encodes a 1-bit flag (hasMonotonic),
	// a 33-bit seconds field, and a 30-bit wall time nanoseconds field.
	// The nanoseconds field is in the range [0, 999999999].
	// If the hasMonotonic bit is 0, then the 33-bit field must be zero
	// and the full signed 64-bit wall seconds since Jan 1 year 1 is stored in ext.
	// If the hasMonotonic bit is 1, then the 33-bit field holds a 33-bit
	// unsigned wall seconds since Jan 1 year 1885, and ext holds a
	// signed 64-bit monotonic clock reading, nanoseconds since process start.
	wall uint64
  // 从程序启动开始计算
	ext  int64

	// loc specifies the Location that should be used to
	// determine the minute, hour, month, day, and year
	// that correspond to this Time.
	// The nil location means UTC.
	// All UTC times are represented with loc==nil, never loc==&utcLoc.
	loc *Location
}
```

![](http://ww2.sinaimg.cn/large/006tNc79gy1g383ujd412j31780ccq41.jpg)

```go
func Case1() {
	// time.Sleep(time.Second / 2)
	now := time.Now()
	valueNow := reflect.ValueOf(now)
	wall := valueNow.FieldByName("wall")
	ext := valueNow.FieldByName("ext")
	println(wall.Uint(), ext.Int())
}

Output:
--
13776715687332558152 565063
--
13776715794735670552 595682
```



###### 时间的计算

```go
// 加
func (t Time) Add(d Duration) Time
func (t Time) AddDate(years int, months int, days int) Time

// 减
func (t Time) Sub(u Time) Duration

// “余”（时区问题）
func (t Time) Truncate(d Duration) Time

// 等
 func (t Time) Equal(u Time) bool

```

```go
// 格式化
func (t Time) Format(layout string) string
func Parse(layout, value string) (Time, error)
func ParseInLocation(layout, value string, loc *Location) (Time, error)
```

```go
// 序列化和反序列化方法
func (t Time) MarshalBinary() ([]byte, error)
func (t Time) MarshalJSON() ([]byte, error)
func (t Time) MarshalText() ([]byte, error)

func (t *Time) UnmarshalBinary(data []byte) error
func (t *Time) UnmarshalJSON(data []byte) error
func (t *Time) UnmarshalText(data []byte) error

func (t Time) String() string
```



###### 定时器（timer）

我们经常使用的是time包暴露出来的方法，但是在time包中仅包含一些对timer操作的封装，在`runtime/time.go`包含绝大部分的底层实现；

###### time包

**Timer**

```go
type Timer struct {
  // 时间通道
	C <-chan Time
  // timer更底层的封装,在runtime/time.go实现
	r runtimeTimer
}
```

```go
//  创建并启动（结束会向C中写入当前时间）
func NewTimer(d Duration) *Timer
//  重置计时（Stop --> 创建）
func (t *Timer) Reset(d Duration) bool
//  停止并从全局时间堆中移除
func (t *Timer) Stop() bool
```

```go
func NewTimer(d Duration) *Timer {
	c := make(chan Time, 1)
	t := &Timer{
		C: c,
		r: runtimeTimer{
			when: when(d),
			f:    sendTime,
			arg:  c,
		},
	}
	startTimer(&t.r)
	return t
}
```

golang中最基础的就是Timer，在Timer的基础上封装了After，Tick，Sleep等场景；

**After**

在给定时间d后触发，只想f函数或者默认函数；例如超时场景

```go
func AfterFunc(d Duration, f func()) *Timer

func After(d Duration) <-chan Time
```

**Tick**

在给定时间d的间隔内，循环触发；只支持执行默认函数（写channel）；比如限速场景

```go
// Failure中的例子(慎用)
func Tick(d Duration) <-chan Time

// 创建Tick
func NewTicker(d Duration) *Ticker
// 关闭Tick，结束循环
func (t *Ticker) Stop()

```

```go
func NewTicker(d Duration) *Ticker {
	if d <= 0 {
		panic(errors.New("non-positive interval for NewTicker"))
	}
	// Give the channel a 1-element time buffer.
	// If the client falls behind while reading, we drop ticks
	// on the floor until the client catches up.
	c := make(chan Time, 1)
	t := &Ticker{
		C: c,
		r: runtimeTimer{
			when:   when(d),
      // 和NewTimer仅仅相差这一个字段
      // 在timerproc会使用这个字段来判断底层timer是否进行reset并重新加入计时循环
			period: int64(d),
			f:      sendTime,
			arg:    c,
		},
	}
	startTimer(&t.r)
	return t
}
```

> 这里可以发现在Tick的场景中，由于period被赋值，底层timer会一直生效，所以运行一段时间之后，全局的时间堆回存在大量的timer（timer泄漏），去check和维护这个时间堆，会占用大量的cpu资源；



*最佳实践*

```go
var ticker = time.NewTicker(100 * time.Millisecond)
// 使用defer在函数退出时关闭timer
defer ticker.Stop()
var counter = 0
for {
    select {
    case <-serverDone:
        return
    case <-ticker.C:
        counter += 1
    }
}
```

**Sleep**

```go
func Sleep(d Duration)
```

> 具体的实现在runtime/time.go中，使用了比较hack的方式 — go:linkname, 达到跨包访问；



```go
//go:linkname timeSleep time.Sleep
```

![](http://ww3.sinaimg.cn/large/006tNc79gy1g383z3tezrj314609iq43.jpg)

所以简单来说，Sleep做的事情是，将当前goroutine置入waiting状态，再由定时器来唤醒；



###### runtime/time.go

```go
// runtimeTimer 与 timer定义完全一致，在golang中可以直接强转
// runtime/time.go
type timer struct {
  tb *timersBucket // the bucket the timer lives in
	i  int           // heap index
  // 触发时间（now + duration)
	when   int64
  period int64
  // 触发动作（默认为sendTime）
	f      func(interface{}, uintptr) // NOTE: must not be closure
  // f的参数（默认为时间通道）
	arg    interface{}
  // timer更新的时候会修改
	seq    uintptr
}

// 时间堆结构 -- 由单独的timerproc协程维护
type timersBucket struct {
	lock         mutex
  // 当前timerproc执行goroutine
	gp           *g
  // timer goroutine：是否已启动
	created      bool
  // timer goroutine：是否Sleep
	sleeping     bool
  // timer goroutine：是否暂停（len(t) == 0）
	rescheduling bool
  // timer goroutine：Sleep恢复时间
	sleepUntil   int64
  // timer goroutine：挂起/唤醒 状态量
	waitnote     note
	t            []*timer
}

// 实际存储结构,加入填充字段
// runtime/time.go
// timersLen == 64
var timers [timersLen]struct {
	timersBucket

	// The padding should eliminate false sharing
	// between timersBucket values.
  // 
	pad [cpu.CacheLinePadSize - unsafe.Sizeof(timersBucket{})%cpu.CacheLinePadSize]byte
}

```



> false sharing: CPU的缓存系统通常是以缓存行（cacheline,一般为64字节）来读取数据的，如果有两个进程的数据同时落到了一个cacheline, 一个进程的数据被修改了，整个cacheline需要重新加载，在高并发的场景中，这种相互之间的影响是不可忽略的；
>
> 而对于，定时器这样可能会频繁更新的数据结构，单独存在于一个或者多个cacheline是很有必要的；



**时间堆结构（timersBucket）**

全局`timers`的长度为64，每个timer为独立的`timersBucket`结构，每个`timersBucket`独立维护一个`timer`堆（以数组结构存储）;

![timers](http://ww1.sinaimg.cn/large/006tNc79gy1g38prmea4pj30i00hywf8.jpg)

在每个timersBucket中，timer满足四叉堆数据结构，按`广度优先`的顺序存储在数组中；以及有如下特性：

```
1.父节点index = （当前节点index - 1）/ 4
2.每次调整之后，index为0的节点对应的timer为优先级最高的（when最小）；
3.层数越大的timer，when越大（index越大的timer，when越大）,方便查找when值最小的timer；
```

**调整算法（siftupTimer & siftdownTimer）**

调整涉及到两种场景：新增和删除；

*新增*

— timerBucket分配

```go
func addtimer(t *timer) {
	tb := t.assignBucket()
	lock(&tb.lock)
	ok := tb.addtimerLocked(t)
	unlock(&tb.lock)
	if !ok {
		badTimer()
	}
}

func (t *timer) assignBucket() *timersBucket {
  // 和M相关的
	id := uint8(getg().m.p.ptr().id) % timersLen
	t.tb = &timers[id].timersBucket
	return t.tb
}

func (tb *timersBucket) addtimerLocked(t *timer) bool {
	// when must never be negative; otherwise timerproc will overflow
	// during its delta calculation and never expire other runtime timers.
	if t.when < 0 {
		t.when = 1<<63 - 1
	}
	t.i = len(tb.t)
	tb.t = append(tb.t, t)
	if !siftupTimer(tb.t, t.i) {
		return false
	}
	if t.i == 0 {
    // 以下状态由timerproc更新
		// siftup moved to top: new earliest deadline.
		if tb.sleeping && tb.sleepUntil > t.when {
			tb.sleeping = false
			notewakeup(&tb.waitnote)
		}
		if tb.rescheduling {
			tb.rescheduling = false
			goready(tb.gp, 0)
		}
		if !tb.created {
      // 初次使用timerBucket,触发对应timerproc
			tb.created = true
			go timerproc(tb)
		}
	}
	return true
}
```

— siftupTimer

![](http://ww1.sinaimg.cn/large/006tNc79gy1g38u62m0bwj30s61060uj.jpg)

```go
func siftupTimer(t []*timer, i int) bool {
	if i >= len(t) {
		return false
	}
	when := t[i].when
	tmp := t[i]
	for i > 0 {
		p := (i - 1) / 4 // parent
		if when >= t[p].when {
			break
		}
		t[i] = t[p]
		t[i].i = i
		i = p
	}
	if tmp != t[i] {
		t[i] = tmp
		t[i].i = i
	}
	return true
}
```

*删除*

— 删除元素

将last元素调整到已删除的index位置，调整数组的长度，以达到修改删除元素的目的；

```go
func (tb *timersBucket) deltimerLocked(t *timer) (removed, ok bool) {
	// t may not be registered anymore and may have
	// a bogus i (typically 0, if generated by Go).
	// Verify it before proceeding.
	i := t.i
	last := len(tb.t) - 1
	if i < 0 || i > last || tb.t[i] != t {
		return false, true
	}
	if i != last {
		tb.t[i] = tb.t[last]
		tb.t[i].i = i
	}
	tb.t[last] = nil
	tb.t = tb.t[:last]
	ok = true
  // i对应的是原堆中最后一个元素
	if i != last {
		if !siftupTimer(tb.t, i) {
			ok = false
		}
		if !siftdownTimer(tb.t, i) {
			ok = false
		}
	}
	return true, ok
}
```



— siftupTimer & siftdownTimer

> 为什么需要siftupTimer？

i对应的是原堆的最后一个元素，因该是属于when最大的一个层次的timers；但是存在的情况是，删除的元素和last在同一层，交换值后不满足父子关系；

>siftdownTimer



![](http://ww1.sinaimg.cn/large/006tNc79gy1g391j6v4wcj30ug0nkmya.jpg)

```go
func siftdownTimer(t []*timer, i int) bool {
	n := len(t)
	if i >= n {
		return false
	}
	when := t[i].when
	tmp := t[i]
	for {
		c := i*4 + 1 // left child
		c3 := c + 2  // mid child
		if c >= n {
			break
		}
		w := t[c].when
		if c+1 < n && t[c+1].when < w {
			w = t[c+1].when
			c++
		}
		if c3 < n {
			w3 := t[c3].when
			if c3+1 < n && t[c3+1].when < w3 {
				w3 = t[c3+1].when
				c3++
			}
			if w3 < w {
				w = w3
				c = c3
			}
		}
		if w >= when {
			break
		}
		t[i] = t[c]
		t[i].i = i
		i = c
	}
	if tmp != t[i] {
		t[i] = tmp
		t[i].i = i
	}
	return true
}
```



**时间检查（timerproc）**

每一个timerBucket会有一个单独的timer goroutine来维护,所以并不是每一个timer对应一个goroutine；

waiting：无timer，timer goroutine置入waiting状态，等待重新调度；

sleeping(挂起)：有timer等待触发但不是现在，timer goroutine进入短暂挂起；

```go
// 时间堆结构 -- 由单独的timerproc协程维护
type timersBucket struct {
	lock         mutex
  // 当前timerproc执行goroutine
	gp           *g
  // timer goroutine：是否已启动
	created      bool
  // timer goroutine：是否挂起
	sleeping     bool
  // timer goroutine：是否需要重新调度（goready）
	rescheduling bool
  // timer goroutine：挂起结束时间
	sleepUntil   int64
  // timer goroutine：挂起/唤醒 状态量（默认note_cleared）
	waitnote     note
	t            []*timer
}
```

> noteclear/ notetsleepg / notewakeup 底层原理和gopark/goready一致，只是notetsleepg支持定时唤醒；

![](http://ww3.sinaimg.cn/large/006tNc79gy1g3953fepmmj30zf0inwki.jpg)



### 总结

1. 所有timer由golang底层统一管理；

2. 底层数据结构为4叉堆（时间堆），方便查找最早需要触发的timer；

3. 时间堆的数量固定为64，每一个时间堆对应一个goroutine（timer goroutine）;

4. 并发性能优化：timer goroutine只在必要时执行，混存优化等；

参考：

https://github.com/cch123/golang-notes/blob/master/timer.md



TODO：

1. for循环更新变量，变量更新无法读

2. 时间轮算法（定时器）





