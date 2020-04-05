### Defer

defer是golang中一个很独特的关键字，用于创建延迟执行函数；经常被用来做处理函数退出之前的资源释放或者执行公共的代码逻辑；

在使用defer我们也经常遇到以下的一些问题，先看两个典型的案例：

> 如果可以提前了解golang闭包相关概念可以更好的理解defer的特性

**1.defer/return问题**

```go
//defer:  2
//1
func DeferReturn1() int {
	var i int
	defer func() {
		i++
		fmt.Println("defer: ", i)
	}()
	i++
	return i
}

//defer:  2
//2
func DeferReturn2() (i int) {
	defer func() {
		i++
		fmt.Println("defer: ", i)
	}()
	i++
	return i
}

func TestReturn(t *testing.T) {
	fmt.Println(DeferReturn1())
	fmt.Println(DeferReturn2())
}
```

可以理解为先return，在执行执行defer，在没有提前声明返回参数时，会创建一个匿名参数；

**2.参数预计算**

```
// 0
func DeferArg1() {
	var i int
	defer fmt.Println(i)
	i++
}

// 1
func DeferArg2() {
	var i int
	defer func() {
		fmt.Println(i)
	}()
	i++
}

func TestArg(t *testing.T) {
	//DeferArg1()
	DeferArg2()
}
```

这里更多的是受“闭包”特性的影响，函数+作用于内的变量；对于直接的函数调用（DeferArg1）来说参数通过复制直接传递到了所调用的函数，而对于匿名函数（DeferArg2）来说，可以当做一个闭包来看待，`i`所处的上下文属于外层函数（DeferArg2），和匿名函数在执行时，使用的`i`是同一个；



#### defer的实现(1.14)

简单描述一下1.13的实现，golang会在堆、栈上维护一个defer链表，在返回时按照栈的顺序递归执行每个defer；而保存没存每个defer对象需要包含以下信息：

function pointer, arguments, closure information等

这样导致defer在1.13上的表现并不满意，平均的处理时间为35ns;



在1.14中加入了open-coded开放编码模式，直接将将延迟defer函数的代码添加到函数尾部，来减少直接递归调用的开销；

1.14的优化，核心在ssa构建过程（看不懂的），我们主要关注defer对象的处理流程上的优化；

以下是1.13的实现：

```go
// cmd/compile/internal/gc/ssa.go
// exit processes any code that needs to be generated just before returning.
// It returns a BlockRet block that ends the control flow. Its control value
// will be set to the final memory state.
func (s *state) exit() *ssa.Block {
	if s.hasdefer {
		s.rtcall(Deferreturn, true, nil)
	}
    ...
}

// runtime/panic.go
func deferreturn(arg0 uintptr) {
	gp := getg()
	d := gp._defer
	if d == nil {
		return
	}
	sp := getcallersp()
	if d.sp != sp {
		return
	}
	
	// Moving arguments around.
	//
	// Everything called after this point must be recursively
	// nosplit because the garbage collector won't know the form
	// of the arguments until the jmpdefer can flip the PC over to
	// fn.
    // 参数设置
	switch d.siz {
	case 0:
		// Do nothing.
	case sys.PtrSize:
		*(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
	default:
		memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz))
	}
	fn := d.fn
	d.fn = nil
	gp._defer = d.link
    // 服用defer对象(当前goroutine)
	freedefer(d)
    // 执行defer函数， 并递归下一个defer对象
	jmpdefer(fn, uintptr(unsafe.Pointer(&arg0)))
}

type _defer struct {
	siz     int32 // includes both arguments and results
	started bool
	heap    bool
	sp      uintptr // sp at time of defer
	pc      uintptr
	fn      *funcval
	_panic  *_panic // panic that is running defer
	link    *_defer
}

```

在ssa构建阶段，编译器根据函数是否有defer语句，决定是否在exit阶段插入deferreturn代码；

而deferreturn的代码也主要分为三个阶段：参数处理，释放defer对象，执行defer函数并递归到下一个defer对象；



以下是1.14的实现：

