# 11.5 关键字: chan 与 select

[TOC]

Tony Hoare 于 1977 年提出通信顺序进程（CSP）理论。
简单来说，CSP 的模型由并发执行的实体（线程或者进程）所组成，实体之间通过发送消息进行通信，
这里发送消息时使用的就是通道（channel）。也就是我们常说的
『 Don't communicate by sharing memory; share memory by communicating 』。

Go 语言实现了部分 CSP 理论，goroutine 就是 CSP 理论中的并发实体，而 channel 则对应 CSP 中的 channel。
其中的区别在于 CSP 理论中通信是隐式的，而 Go 的通信则是显式的由程序员进行控制，channel 是 Go 提供
的除了 sync 这种基于共享内存的同步原语之外的，基于消息传递的同步原语。

## channel 的本质

### 基本使用

channel 主要有两种：

- buffered channel: `make(chan interface{}, n)`
- unbuffered channel: `make(chan interface{})`

没有使用 make 创建的 channel 无法向其发送数据，相反会造成死锁。这两种 channel 的读写操作都非常简单：

```go
ch := make(chan interface{}, 10)
// 读
v <- ch
// 写
ch <- v
```

他们之间的本质区别在于其内存模型的差异，在垃圾回收器中我们讨论过了这些 Go 内存模型的差异：

- buffered channel: `ch <- v` < `v <- ch`
- buffered channel: `close(ch)` < `v <- ch & v == isZero(v)`
- unbuffered channel: `ch <- v` > `v <- ch`
- unbuffered channel: `len(ch) == C` => `从 channel 中收到第 k 个值` < `k+C 个值得发送完成`

直观上我们很好理解他们之间的差异，
对于 buffered channel 而言，内部有一个缓冲队列，数据会优先进入缓冲队列，而后被消费，即 `ch <- v` < `v <- ch`；
对于 unbuffered channel 而言，内部没有缓冲队列，`v <- ch` 会一直阻塞到 `ch <- v` 执行完毕，因此 `ch <- v` > `v <- ch`

Go 内建了 `close()` 函数来关闭一个 channel，但：

- 关闭一个已关闭的 channel 会导致 panic
- 向关闭的 channel 发送数据会导致 panic
- 向关闭的 channel 读取数据不会导致 panic，但读取的值为 channel 缓存数据的零值，可以通过第二个返回值来检查 channel 是否关闭

select 语句伴随 channel 一起出现，常见的用法是：

```go
select {
case ch <- v:
    // ...
default:
    // ...
}
```

或者：

```go
select {
case v := <- ch:
    // ...
default:
    // ...
}
```

用于处理多个不同类型的 v 的发送与接收，并提供默认处理方式。

### channel 底层结构

实现 channel 的结构并不神秘，本质上就是一个 mutex 锁加上一个环状队列。

```go
   hchan           0   1   2   3
+---------+      +---+---+---+---+
|   buf   | ---> | e | e |   |   |  环状队列
+---------+      +---+---+---+---+
|  sendx  | 发送索引
+---------+
|  recvx  | 接受索引
+---------+     +---+     +---+     +---+
|  recvq  | --> | G | --> |   | --> |   | --> nil 接收队列
+---------+     +---+     +---+     +---+
|  sendq  | --+     +---+     +---+     +---+
+---------+   +---> | G | --> |   | --> |   | --> nil 发送队列
|   lock  |         +---+     +---+     +---+
+---------+
|   ...   |


type hchan struct {
	qcount   uint           // 队列中的所有数据数
	dataqsiz uint           // 环形队列的大小
	buf      unsafe.Pointer // 指向大小为 dataqsiz 的数组
	elemsize uint16
	closed   uint32
	elemtype *_type // 元素类型
	sendx    uint   // 发送索引
	recvx    uint   // 接受索引
	recvq    waitq  // recv 等待列表，即（ <-ch ）
	sendq    waitq  // send 等待列表，即（ ch<- ）

	// lock 保护了 hchan 的所有字段，以及在此 channel 上阻塞的 sudog 的一些字段
	//
	// 当持有此锁时不改变其他 G 的状态（特别的，不 ready 一个 G），因为
	// 它会在栈收缩时发生死锁
	//
	lock mutex
}
type waitq struct { // 等待队列 sudog 双向队列
	first *sudog
	last  *sudog
}
```

