## 版本
+ Github:golang/go
+ 分支:release-branch.go.1.17

## 创建channel
### 编译时的遇到 make chan
```go
// compile/internal/walk/expr.go
func walkExpr1(n ir.Node, init *ir.Nodes) ir.Node {
	switch n.Op() {
        // ...
        case ir.OMAKECHAN:
		n := n.(*ir.MakeExpr)
		return walkMakeChan(n, init)
        // ...
    }
}

// compile/internal/walk/builtin.go
func walkMakeChan(n *ir.MakeExpr, init *ir.Nodes) ir.Node {

	if size.Type().IsKind(types.TIDEAL) || size.Type().Size() <= types.Types[types.TUINT].Size() {
		fnname = "makechan" // 对应的在运行时调用的代码是“makechan”
		argtype = types.Types[types.TINT]
	}

	return mkcall1(chanfn(fnname, 1, n.Type()), n.Type(), init, reflectdata.TypePtr(n.Type()), typecheck.Conv(size, argtype))
}
```

### 运行时makechan
#### chan header结构
```go
// runtime/chan.go
type hchan struct {
	qcount   uint           // 当前channel里有多少个元素
	dataqsiz uint           // channel的buf长度
	buf      unsafe.Pointer // 存放channel里的消息的地方
	elemsize uint16 // channel内元素的大小
	closed   uint32
	elemtype *_type // channel内元素的类型
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	lock mutex // 保护channel能在多goroutine下的临界区
}
```
#### makechan
```go
// runtime/chan.go
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	mem, overflow := math.MulUintptr(elem.size, uintptr(size)) //根据channel的buf值大小算出存消息的空间大小
	// ...
	var c *hchan
	switch {
	case mem == 0: // 无buffer值的情况
		// 分配一个hchan大小的空间就行
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// 无buffer值也同样需要给对buf做个处理
		c.buf = c.raceaddr() 
	case elem.ptrdata == 0:
		// todo
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true) //有buffer值时需要分配空间给buffer
	}
    
	c.elemsize = uint16(elem.size) // 元素大小
	c.elemtype = elem // 元素类型
	c.dataqsiz = uint(size) // buffer大小
	return c
}

// 没有指定channel大小时是个阻塞channel,没有申请存消息的空间
func (c *hchan) raceaddr() unsafe.Pointer {
	return unsafe.Pointer(&c.buf)
}
```

## channel 写
### 编译时
```go
// compile/internal/walk/expr.go
func walkSend(n *ir.SendStmt, init *ir.Nodes) ir.Node {
	n1 := n.Value
	n1 = typecheck.AssignConv(n1, n.Chan.Type().Elem(), "chan send")
	n1 = walkExpr(n1, init)
	n1 = typecheck.NodAddr(n1)
    // 编译时把对应的运行时代吗 chansend1 设置好了
	return mkcall1(chanfn("chansend1", 2, n.Chan.Type()), nil, init, n.Chan, n1)
}
```

### 运行时
```go
// runtime/chan.go
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {

	lock(&c.lock) // 锁住channel,防止竞态

	if c.closed != 0 { // channel已经关闭了就直接panic
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}
    
    // 找到一个正在等待这个channel有新的可读内容的goroutine,把消息发给‘ta’
	if sg := c.recvq.dequeue(); sg != nil { // todo
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

    // 找不到正在读的goroutine时,需要把消息先放到buf里(没有指定buf时这里直接会跳过)
	if c.qcount < c.dataqsiz { // buf还没满时

		qp := chanbuf(c, c.sendx)
		typedmemmove(c.elemtype, qp, ep) // 将消息复制到buf的指定位置
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
        // 解锁并返回
		unlock(&c.lock) 
		return true
	}

	// 既没有goroutine正在等待channel内容,channel也没有多的buf空间时,需要将当前goroutine阻塞,等待有别的goroutine读了这条消息后再继续运行
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
}
```