```go
func deferreturn(arg0 uintptr) {
	gp := getg()
	d := gp._defer
	if d == nil {
		return
	}
	sp := getcallersp()
	if d.sp != sp {
		return
	}
	// 是否为open-coded模式
	if d.openDefer {
		// 遍历执行defer函数
		done := runOpenDeferFrame(gp, d)
		if !done {
			throw("unfinished open-coded defers in deferreturn")
		}
		gp._defer = d.link
		// 
		freedefer(d)
		return
	}
	
    // 如果无法满足open-coded模式的条件，流程和1.13一致
	// Moving arguments around.
	//
	// Everything called after this point must be recursively
	// nosplit because the garbage collector won't know the form
	// of the arguments until the jmpdefer can flip the PC over to
	// fn.
	switch d.siz {
	case 0:
		// Do nothing.
	case sys.PtrSize:
		*(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
	default:
		memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz))
	}
	fn := d.fn
	d.fn = nil
	gp._defer = d.link
	freedefer(d)
	// If the defer function pointer is nil, force the seg fault to happen
	// here rather than in jmpdefer. gentraceback() throws an error if it is
	// called with a callback on an LR architecture and jmpdefer is on the
	// stack, because the stack trace can be incorrect in that case - see
	// issue #8153).
	_ = fn.fn
	jmpdefer(fn, uintptr(unsafe.Pointer(&arg0)))
}

func runOpenDeferFrame(gp *g, d *_defer) bool {
	done := true
    // 保存了所有oped-coded defer的信息
	fd := d.fd

	// Skip the maxargsize
	_, fd = readvarintUnsafe(fd)
	deferBitsOffset, fd := readvarintUnsafe(fd)
	nDefers, fd := readvarintUnsafe(fd)
	deferBits := *(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset)))
	
    // 遍历恢复函数指针，参数，上下文信息等，并执行对应defer函数
	for i := int(nDefers) - 1; i >= 0; i-- {
		// read the funcdata info for this defer
		var argWidth, closureOffset, nArgs uint32
		argWidth, fd = readvarintUnsafe(fd)
		closureOffset, fd = readvarintUnsafe(fd)
		nArgs, fd = readvarintUnsafe(fd)
        // deferBits对应为0，表示对应的defer函数不需要执行(包含if,case条件)
		if deferBits&(1<<i) == 0 {
			for j := uint32(0); j < nArgs; j++ {
				_, fd = readvarintUnsafe(fd)
				_, fd = readvarintUnsafe(fd)
				_, fd = readvarintUnsafe(fd)
			}
			continue
		}
		closure := *(**funcval)(unsafe.Pointer(d.varp - uintptr(closureOffset)))
		d.fn = closure
		deferArgs := deferArgs(d)
		// If there is an interface receiver or method receiver, it is
		// described/included as the first arg.
		for j := uint32(0); j < nArgs; j++ {
			var argOffset, argLen, argCallOffset uint32
			argOffset, fd = readvarintUnsafe(fd)
			argLen, fd = readvarintUnsafe(fd)
			argCallOffset, fd = readvarintUnsafe(fd)
			memmove(unsafe.Pointer(uintptr(deferArgs)+uintptr(argCallOffset)),
				unsafe.Pointer(d.varp-uintptr(argOffset)),
				uintptr(argLen))
		}
        // deferBits对应位置0
		deferBits = deferBits &^ (1 << i)
		*(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset))) = deferBits
		p := d._panic
		reflectcallSave(p, unsafe.Pointer(closure), deferArgs, argWidth)
		if p != nil && p.aborted {
			break
		}
		d.fn = nil
		// These args are just a copy, so can be cleared immediately
		memclrNoHeapPointers(deferArgs, uintptr(argWidth))
		if d._panic != nil && d._panic.recovered {
			done = deferBits == 0
			break
		}
	}

	return done
}

type _defer struct {
	siz     int32 // includes both arguments and results
	started bool
	heap    bool
	// openDefer indicates that this _defer is for a frame with open-coded
	// defers. We have only one defer record for the entire frame (which may
	// currently have 0, 1, or more defers active).
	openDefer bool
	sp        uintptr  // sp at time of defer
	pc        uintptr  // pc at time of defer
	fn        *funcval // can be nil for open-coded defers
	_panic    *_panic  // panic that is running defer
	link      *_defer
	
    // open-coded模式下，保存所有的defer函数信息（只有一个defer对象）
	// If openDefer is true, the fields below record values about the stack
	// frame and associated function that has the open-coded defer(s). sp
	// above will be the sp for the frame, and pc will be address of the
	// deferreturn call in the function.
	fd   unsafe.Pointer // funcdata for the function associated with the frame
	varp uintptr        // value of varp for the stack frame
	// framepc is the current pc associated with the stack frame. Together,
	// with sp above (which is the sp associated with the stack frame),
	// framepc/sp can be used as pc/sp pair to continue a stack trace via
	// gentraceback().
	framepc uintptr
}

```

在1.14中大部分的流程并没有改变，主要是在open-coded的基础上对deferreturn函数做了一定的调整；

在open-coded模式下，仅有**一个defer对象**，`fd`、`varp`、`	framepc uintptr`等字段保存了所有的defer函数信息（函数指针、参数、上下文等），在`runOpenDeferFrame`仅通过一个**遍历**就执行了所有的defer函数；



**deferBits（延迟比特）**

根据open-code的描述，编译器需要在编译阶段将相关的defer函数代码插入到函数尾部，而特定的defer函数可能处于if,case分支中，需要在运行时确定是否需要执行defer函数；

deferBits就是为了这个问题，在执行过程中动态标记对应位的0/1值，来影响最终defer函数的执行情况；

由deferBits的初始化可以看出，其长度为8，也就是说open-coded模式，需要defer函数的总数不能超过8；

```go
// cmd/compile/internal/gc/ssa.go
bitvalue := s.constInt8(types.Types[TUINT8], 1<<uint(index))
	newDeferBits := s.newValue2(ssa.OpOr8, types.Types[TUINT8], s.variable(&deferBitsVar, types.Types[TUINT8]), bitvalue)
```



**open-coded模式条件**

这部分规则在ssa的相关代码中，感兴趣可以去查看，这里直接参考大佬们的结论；

1. defer总数不超过8个

2. return * defer < 15

3. 非循环内使用defer

4. gcflags包含 “N” ，则取消优化



#### 总结

相对于1.13，1.14中新增了open-coded模式，在ssa阶段进行了优化，将递归的调用通过单次循环来完成；虽然使用open-coded模式有一定的条件，但是也可以看出大部分情况下是能满足这个条件的；



**参考**

https://github.com/golang/proposal/blob/master/design/34481-opencoded-defers.md

http://xiaorui.cc/archives/6579

