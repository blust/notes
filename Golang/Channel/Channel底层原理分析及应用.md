
# channel

## 底层结构
```
type hchan struct {
    qcount   uint           // channel中的元素个数
    dataqsiz uint           // channel中循环队列的长度
    buf      unsafe.Pointer // channel缓冲区数据指针
    elemsize uint16            // buffer中每个元素的大小
    closed   uint32            // channel是否已经关闭，0未关闭
    elemtype *_type // channel中的元素的类型
    sendx    uint   // channel发送操作处理到的位置
    recvx    uint   // channel接收操作处理到的位置
    recvq    waitq  // 等待接收的sudog（sudog为封装了goroutine和数据的结构）队列由于缓冲区空间不足而阻塞的Goroutine列表
    sendq    waitq  // 等待发送的sudogo队列，由于缓冲区空间不足而阻塞的Goroutine列表

    lock mutex   // 一个轻量级锁
}

详见:src/runtime/channel.go
```
- 环形队列缓存：当channel缓冲区未满且仍有send请求的时候，该队列将用于缓存此时收到的消息。
- 双向链表：当channel缓冲区已满且仍有send请求或者已空但仍有recv请求的时候，channel将进入阻塞状态。通过该双向链表存放阻塞的Goroutine信息，以sudog的形式包裹G。
- 锁：go语言的channel基于悲观锁实现。

