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
	closed   uint32 // channel是否已关闭
	elemtype *_type // channel内元素的类型
	sendx    uint   // 当前写进buf的index
	recvx    uint   // 当前从buf读的index
	recvq    waitq  // 读阻塞的gotoutine
	sendq    waitq  // 写阻塞的goroutine

	lock mutex // 保护channel在多goroutine下的临界区
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
    // 编译时设置了对应的运行时代码 chansend1
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
    
    // 找到一个正在等待这个channel有新的可读内容的阻塞中的goroutine,把消息发给‘ta’
	if sg := c.recvq.dequeue(); sg != nil { 
        // todo
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

	// 既没有goroutine正在等待channel内容,channel也没有多的buf空间时,需要将当前goroutine阻塞,将运行控制权利交给g0,等待有别的goroutine读了这条消息后再继续运行
	gp := getg()
	mysg := acquireSudog() // 获取一个sudoG任务
    // 更新一些任务上下文信息
	mysg.releasetime = 0
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
    // 将sudoG任务加入到channel的sendq队列中,这样该channel的消费方goroutine可以取到当前goroutine的信息,在消费完后更新当前goroutine的调度状态,让当前goroutine有机会继续运行下去
	c.sendq.enqueue(mysg) 
	// ...
    // 移交当前goroutine的运行权利给g0,
    // todo 当前goroutine是否会阻塞在这里直到g0下一次调用当前goroutine??
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)

	// 到这里时说明当前goroutine又被g0唤醒了,此时当前goroutine因往channel里写而阻塞的消息已经被消费者goroutine读取了,当前goroutine继续做该做的事,更新信息以及释放sudoG
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg) // 释放sudoG
	return true
}
```
### send 直接把消息发到阻塞的读goroutine里
```go
// runtime/chan.go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {

	if sg.elem != nil {
        // 直接将内容写到读阻塞的goroutine里
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
    // 更改阻塞的读goroutine的状态,让ta有机会继续运行
	goready(gp, skip+1)
}
```

## channel 读
### 编译时
#### 返回两个变量的情况
```go
// cmd/compile/internal/walk/expr.go
    // x, y = <-c
	// order.stmt made sure x is addressable or blank.
	case ir.OAS2RECV:
		n := n.(*ir.AssignListStmt)
		return walkAssignRecv(init, n)

// cmd/compile/internal/walk/assign.go
func walkAssignRecv(init *ir.Nodes, n *ir.AssignListStmt) ir.Node {
	// ...
    // 编译时设置了对应的运行时代码 chanrecv2
	fn := chanfn("chanrecv2", 2, r.X.Type())
	ok := n.Lhs[1]
	call := mkcall1(fn, types.Types[types.TBOOL], init, r.X, n1)
	return typecheck.Stmt(ir.NewAssignStmt(base.Pos, ok, call))
}
```
####  返回一个变量的情况
```go
// compile/internal/walk/assign.go
    case ir.ORECV:
		// x = <-c; as.Left is x, as.Right.Left is c.
		// order.stmt made sure x is addressable.
		recv := as.Y.(*ir.UnaryExpr)
		recv.X = walkExpr(recv.X, init)

		n1 := typecheck.NodAddr(as.X)
		r := recv.X // the channel
		return mkcall1(chanfn("chanrecv1", 2, r.Type()), nil, init, r, n1)
```
### 运行时
```go
// runtime/chan.go
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // ...
    // channel锁
	lock(&c.lock)

	if c.closed != 0 && c.qcount == 0 { // channel已经关闭,且没有内容了,需要返回给调用方知道这个信息(return true, false)
        //...
		unlock(&c.lock)
		return true, false
	}

	if sg := c.sendq.dequeue(); sg != nil {  // 如果有goroutine阻塞在写消息到channel的阶段,调用recv直接从c.sendq里拿消息出来
		// todo
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

    // 如果是那种带buf的channel,buf里有内容,则从buf里取消息
	if c.qcount > 0 {
		// ...
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

    // 既没有阻塞的goroutine写入方,buf里也没内容可读,需要阻塞当前读取方,直到有goroutine往channel里写内容
	// 注:逻辑与写阻塞类似
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	// ...
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
    // 把当前sudog入队recvq,这样有写入channel的goroutine就可以直接把消息写入
	c.recvq.enqueue(mysg)
	
	atomic.Store8(&gp.parkingOnChan, 1)
    // 移交当前goroutine的运行权利给g0
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// 到这里时说明当前goroutine又被g0唤醒了,此时当前goroutine因读阻塞的消息已经被读到了相应的空间了,当前goroutine继续做该做的事,更新信息以及释放sudoG
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, success
}
```
### recv 直接从阻塞的写goroutine里读
```go
// runtime/chan.go
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 { // 无buf的channel,直接从sender把消息取出来
		// ...
		if ep != nil {
			// 直接把值复制过来
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// 有buf的channel,需要讲究个先来后到,把buf里最前面的消息取了,然后把阻塞的sender消息添加到buf里去

        // 先找到当前buf的读位置
		qp := chanbuf(c, c.recvx)
		// 将buf当前可读位置的内容复制出来
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// 然后复制阻塞的消息到buf里
		typedmemmove(c.elemtype, qp, sg.elem)

        // 更新buf的读位置
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
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
    // 更改阻塞的写goroutine的状态,让ta有机会继续运行
	goready(gp, skip+1)
}

```

## select 中的读 todo