#### 大量TIME_WAIT的惨案

##### 1.背景介绍

XX系统（新上线）：实时处理商家订单和客服成单信息；

部署情况：灰测阶段，单实例；



业务逻辑中通过redis缓存顾客和客服状态，订阅所有的付款订单消息，并同步更新顾客和客服状态；

保证数据一致性，通过redis实现对状态数据加锁；

redis操作主要包含：GET /SET(NX)

错误信息：

```
err:dial tcp 10.19.0.214:6379: connect: cannot assign requested address
redsync: failed to acquire lock
```

![](http://storage.aaronzz.xyz/docs/%7B93899BEB-637A-45C1-B429-459F1D161722%7D_20191212035704.jpg)

![](http://storage.aaronzz.xyz/docs/%7B36CD1CFD-FB34-47E7-AA2A-4773B3EC3657%7D_20191212151255.jpg)

翻译一下就是：本地已经没有端口用于建立tcp链接，同时CPU使用率居高不下,内存占用很高；



> 事后分析发现：上下文切换，系统调用都很高



##### 2.问题分析过程

*a. “连接泄露”*

首先检查代码，都是通过连接池获取连接在使用完成后归还；

已排除



*b. 连接池参数设置问题*

```
1. cannot assign requested address
2. TCP链接客户端的问题: 端口不足
3. 本地端口动态分配: 说明连接建立太多了
4. 订单量~连接数: 在并发量大的时候确实是有这个可能的，同时服务初始化过程中并没有设置MaxActive
5. 本地压测：发现即使设置了MaxActive,TIME_WAIT仍然在增加，如图1
6. 分析TCP连接：TIME_WAIT处于主动关闭连接方的最后一个阶段(等待2ms在进入CLOSE阶段，并释放资源)
7. 主动关闭连接：TIME_WAIT出现在主动关闭连接一方，而且是正常的链接关闭，在连接池中基本上只会出现两种情况：空闲连接过多&空闲连接生存周期过长
8. 发现max_idle=10,idle_timeout = 120，增大max_idle,压测结果如图2；
```



原因分析：

```go
func (p *Pool) put(pc *poolConn, forceClose bool) error {
	p.mu.Lock()
	if !p.closed && !forceClose {
		pc.t = nowFunc()
		p.idle.pushFront(pc)
        // 如果MaxIdle设置过小，会导致连接不停的插入/删除,并close
		if p.idle.count > p.MaxIdle {
			pc = p.idle.back
			p.idle.popBack()
		} else {
			pc = nil
		}
	}

	if pc != nil {
        // 关闭连接
		p.mu.Unlock()
		pc.c.Close()
		p.mu.Lock()
		p.active--
	}

	if p.ch != nil && !p.closed {
		p.ch <- struct{}{}
	}
	p.mu.Unlock()
	return nil
}
```





*图1：*

![](http://storage.aaronzz.xyz/docs/max_active_20191217194515.jpg)



*图2：*

![](http://storage.aaronzz.xyz/docs/max_idle_20191217194515.jpg)



##### 3.问题修复

修改前：

```
[[redis]]
    alias = "ach-cache"
    address = "r-k2j...:6379"
    password = ""
    db = 0
    connect_timeout = 5
    read_timeout = 5
    write_timeout = 5
    wait = true
    max_idle = 10
    idle_timeout = 120
```

修改后：

```
[[redis]]
    alias = "ach-cache"
    address = "r-k2jv7re...:6379"
    password = ""
    db = 0
    connect_timeout = 5
    read_timeout = 5
    write_timeout = 5
    wait = true
    max_active = 1000
    max_idle = 500
    idle_timeout = 120
```