其中 recvq 和 sendq 分别是 sudog 的一个链式队列，其元素是一个包含当前包含队 goroutine 及其要在 channel 中发送的数据的一个封装：

```go
   sudog   
+---------+
|    g    | ---> goroutine
+---------+
|   next  | ---> 下一个 g
+---------+
|  	prev  | ---> 上一个 g
+---------+
|   elem  | ---> 发送的元素，可能指向其他 goroutine 的执行栈
+---------+
|   ...   |

type sudog struct {
	// 下面的字段由这个 sudog 阻塞的通道的 hchan.lock 进行保护。
	// shrinkstack 依赖于它服务于 sudog 相关的 channel 操作。

	g *g

	// isSelect 表示 g 正在参与一个 select，因此 g.selectDone 必须以 CAS 的方式来避免唤醒时候的 data race。
	isSelect bool
	next     *sudog
	prev     *sudog
	elem     unsafe.Pointer // 数据元素（可能指向栈）

	// 下面的字段永远不会并发的被访问。对于 channel waitlink 只会被 g 访问
	// 对于 semaphores，所有的字段（包括上面的）只会在持有 semaRoot 锁时被访问

	acquiretime int64
	releasetime int64
	ticket      uint32
	parent      *sudog // semaRoot 二叉树
	waitlink    *sudog // g.waiting 列表或 semaRoot
	waittail    *sudog // semaRoot
	c           *hchan // channel
}
```

### channel 的创生

channel 的创建由编译器完成翻译工作：

```go
make(chan type, n)

=>

runtime.makechan(type, n)
```

而具体的 makechan 实现如下，从 `mallocgc` 可以看出，channel 总是在堆上进行分配，它们会被垃圾回收器进行回收，这也是为什么 channel 不一定总是需要显式进行关闭。

```go
// 将 hchan 的大小对齐
const hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&7)

func makechan(t *chantype, size int) *hchan {
	elem := t.elem
	(...)

	// 检查确认 channel 的容量不会溢出
	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// hchan 在当元素存储在 buf 且不包含指针时，不则不会包含需要被 GC 进行处理的指针，
	// buf 指向了相同的分配区，elemtype 则是恒定的。
	// SudoG 则被他们拥有的线程索引，因此他们无法被搜集。
	var c *hchan
	switch {
	case mem == 0:
		// 队列或元素大小为零
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		(...)
	case elem.ptrdata == 0:
		// 元素不包含指针
		// 在一个调用中分配 hchan 和 buf
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// 元素包含指针
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)

	(...)
	return c
}
```

channel 并不严格支持 `int64` 大小的缓冲，当 `make(chan type, n)` 中 n 为 int64 类型时，
运行时的实现仅仅只是将其强转为 int，提供了对 int 转型是否成功的检查：

```go
func makechan64(t *chantype, size int64) *hchan {
	if int64(int(size)) != size {
		panic(plainError("makechan: size out of range"))
	}

	return makechan(t, int(size))
}
```

所以创建一个 channel 最重要的操作就是 channel 自身以及所需创建的 buf 的大小分配内存。

### 向 channel 发送数据

发送数据完成的是如下的翻译过程：

```go
ch <- v

=>

runtime.chansend1(ch, v)
```

而本质上它会去调用更为通用的 `chansend`：

```go
//go:nosplit
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}
```

注意，到目前为止，我们尚未发现 buffered channel 和 unbuffered channel 之间的区别。

