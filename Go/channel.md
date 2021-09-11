## 版本
+ Github:golang/go
+ 分支:release-branch.go.1.17

## 编译时的遇到 make chan
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

## 运行时makechan
### chan header结构
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

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```
### makechan
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
		// 无buffer值也同样需要给buf分配一个特殊的空间
		c.buf = c.raceaddr() // todo
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
	lockInit(&c.lock, lockRankHchan) //todo

	return c
}

// raceaddr 
// lockInit
```