![image](http://m.weizhi.cc/upload/202205/28/202205280122521701.png)

![image](http://m.weizhi.cc/upload/202205/28/202205280122529743.png)

![image](http://m.weizhi.cc/upload/202205/28/202205280122535568.png)

注：引用自 http://m.weizhi.cc/tech/detail-321185.html


## 创建channel
```

```

- **核心源码**
```go
var c *hchan
switch {
case mem == 0: 
    // Queue or element size is zero. 不带缓冲区
    c = (*hchan)(mallocgc(hchanSize, nil, true)) 
    // Race detector uses this location for synchronization. 只需要给hchan分配内存空间(调用一次mallocgc分配一段连续的内存空间)。
    c.buf = c.raceaddr()
case elem.ptrdata == 0: 
    // Elements do not contain pointers. 带缓冲区且不包括指针类型
    // Allocate hchan and buf in one call. 同时给hchan和环形队列缓存buf分配一段连续的内存空间。
    c = (*hchan)(mallocgc(hchanSize+mem, nil, true)) 
    c.buf = add(unsafe.Pointer(c), hchanSize)
default: 
    // Elements contain pointers. 带缓冲区且包括指针类型：
    c = new(hchan) 分别给hchan和环形队列缓存buf分配不同的内存空间。
    c.buf = mallocgc(mem, elem, true)
}
```

## channel发送数据

### 源码分析
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// 如果 channel 是 nil
	if c == nil {
		// 不能阻塞，直接返回 false，表示未发送成功
		if !block {
			return false
		}
		// 当前 goroutine 被挂起
		gopark(nil, nil, "chan send (nil chan)", traceEvGoStop, 2)
		throw("unreachable")
	}

	// 省略 debug 相关……

	// 对于不阻塞的 send，快速检测失败场景
	//
	// 如果 channel 未关闭且 channel 没有多余的缓冲空间。这可能是：
	// 1. channel 是非缓冲型的，且等待接收队列里没有 goroutine
	// 2. channel 是缓冲型的，但循环数组已经装满了元素
	if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
		(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	// 锁住 channel，并发安全
	lock(&c.lock)

	// 如果 channel 关闭了
	if c.closed != 0 {
		// 解锁
		unlock(&c.lock)
		// 直接 panic
		panic(plainError("send on closed channel"))
	}

	// 如果接收队列里有 goroutine，直接将要发送的数据拷贝到接收 goroutine
	if sg := c.recvq.dequeue(); sg != nil {
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	// 对于缓冲型的 channel，如果还有缓冲空间
	if c.qcount < c.dataqsiz {
		// qp 指向 buf 的 sendx 位置
		qp := chanbuf(c, c.sendx)

		// ……

		// 将数据从 ep 处拷贝到 qp
		typedmemmove(c.elemtype, qp, ep)
		// 发送游标值加 1
		c.sendx++
		// 如果发送游标值等于容量值，游标值归 0
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		// 缓冲区的元素数量加一
		c.qcount++

		// 解锁
		unlock(&c.lock)
		return true
	}

	// 如果不需要阻塞，则直接返回错误
	if !block {
		unlock(&c.lock)
		return false
	}

	// channel 满了，发送方会被阻塞。接下来会构造一个 sudog

	// 获取当前 goroutine 的指针
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}

	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.selectdone = nil
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil

	// 当前 goroutine 进入发送等待队列
	c.sendq.enqueue(mysg)

	// 当前 goroutine 被挂起
	goparkunlock(&c.lock, "chan send", traceEvGoBlockSend, 3)

	// 从这里开始被唤醒了（channel 有机会可以发送了）
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	if gp.param == nil {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		// 被唤醒后，channel 关闭了。坑爹啊，panic
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	// 去掉 mysg 上绑定的 channel
	mysg.c = nil
	releaseSudog(mysg)
	return true
}
```

## channel接收数据

### 源码分析

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // 省略 debug 内容 …………
    
    // 如果是一个 nil 的 channel
	if c == nil {
		// 如果不阻塞，直接返回 (false, false)
		if !block {
			return
		}
		// 否则，接收一个 nil 的 channel，goroutine 挂起
		gopark(nil, nil, "chan receive (nil chan)", traceEvGoStop, 2)
		// 不会执行到这里
		throw("unreachable")
	}

	// 在非阻塞模式下，快速检测到失败，不用获取锁，快速返回
	// 当我们观察到 channel 没准备好接收：
	// 1. 非缓冲型，等待发送列队 sendq 里没有 goroutine 在等待
	// 2. 缓冲型，但 buf 里没有元素
	// 之后，又观察到 closed == 0，即 channel 未关闭。
	// 因为 channel 不可能被重复打开，所以前一个观测的时候 channel 也是未关闭的，
	// 因此在这种情况下可以直接宣布接收失败，返回 (false, false)
	if !block && (
	    c.dataqsiz == 0 &&
	    c.sendq.first == nil ||
	    c.dataqsiz > 0 &&
	    atomic.Loaduint(&c.qcount) == 0) &&
	    atomic.Load(&c.closed) == 0 {
		return
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	// 加锁
	lock(&c.lock)

	// channel 已关闭，并且循环数组 buf 里没有元素
	// 这里可以处理非缓冲型关闭 和 缓冲型关闭但 buf 无元素的情况
	// 也就是说即使是关闭状态，但在缓冲型的 channel，
	// buf 里有元素的情况下还能接收到元素
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(unsafe.Pointer(c))
		}
		// 解锁
		unlock(&c.lock)
		if ep != nil {
			// 从一个已关闭的 channel 执行接收操作，且未忽略返回值
			// 那么接收的值将是一个该类型的零值
			// typedmemclr 根据类型清理相应地址的内存
			typedmemclr(c.elemtype, ep)
		}
		// 从一个已关闭的 channel 接收，selected 会返回true
		return true, false
	}

	// 等待发送队列里有 goroutine 存在，说明 buf 是满的
	// 这有可能是：
	// 1. 非缓冲型的 channel
	// 2. 缓冲型的 channel，但 buf 满了
	// 针对 1，直接进行内存拷贝（从 sender goroutine -> receiver goroutine）
	// 针对 2，接收到循环数组头部的元素，并将发送者的元素放到循环数组尾部
	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	// 缓冲型，buf 里有元素，可以正常接收
	if c.qcount > 0 {
		// 直接从循环数组里找到要接收的元素
		qp := chanbuf(c, c.recvx)

		// …………

		// 代码里，没有忽略要接收的值，不是 "<- ch"，而是 "val <- ch"，ep 指向 val
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// 清理掉循环数组里相应位置的值
		typedmemclr(c.elemtype, qp)
		// 接收游标向前移动
		c.recvx++
		// 接收游标归零
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		// buf 数组里的元素个数减 1
		c.qcount--
		// 解锁
		unlock(&c.lock)
		return true, true
	}

	if !block {
		// 非阻塞接收，解锁。selected 返回 false，因为没有接收到值
		unlock(&c.lock)
		return false, false
	}

	// 接下来就是要被阻塞的情况了
	// 构造一个 sudog
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}

	// 待接收数据的地址保存下来
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.selectdone = nil
	mysg.c = c
	gp.param = nil
	// 进入channel 的等待接收队列
	c.recvq.enqueue(mysg)
	// 将当前 goroutine 挂起
	goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv, 3)

	// 被唤醒了，接着从这里继续执行一些扫尾工作
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
}

```

## go channel 生产 消费示例代码



## channel规则

## 应用场景

### 定时任务
- **超时处理**

```go
select {
    case <-time.After(time.Second):
}
```
- **定时任务**

```go
select {
    case <- time.Tick(time.Second):
}
```
### 解耦生产者和消费者
```
可以将生产者和消费者解耦出来，生产者只需要往channel发送数据，而消费者只管从channel中获取数据
```

#### 单生产 单消费
```go

```

#### 单生产 多消费
```go

```

#### 多生产 单消费
```go

```

#### 多生产 多消费
```go

```

### 控制并发数
```go
ch := make(chan int, 5)

for _, foo := range foos {
    go func() {
        ch <- 1
        bar(foo)
        <- ch
    }
}
```
### 锁机制

- 利用channel实现互斥机制。(详见 用channel如何实现分布式锁?)

## 特殊知识点

### 同一个协程里面，对无缓冲channel同时发送和接收数据有什么问题
```
同一个协程里，不能对无缓冲channel同时发送和接收数据，如果这么做会直接报错死锁。

对于一个无缓冲的channel而言，只有不同的协程之间一方发送数据一方接受数据才不会阻塞。channel无缓冲时，发送阻塞直到数据被接收，接收阻塞直到读到数据。
```

### channel和锁的对比
```
并发问题可以用channel解决也可以用Mutex解决，但是它们的擅长解决的问题有一些不同。

channel关注的是并发问题的数据流动，适用于数据在多个协程中流动的场景。

而mutex关注的是是数据不动，某段时间只给一个协程访问数据的权限，适用于数据位置固定的场景。
```

### channel有缓冲和无缓冲在使用上有什么区别？
```
无缓冲：发送和接收需要同步。
有缓冲：不要求发送和接收同步，缓冲满时发送阻塞。
因此 channel 无缓冲时，发送阻塞直到数据被接收，接收阻塞直到读到数据；channel有缓冲时，当缓冲满时发送阻塞，当缓冲空时接收阻塞。
```

### channel是否线程安全

- **channel为什么设计成线程安全?**
```
不同协程通过channel进行通信，本身的使用场景就是多线程，为了保证数据的一致性，必须实现线程安全。
```
- **channel如何实现线程安全的?**
```
channel的底层实现中， hchan结构体中采用Mutex锁来保证数据读写安全。在对循环数组buf中的数据进行入队和出队操作时，必须先获取互斥锁，才能操作channel数据。
```
### 用channel如何实现分布式锁?
```go
package main
import ("sync") 

// Lock
type Lock struct {
    c chan struct{}
}

// NewLock
func NewLock() Lock {
    var l Lock
    l.c = make(chan struct{}, 1)
    l.c <- struct{}{}
    return l
}

// Lock, return lock result
func (l Lock) Lock() bool {
    lockResult := false
    select {
        case <-l.c:
        lockResult = true
        default:
    }
    return lockResult
}

// Unlock 
func (l Lock) Unlock() {
	l.c <- struct{}{}
}

var counter int

func main() {
    var l = NewLock()
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            if !l.Lock() {
                // log error
                println("lock failed")
                return
            }
            counter++
            println("current counter", counter)
            l.Unlock()
        }()
    }
    wg.Wait()
}
```

### 如何限制 goroutine 并发数量
```go
package main

import (  
    "fmt"
    "sync"
    "time"
)

func process(i int, wg *sync.WaitGroup) {  
    fmt.Println("started Goroutine ", i)
    time.Sleep(2 * time.Second)
    fmt.Printf("Goroutine %d ended", i)
    wg.Done()
}

func main() {  
    no := 3
    var wg sync.WaitGroup
    for i := 0; i < no; i++ {
        wg.Add(1)
        go process(i, &wg)
    }
    wg.Wait()
    fmt.Println("All go routines finished executing")
}
```

### go channel实现并发顺序执行
```go
var wg sync.WaitGroup

func channelTest7() {
	ch1 := make(chan struct{}, 1)
	ch2 := make(chan struct{}, 1)
	ch3 := make(chan struct{}, 1)
	ch1 <- struct{}{}
	wg.Add(30)
	start := time.Now().Unix()
	for i := 0; i < 10; i++ {
		go print("猫", ch1, ch2)
	}

	for i := 0; i < 10; i++ {
		go print("狗", ch2, ch3)
	}

	for i := 0; i < 10; i++ {
		go print("老虎", ch3, ch1)
	}

	wg.Wait()
	end := time.Now().Unix()
	fmt.Printf("duration:%d", end-start)
}

func print(g string, inputchan chan struct{}, outchan chan struct{}) {
	// 模拟内部操作耗时
	time.Sleep(1 * time.Second)
	select {
	case <-inputchan:
		fmt.Printf("%s \n", g)
		outchan <- struct{}{}
	}
	wg.Done()
}

```

### go channel实现归并排序
```go
func Merge(ch1 <-chan int, ch2 <-chan int) <-chan int {
 
    out := make(chan int)
 
    go func() {
        // 等上游的数据 （这里有阻塞，和常规的阻塞队列并无不同）
        v1, ok1 := <-ch1
        v2, ok2 := <-ch2
        
        // 取数据
        for ok1 || ok2 {
            if !ok2 || (ok1 && v1 <= v2) {
                // 取到最小值, 就推到 out 中
                out <- v1
                v1, ok1 = <-ch1
            } else {
                out <- v2
                v2, ok2 = <-ch2
            }
        }
        // 显式关闭
        close(out)
    }()
 
    // 开完goroutine后, 主线程继续执行, 不会阻塞
    return out
}
```