下面我们来关注 chansend 的具体实现的第一个部分：

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// 当向 nil channel 发送数据时，会调用 gopark
	// 而 gopark 会将当前的 goroutine 休眠，并用过第一个参数的 unlockf 来回调唤醒
	// 但此处传递的参数为 nil，因此向 channel 发送数据的 goroutine 和接收数据的 goroutine 都会阻塞，
	// 进而死锁
	if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	(...)
}
```

在这个部分中，我们可以看到，如果一个 channel 为空（比如没有初始化），这时候的发送操作会尝试暂止当前的 goroutine（`gopark`）。
在调度器中，我们了解到 gopark 的第一个参数用于提供唤醒当前 goroutine 的回调函数，但此处为 `nil`。
这时，尝试发送数据的 goroutine 会休眠，而等待接收数据的 goroutine 由于接收不到任何数据，也会休眠，进而使得这两者产生永久性的休眠，
从而产生死锁，并泄露 goroutine。

现在我们来看一切已经准备就绪，开始对 channel 加锁：

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {

	(...)

	lock(&c.lock)

	// 持有锁之前我们已经检查了锁的状态，但这个状态可能在持有锁之前、该检查之后发生变化，
	// 因此还需要再检查一次 channel 的状态
	if c.closed != 0 { // 不允许向已经 close 的 channel 发送数据
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

	// 1. 找到了阻塞在 channel 上的读者，直接发送
	if sg := c.recvq.dequeue(); sg != nil {
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	// 2. 判断 channel 中缓存是否仍然有空间剩余
	if c.qcount < c.dataqsiz {
		// 有空间剩余，存入 buffer
		qp := chanbuf(c, c.sendx)
		(...)
		typedmemmove(c.elemtype, qp, ep) // 将要发送的数据拷贝到 buf 中
		c.sendx++
		if c.sendx == c.dataqsiz { // 如果 sendx 索引越界则设为 0
			c.sendx = 0
		}
		c.qcount++ // 完成存入，记录增加的数据，解锁
		unlock(&c.lock)
		return true
	}
	if !block {
		unlock(&c.lock)
		return false
	}

	(...)
}
```

到目前位置，代码中考虑了当 channel 上直接有 reader 等待，可以直接将数据发送走，并返回（情况 1）；或没有 reader
但还有剩余缓存来存放没有读取的数据（情况 2）。对于直接发送数据的情况，由 `send` 调用完成：

```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	(...)
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf() // unlock(&c.lock)
	gp.param = unsafe.Pointer(sg)
	(...)
	// 复始一个 goroutine，放入调度队列等待被后续调度
	// 第二个参数用于 trace 追踪 ip 寄存器的位置，go runtime 又不希望暴露太多内部的调用，因此记录需要跳过多少 ip
	goready(gp, skip+1)
}
func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	dst := sg.elem
	(...) // 为了确保发送的数据能够被立刻观察到，需要写屏障支持，执行写屏障，保证代码正确性
	memmove(dst, src, t.size) // 直接写入 reader 的执行栈！
}
```

`send` 操作其实是隐含了有 reader 阻塞在 channel 上，换句话说有 reader 已经被暂止，当我们发送完数据后，
应该让该 reader 就绪（让调度器继续开始调度 reader）。

这个 `send` 操作其实是一种优化。原因在于，已经处于等待状态的 goroutine 是没有被执行的，因此用户态代码不会
与当前所发生数据发生任何竞争。我们也更没有必要冗余的将数据写入到缓存，再让 reader 从缓存中进行读取。
因此我们可以看到， `sendDirect` 的调用，本质上是将数据直接写入 reader 的执行栈。

