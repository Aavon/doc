### channel使用tips

在使用channel进行并发调度的时候，注意以下几种不同状态的channel的行为：

未初始化：var c chan int = nil

无缓冲: c := make(chan int)

有缓冲: c := make(chan int, 1)



```go
func Case6() {
	c1 := make(chan int)
	// c1 := make(chan int, 1)
    // var c1 chan int = nil
	c1 <- 1
	fmt.Println(<-c1)
}

## 输出
未初始化：all goroutines are asleep - deadlock!
无缓冲：all goroutines are asleep - deadlock!
有缓冲区：1

func Case7() {
	c1 := make(chan int)
	// c1 := make(chan int, 1)
    // var c1 chan int = nil
	go func() {
		fmt.Println(<-c1)
	}()
	c1 <- 1
    time.Sleep(time.Second)
}

## 输出
未初始化：all goroutines are asleep - deadlock!
无缓冲：1
有缓冲区：1


```









###  				