最后我们来看第三种情况，如果既找不到 reader，buf 也已经存满，这时我们就应该阻塞当前的 goroutine 了：

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {

	(...)

	// 1. 找到了阻塞在 channel 上的读者，直接发送
	(...)	

	// 2. 判断 channel 中缓存是否仍然有空间剩余
	(...)

	// 3. 阻塞在 channel 上，等待接收方接收数据
	gp := getg()
	mysg := acquireSudog()
	(...)
	c.sendq.enqueue(mysg)
	goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3) // 将当前的 g 从调度队列移出

	// 因为调度器在停止当前 g 的时候会记录运行现场，当恢复阻塞的发送操作时候，会从此处继续开始执行
	(...)
	gp.waiting = nil
	if gp.param == nil {
		if c.closed == 0 { // 正常唤醒状态，goroutine 应该包含需要传递的参数，但如果没有唤醒时的参数，且 channel 没有被关闭，则为虚假唤醒
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	(...)
	mysg.c = nil // 取消与之前阻塞的 channel 的关联
	releaseSudog(mysg) // 从 sudog 中移除
	return true
}
```

简单总结一下，发送过程包含三个步骤：

1. 持有锁
2. 入队，拷贝要发送的数据
3. 释放锁

其中第二个步骤包含三个子步骤：

1. 找到是否有正在阻塞的 reader，是则直接发送
2. 找到是否有空余的缓存，是则存入
3. 阻塞直到被唤醒

### 从 channel 接收数据

接收数据主要是完成以下翻译工作：

```go
v <- ch

=>

runtime.chanrecv1(ch, v)
```

或者

```go
v, ok <- ch

=>

ok := runtime.chanrecv2(ch, v)
```

他们的本质都是调用 `runtime.chanrecv`：

```go
//go:nosplit
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}
//go:nosplit
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}
```

chansend 的具体实现如下，由于我们已经仔细分析过发送过程了，我们不再详细分拆下面代码的步骤，其处理方式基本一致：

1. 上锁
2. 从缓存中出队，拷贝要接受的数据
3. 解锁

其中第二个步骤包含三个子步骤：

1. 如果 channel 已被关闭，且 channel 没有数据，立刻返回
2. 如果存在正在阻塞的发送方，说明缓存已满，从缓存队头取一个数据，再复始一个阻塞的发送方
3. 否则，检查缓存，如果缓存中仍有数据，则从缓存中读取，读取过程会将队列中的数据拷贝一份到接收方的执行栈中
4. 没有能接受的数据，阻塞当前的接收方 goroutine

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	(...)
	// nil channel，同 send，会导致两个 goroutine 的死锁
	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
		c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
		atomic.Load(&c.closed) == 0 {
		return
	}

	(...)

	lock(&c.lock)

    // 1. channel 已经 close，且 channel 中没有数据，则直接返回
	if c.closed != 0 && c.qcount == 0 {
		(...)
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

	// 2. 找到发送方，直接接收
	if sg := c.sendq.dequeue(); sg != nil {
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	// 3. channel 的 buf 不空
	if c.qcount > 0 {
		// 直接从队列中接收
		qp := chanbuf(c, c.recvx)
		(...)
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	if !block {
		unlock(&c.lock)
		return false, false
	}

	// 4. 没有更多的发送方，阻塞 channel
	gp := getg()
	mysg := acquireSudog()
	(...)
	c.recvq.enqueue(mysg)
	goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3)

	(...)
	// 唤醒
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

接受数据同样包含直接从接收方的执行栈中拷贝要发送的数据，但这种情况当且仅当缓存中数据为空时。那么什么时候
一个 channel 的缓存为空，且会阻塞呢？没错，unbuffered channel 就是如此。
```go
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {
		(...)
		if ep != nil {
			// 直接从对方的栈进行拷贝
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// 从缓存队列拷贝
		qp := chanbuf(c, c.recvx)
		(...)
		// copy data from queue to receiver
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// copy data from sender to queue
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}
```

到目前为止我们终于明白了为什么 unbuffered channel 而言 `v <- ch` happens before `ch <- v` 了，
因为接收方会先从发送方栈重拷贝数据，但这时发送方会阻塞到重新被调度：

### channel 的死亡

关闭 channel 主要是完成以下翻译工作：

```go
close(ch)

=>

runtime.closechan(ch)
```

具体的实现中，首先将 channel 阻塞自身的锁上，而后依次将阻塞在 channel 的 g 添加到一个
gList 中，当所有的 g 均从 channel 上移除时，可释放锁，并唤醒 gList 中的所有 reader 和 writer：

```go
func closechan(c *hchan) {
	if c == nil { // close 一个空的 channel 会 panic
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	if c.closed != 0 { // close 一个已经关闭的的 channel 会 panic
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	(...)
	c.closed = 1

	var glist gList

	// 释放所有的读者
	// 此处的 reader 都是阻塞在 channel 上的，先统一将他们加到一个 gList 上
	for {
		sg := c.recvq.dequeue()
		if sg == nil { // 队列已空
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem) // 清零
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = nil
		(...)
		glist.push(gp)
	}

	// 释放所有的写者 (panic)
	// 写者同理
	for {
		sg := c.sendq.dequeue()
		if sg == nil { // 队列已空
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = nil
		(...)
		glist.push(gp)
	}
	// 就绪所有的 G 时可释放 channel 的锁
	unlock(&c.lock)

	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

当 channel 关闭时，我们必须让所有阻塞的 reader 重新被调度，让所有的 writer 也重新被调度，这时候
的实现先将 goroutine 统一添加到一个列表中（需要锁），然统一的一个一个的 read（不需要锁）。

## select 的本质

### 随机化分支

select 本身会被编译为 `selectgo` 调用。这与普通的多个 if 分支不同。
`selectgo` 则用于随机化每条分支的执行顺序，普通多个 if 分支的执行顺序始终是一致的。

```go
type scase struct {
	c           *hchan         // chan
	elem        unsafe.Pointer // data element
	kind        uint16
	pc          uintptr // race pc (for race detector / msan)
	releasetime int64
}
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
	(...)

	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))

	scases := cas1[:ncases:ncases]
	pollorder := order1[:ncases:ncases]
	lockorder := order1[ncases:][:ncases:ncases]

	// 替换 closed channel
	for i := range scases {
		cas := &scases[i]
		if cas.c == nil && cas.kind != caseDefault {
			*cas = scase{}
		}
	}

	(...)

	(...)
	// 生成随机顺序
	for i := 1; i < ncases; i++ {
		j := fastrandn(uint32(i + 1))
		pollorder[i] = pollorder[j]
		pollorder[j] = uint16(i)
	}

	// 根据 channel 的地址进行堆排序，决定获取锁的顺序
	for i := 0; i < ncases; i++ {
		(...)
	}
	(...)

	// 依次加锁
	sellock(scases, lockorder)

	var (
		gp     *g
		sg     *sudog
		c      *hchan
		k      *scase
		sglist *sudog
		sgnext *sudog
		qp     unsafe.Pointer
		nextp  **sudog
	)

loop:
	// 1 检查是否有正在等待
	var dfli int
	var dfl *scase
	var casi int
	var cas *scase
	var recvOK bool
	for i := 0; i < ncases; i++ {
		casi = int(pollorder[i])
		cas = &scases[casi]
		c = cas.c
		switch cas.kind {
		case caseNil:
			continue
		case caseRecv:
			sg = c.sendq.dequeue()
			if sg != nil {
				goto recv
			}
			if c.qcount > 0 {
				goto bufrecv
			}
			if c.closed != 0 {
				goto rclose
			}
		case caseSend:
			(...)
			if c.closed != 0 {
				goto sclose
			}
			sg = c.recvq.dequeue()
			if sg != nil {
				goto send
			}
			if c.qcount < c.dataqsiz {
				goto bufsend
			}
		case caseDefault:
			dfli = casi
			dfl = cas
		}
	}
	// 存在 default 分支，直接去 retc 执行
	if dfl != nil {
		selunlock(scases, lockorder)
		casi = dfli
		cas = dfl
		goto retc
	}

	// 2 入队所有的 channel
	gp = getg()
	(...)
	nextp = &gp.waiting
	for _, casei := range lockorder {
		casi = int(casei)
		cas = &scases[casi]
		if cas.kind == caseNil {
			continue
		}
		c = cas.c
		sg := acquireSudog()
		sg.g = gp
		sg.isSelect = true
		// No stack splits between assigning elem and enqueuing
		// sg on gp.waiting where copystack can find it.
		sg.elem = cas.elem
		sg.releasetime = 0
		if t0 != 0 {
			sg.releasetime = -1
		}
		sg.c = c
		// 按锁的顺序创建等待链表
		*nextp = sg
		nextp = &sg.waitlink

		switch cas.kind {
		case caseRecv:
			c.recvq.enqueue(sg)

		case caseSend:
			c.sendq.enqueue(sg)
		}
	}

	// 等待被唤醒
	gp.param = nil
	// selparkcommit 根据等待列表依次解锁
	gopark(selparkcommit, nil, waitReasonSelect, traceEvGoBlockSelect, 1)

	// 重新上锁
	sellock(scases, lockorder)

	gp.selectDone = 0
	sg = (*sudog)(gp.param)
	gp.param = nil

	// pass 3 - dequeue from unsuccessful chans
	// otherwise they stack up on quiet channels
	// record the successful case, if any.
	// We singly-linked up the SudoGs in lock order.
	casi = -1
	cas = nil
	sglist = gp.waiting
	// Clear all elem before unlinking from gp.waiting.
	for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink {
		sg1.isSelect = false
		sg1.elem = nil
		sg1.c = nil
	}
	gp.waiting = nil

	for _, casei := range lockorder {
		k = &scases[casei]
		if k.kind == caseNil {
			continue
		}
		if sglist.releasetime > 0 {
			k.releasetime = sglist.releasetime
		}
		if sg == sglist {
			// sg has already been dequeued by the G that woke us up.
			casi = int(casei)
			cas = k
		} else {
			c = k.c
			if k.kind == caseSend {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		releaseSudog(sglist)
		sglist = sgnext
	}

	if cas == nil {
		// We can wake up with gp.param == nil (so cas == nil)
		// when a channel involved in the select has been closed.
		// It is easiest to loop and re-run the operation;
		// we'll see that it's now closed.
		// Maybe some day we can signal the close explicitly,
		// but we'd have to distinguish close-on-reader from close-on-writer.
		// It's easiest not to duplicate the code and just recheck above.
		// We know that something closed, and things never un-close,
		// so we won't block again.
		goto loop
	}

	c = cas.c
	(...)
	if cas.kind == caseRecv {
		recvOK = true
	}
	(...)
	selunlock(scases, lockorder)
	goto retc

bufrecv:
	// 可以从 buf 接收
	(...)
	recvOK = true
	qp = chanbuf(c, c.recvx)
	if cas.elem != nil {
		typedmemmove(c.elemtype, cas.elem, qp)
	}
	typedmemclr(c.elemtype, qp)
	c.recvx++
	if c.recvx == c.dataqsiz {
		c.recvx = 0
	}
	c.qcount--
	selunlock(scases, lockorder)
	goto retc

bufsend:
	// 可以发送到 buf
	(...)
	typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
	c.sendx++
	if c.sendx == c.dataqsiz {
		c.sendx = 0
	}
	c.qcount++
	selunlock(scases, lockorder)
	goto retc

recv:
	// can receive from sleeping sender (sg)
	recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	(...)
	recvOK = true
	goto retc

rclose:
	// read at end of closed channel
	selunlock(scases, lockorder)
	recvOK = false
	if cas.elem != nil {
		typedmemclr(c.elemtype, cas.elem)
	}
	(...)
	goto retc

send:
	// can send to a sleeping receiver (sg)
	(...)
	send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	(...)
	goto retc

retc:
	(...)
	return casi, recvOK

sclose:
	// 向已关闭的 channel 进行发送
	selunlock(scases, lockorder)
	panic(plainError("send on closed channel"))
}
```

### 发送数据的分支

select 的诸多用法其实本质上仍然是 channel 操作，编译器会完成如下翻译工作：

```go
// 编译器会将这段语法：
//
//	select {
//	case c <- v:
//		... foo
//	default:
//		... bar
//	}
//
// 转换为：
//
//	if selectnbsend(c, v) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
	return chansend(c, elem, false, getcallerpc())
}
```

注意，这时 chansend 的第三个参数为 `false`，这与前面的普通 channel 发送操作不同，
说明这时 select 的操作是非阻塞的。

我们现在来关注 chansend 中当 block 为 `false` 的情况：

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {

	(...)

	// 快速路径: 检查不需要加锁时失败的非阻塞操作
	if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
		(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
		return false
	}

	(...)

	lock(&c.lock)

	(...)
}
```

这里的快速路径是一个优化，它发生在持有 channel 锁之前，而且这个检查顺序非常重要。
当 select 语句需要发送数据时，处于非阻塞状态发送数据，如果 channel 处于未初始化状态，

在检查 channel 还未就绪后，在检查 channel 是否关闭。
每个检查都是一个 单字 的读操作 (首先 c.recvq.first 或者 c.qcount（取决于 channel 的类型）
因为即使 channel 在两次观察之间被关闭，关闭的 channel 的状态也不可能从已经「就绪可以发送」
转移到「尚未就绪」。
这就隐含了当 channel 在「既没有被关闭，也没有 ready for send」之间的瞬间。
因此表现为那个时候我们观察到 channel 并报告接收数据不能继续进行。换句话说，这段优化对变量的读取顺序不能调整。

第二个关于 select 的处理则是在当判断完 channel 是否有 buf 可缓存当前的数据后，如果没有读者阻塞在 channel 上
则会立即返回失败：

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {

	(...)

	lock(&c.lock)

	(...)

	// 2. 判断 channel 中缓存是否仍然有空间剩余
	if c.qcount < c.dataqsiz {
		// 有空间剩余，存入 buffer
		(...)
		unlock(&c.lock)
		return true
	}
	if !block {
		unlock(&c.lock)
		return false
	}

	(...)
}
```

因此这也是为什么，我们在没有配合 for 循环使用 select 时，需要对发送失败进行处理，例如：

```go
func main() {
	ch := make(chan interface{})
	x := 1
	select {
	case ch <- x:
		println("send success") // 如果初始化为 buffered channel，则会发送成功
	default:
		println("send failed") // 此时 send failed 会被输出
	}
	return
}
```

如果读者进一步尝试没有 default 的例子：

```go
func main() {
	ch := make(chan interface{})
	x := 1
	select {
	case ch <- x:
		println("send success") // 如果初始化为 buffered channel，则会发送成功
	}
	return
}
```

会发现，此时程序会发生 panic：

```go
$ go run main.go
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
```

似乎与源码中发生的行为并不一致，因为按照调用，当锁被解除后，并没有任何 panic。
这是为什么呢？事实上，通过对程序进行反编译，我们能够观察到，**当 select 语句只有一个 case 时，`select` 关键字
是没有被翻译成 `selectgo` 的。因为只有一个 case 的 `select` 与 `if` 是没有区别的，这也是编译器本身对代码的一个优化，
消除了这种情况下调用 `selectgo` 的性能开销：

```go
// src/cmd/compile/internal/gc/select.go
func walkselectcases(cases *Nodes) []*Node {
	n := cases.Len()
	sellineno := lineno

	// optimization: 没有 case 的情况
	if n == 0 {
		return []*Node{mkcall("block", nil, nil)}
	}

	// optimization: 只有一个 case 的情况
	if n == 1 {
		cas := cases.First()
		setlineno(cas)
		l := cas.Ninit.Slice()
		if cas.Left != nil { // not default:
			(...)

			// if ch == nil { block() }; n;
			a := nod(OIF, nil, nil)

			a.Left = nod(OEQ, ch, nodnil())
			var ln Nodes
			ln.Set(l)
			a.Nbody.Set1(mkcall("block", nil, &ln))
			l = ln.Slice()
			a = typecheck(a, ctxStmt)
			l = append(l, a, n)
		}

		l = append(l, cas.Nbody.Slice()...)
		l = append(l, nod(OBREAK, nil, nil))
		return l
	}.

	(...)
}
```

根据编译器的代码，我们甚至可以看到没有分支的 select 会被编译成 `runtime.block` 的调用：

```go
func block() {
	gopark(nil, nil, waitReasonSelectNoCases, traceEvGoStop, 1) // forever
}
```

即让整个 goroutine 暂止。

### 接收数据的分支


对于接受数据而言：

```go
// 编译器会将这段语法：
//
//	select {
//	case v = <-c:
//		... foo
//	default:
//		... bar
//	}
//
// 转换为：
//
//	if selectnbrecv(&v, c) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
	selected, _ = chanrecv(c, elem, false)
	return
}

// 编译器会将这段语法：
//
//	select {
//	case v, ok = <-c:
//		... foo
//	default:
//		... bar
//	}
//
// 转换为
//
//	if c != nil && selectnbrecv2(&v, &ok, c) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbrecv2(elem unsafe.Pointer, received *bool, c *hchan) (selected bool) {
	// TODO(khr): just return 2 values from this function, now that it is in Go.
	selected, *received = chanrecv(c, elem, false)
	return
}
```

## 总结

channel 的实现是一个典型的环形队列+mutex锁的实现，与 channel 同步出现的 select 更像是一个语法糖，其本质仍然是一个 `chansend` 和 `chanrecv` 的两个通用实现。但为了支持 select 在不同分支上的非阻塞操作，`selectgo` 完成了这一需求。

考虑到整个 channel 操作带锁的成本加高，官方也曾考虑过使用无锁 channel 的设计，但由于年代久远，该改进目前处于搁置状态 [Vyukov, 2014b]。

## 进一步阅读的参考文献

- [Vyukov, 2014a] [Dmitry Vyukov, Go channels on steroids, January 2014](https://docs.google.com/document/d/1yIAYmbvL3JxOKOjuCyon7JhW4cSv1wy5hC0ApeGMV9s/pub)
- [Vyukov, 2014b] [Dmitry Vyukov, runtime: lock-free channels, October 2014](https://github.com/golang/go/issues/8899)
- [Vyukov, 2014c] [Dmitry Vyukov, runtime: chans on steroids, October 2014](https://codereview.appspot.com/12544043)

## 许可

[Go under the hood](https://github.com/changkun/go-under-the-hood) | CC-BY-NC-ND 4.0 & MIT &copy; [changkun](https://changkun.de)