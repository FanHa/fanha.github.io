## 版本
+ Github:golang/go
+ 分支:release-branch.go.1.17

## 序
go 编译器将代码经过分析后生成了可以代表代码的AST(或加工后的IR树),函数对应树中的一个节点,但函数内的变量需要根据实际情形决定是存放在堆中还是存放在栈中(仅本地使用的一般放在栈中,需要共享使用的要`escape`到堆中)

## escape 入口
```go
// cmd/compile/internal/gc/main.go
func Main(archInit func(*ssagen.ArchInfo)) {
	// ...
    // escape 分析
    escape.Funcs(typecheck.Target.Decls)
    // ...
}
```

```go
// cmd/compile/internal/escape/escape.go
// 如函数名,对 ‘所有’ func节点进行逃逸分析, Batch 是注入的一系列回调
func Funcs(all []ir.Node) {
	ir.VisitFuncsBottomUp(all, Batch)
}
```

```go
// cmd/compile/internal/ir/scc.go
func VisitFuncsBottomUp(list []Node, analyze func(list []*Func, recursive bool)) {
	var v bottomUpVisitor
	v.analyze = analyze // 注册要分析的回调方法
	v.nodeID = make(map[*Func]uint32) // 存放已经分析过的Func的 nodeId
	for _, n := range list { // 遍历所有节点
		if n.Op() == ODCLFUNC { // 只对声明的Func进行escape分析
			n := n.(*Func)
			if !n.IsHiddenClosure() {
				v.visit(n) //开工
			}
		}
	}
}
```
## 数据结构
### Func 节点结构
```go
// cmd/compile/internal/ir/func.go
type Func struct {
	miniNode
	Body Nodes 
	Iota int64

	Nname    *Name        // ONAME node
	OClosure *ClosureExpr // OCLOSURE node

	Shortname *types.Sym

	// Extra entry code for the function. For example, allocate and initialize
	// memory for escaping parameters.
	Enter Nodes
	Exit  Nodes

	// ONAME nodes for all params/locals for this func/closure, does NOT
	// include closurevars until transforming closures during walk.
	// Names must be listed PPARAMs, PPARAMOUTs, then PAUTOs,
	// with PPARAMs and PPARAMOUTs in order corresponding to the function signature.
	// However, as anonymous or blank PPARAMs are not actually declared,
	// they are omitted from Dcl.
	// Anonymous and blank PPARAMOUTs are declared as ~rNN and ~bNN Names, respectively.
	Dcl []*Name

	// ClosureVars lists the free variables that are used within a
	// function literal, but formally declared in an enclosing
	// function. The variables in this slice are the closure function's
	// own copy of the variables, which are used within its function
	// body. They will also each have IsClosureVar set, and will have
	// Byval set if they're captured by value.
	ClosureVars []*Name

	// Enclosed functions that need to be compiled.
	// Populated during walk.
	Closures []*Func

	// Parents records the parent scope of each scope within a
	// function. The root scope (0) has no parent, so the i'th
	// scope's parent is stored at Parents[i-1].
	Parents []ScopeID

	// Marks records scope boundary changes.
	Marks []Mark

	FieldTrack map[*obj.LSym]struct{}
	DebugInfo  interface{}
	LSym       *obj.LSym // Linker object in this function's native ABI (Func.ABI)

	Inl *Inline

	// Closgen tracks how many closures have been generated within
	// this function. Used by closurename for creating unique
	// function names.
	Closgen int32

	Label int32 // largest auto-generated label in this function

	Endlineno src.XPos
	WBPos     src.XPos // position of first write barrier; see SetWBPos

	Pragma PragmaFlag // go:xxx function annotations

	flags bitset16

	// ABI is a function's "definition" ABI. This is the ABI that
	// this function's generated code is expecting to be called by.
	//
	// For most functions, this will be obj.ABIInternal. It may be
	// a different ABI for functions defined in assembly or ABI wrappers.
	//
	// This is included in the export data and tracked across packages.
	ABI obj.ABI
	// ABIRefs is the set of ABIs by which this function is referenced.
	// For ABIs other than this function's definition ABI, the
	// compiler generates ABI wrapper functions. This is only tracked
	// within a package.
	ABIRefs obj.ABISet

	NumDefers  int32 // number of defer calls in the function
	NumReturns int32 // number of explicit returns in the function

	// nwbrCalls records the LSyms of functions called by this
	// function for go:nowritebarrierrec analysis. Only filled in
	// if nowritebarrierrecCheck != nil.
	NWBRCalls *[]SymAndPos
}
```

### Visitor 结构
```go
// cmd/compile/internal/ir/scc.go
type bottomUpVisitor struct {
	analyze  func([]*Func, bool) // 注入的分析方法组
	visitgen uint32 // 自增visitID
	nodeID   map[*Func]uint32 // 纪录一个函数是否已经被visit过
	stack    []*Func // todo
}
```
## `escape`分析过程
```go
// cmd/compile/internal/ir/scc.go
func (v *bottomUpVisitor) visit(n *Func) uint32 {
	// 判断当前Func是否已被visit过
	if id := v.nodeID[n]; id > 0 {
		return id
	}

	// 当前Func节点没有被分析过,递增生成新的NodeId,
	v.visitgen++
	id := v.visitgen
	v.nodeID[n] = id
	// todo 这个min是啥?
	v.visitgen++
	min := v.visitgen
	// 所有遍历去重后的节点线放到stack上,后面再统一分析
	v.stack = append(v.stack, n)

	do := func(defn Node) {
		if defn != nil {
			// 递归visit
			if m := v.visit(defn.(*Func)); m < min {
				min = m
			}
		}
	}

	// 深度优先遍历以当前节点为root的树
	Visit(n, func(n Node) {
		switch n.Op() {
		case ONAME:
			if n := n.(*Name); n.Class == PFUNC {
				do(n.Defn)
			}
		case ODOTMETH, OCALLPART, OMETHEXPR:
			if fn := MethodExprName(n); fn != nil {
				do(fn.Defn)
			}
		case OCLOSURE:
			n := n.(*ClosureExpr)
			do(n.Func)
		}
	})

	if (min == id || min == id+1) && !n.IsHiddenClosure() {
		// todo 精简stack(估计是存在递归调用啥的)
		recursive := min == id

		var i int
		for i = len(v.stack) - 1; i >= 0; i-- {
			x := v.stack[i]
			if x == n {
				break
			}
			v.nodeID[x] = ^uint32(0)
		}
		v.nodeID[n] = ^uint32(0)
		block := v.stack[i:]
		// 遍历精简后的stack,调用注入的analyze方法
		v.stack = v.stack[:i]
		v.analyze(block, recursive)
	}

	return min
}
```

## analyze
### 结构
```go
// cmd/compile/internal/escape/escape.go
type batch struct {
	allLocs  []*location
	closures []closure

	heapLoc  location
	blankLoc location
}
```
### Batch方法
v.analyze方法是前面注入的 `Batch`方法
```go
// cmd/compile/internal/escape/escape.go
func Batch(fns []*ir.Func, recursive bool) {
	// ...

	var b batch
	b.heapLoc.escapes = true

	// 初始化所有Func
	for _, fn := range fns {
		// #ref initFunc
		b.initFunc(fn)
	}
	for _, fn := range fns {
		if !fn.IsHiddenClosure() {
			// 解析Func里的各个变量的关系 #ref walkFunc
			b.walkFunc(fn)
		}
	}

	// 普通的函数和方法已经解析完了,开始解析闭包
	for _, closure := range b.closures {
		b.flowClosure(closure.k, closure.clo)
	}
	b.closures = nil

	// 遍历所有变量的location结构
	for _, loc := range b.allLocs {
		// 找出需要放在堆上的变量,目前主要是“too large for stack“ 和 “non-constant size”
		if why := HeapAllocReason(loc.n); why != "" {
			// 这里b.heapHole.addr(loc.n, why)相当于新建了一个隐藏变量x,(x.derefs 值为-1)
			// 调用flow相当于x = &loc,增加了一条x指向loc地址的边
			b.flow(b.heapHole().addr(loc.n, why), loc) 
		}
	}

	// 遍历所有location,根据有向图,给需要逃逸的变量打上escapes标记
	b.walkAll()
	// 收尾工作
	b.finish(fns)
}
```

#### initFunc 初始化Func节点
```go
// src/cmd/compile/internal/escape/escape.go
func (b *batch) initFunc(fn *ir.Func) {
	// 创建一个包裹当前Func节点 和 batch信息的 escape结构 #ref with
	e := b.with(fn)
	// ...

	// 遍历当前Func的所有变量声明
	for _, n := range fn.Dcl {
		if n.Op() == ir.ONAME {
			// 为每一个变量创建一个location,新location初始都是放在了batch的Alllocs里 #ref newLoc
			e.newLoc(n, false)
		}
	}

	// 当前Func的返回属性的处理,返回属性可能是前面Dcl中的变量,所以使用oldLoc #ref oldLoc
	for i, f := range fn.Type().Results().FieldSlice() {
		e.oldLoc(f.Nname.(*ir.Name)).resultIndex = 1 + i // resultIndex!=0表明该location是Func的返回属性, 数值表示该返回属性的位置顺序
	}
}

// escape结构包裹一个func节点和batch信息
func (b *batch) with(fn *ir.Func) *escape {
	return &escape{
		batch:     b,
		curfn:     fn,
		loopDepth: 1,
	}
}

// 给这个变量信息分配存储位置
func (e *escape) newLoc(n ir.Node, transient bool) *location {
	// ...
	// 初始化一个location结构
	loc := &location{
		n:         n,
		curfn:     e.curfn,
		loopDepth: e.loopDepth,
		transient: transient,
	}
	// 默认都保存在allLocs里
	e.allLocs = append(e.allLocs, loc)
	if n != nil {
		if n.Op() == ir.ONAME {
			n := n.(*ir.Name)
			// 将变量节点的Opt属性设置为刚生成的location
			n.Opt = loc
		}
	}
	return loc
}

func (b *batch) oldLoc(n *ir.Name) *location {
	return n.Canonical().Opt.(*location) // 通过变量名找到已有location
}
```

#### walkFunc 解析Func节点,形成一张变量引用图
解析Func节点
```go
// src/cmd/compile/internal/escape/escape.go
func (b *batch) walkFunc(fn *ir.Func) {
	e := b.with(fn)
	fn.SetEsc(escFuncStarted) // 设置Func的Esc状态为escFuncStarted
	// ...
	// 解析节点的body,body是一个节点数组,包含Func函数体内的所有节点
	e.block(fn.Body)

	// ...
}

func (e *escape) block(l ir.Nodes) {
	old := e.loopDepth
	e.stmts(l)
	e.loopDepth = old
}

func (e *escape) stmts(l ir.Nodes) {
	// 遍历Func body里的每一个节点并解析
	for _, n := range l {
		e.stmt(n)
	}
}

func (e *escape) stmt(n ir.Node) {
	// ...
	e.stmts(n.Init())

	switch n.Op() {
		// ...
	case ir.ODCL:
		n := n.(*ir.Decl)
		if !ir.IsBlank(n.X) {
			e.dcl(n.X) //解析声明
		}
	case ir.OIF:
		n := n.(*ir.IfStmt)
		e.discard(n.Cond)
		e.block(n.Body) // 递归解析if语句的body
		e.block(n.Else) // 递归解析else语句的body

	case ir.OFOR, ir.OFORUNTIL:
		n := n.(*ir.ForStmt)
		e.loopDepth++
		e.discard(n.Cond)
		e.stmt(n.Post) // 解析for语句的post区域内容
		e.block(n.Body) // 递归解析for语句的body
		e.loopDepth--

	case ir.ORANGE:
		// for Key, Value = range X { Body }
		n := n.(*ir.RangeStmt)

		// 生成一个新的临时location(tmp) 指向 要range解构 的变量 
		tmp := e.newLoc(nil, false)
		e.expr(tmp.asHole(), n.X)

		e.loopDepth++
		ks := e.addrs([]ir.Node{n.Key, n.Value})
		if n.X.Type().IsArray() { // 要range解构的变量是个数组时
			e.flow(ks[1].note(n, "range"), tmp) // 为key,value 和 临时location(tmp)间生成普通的边(derefs = 0)
		} else {
			e.flow(ks[1].deref(n, "range-deref"), tmp) // 为key, value 和 临时location(tmp)生成引用边(derefs = -1)
		}
		e.reassigned(ks, n) // todo reassigned ??

		e.block(n.Body) // 递归解析range的body内容
		e.loopDepth--

	case ir.OSWITCH: // switch语法
		n := n.(*ir.SwitchStmt)
		// ...
		for _, cas := range n.Cases {
			e.discards(cas.List)
			e.block(cas.Body) // 递归解析所有case的body内容
		}

	case ir.OSELECT: // select 语法
		n := n.(*ir.SelectStmt)
		for _, cas := range n.Cases {
			e.stmt(cas.Comm)
			e.block(cas.Body) // 递归解析所有case的body内容
		}
	case ir.ORECV: // todo ??
		// TODO(mdempsky): Consider e.discard(n.Left).
		n := n.(*ir.UnaryExpr)
		e.exprSkipInit(e.discardHole(), n) // already visited n.Ninit
	case ir.OSEND: // send 语法
		n := n.(*ir.SendStmt)
		e.discard(n.Chan)
		e.assignHeap(n.Value, "send", n) // 要发送到chan里的变量自然是要存到堆上的 #ref assignHeap

	/** 各种类型赋值语句 dst := src 都是通过assignList建立dst 与 src 的有向边 #ref assignList **/
	case ir.OAS:
		n := n.(*ir.AssignStmt)
		// 解析赋值语句,生成有向边 #ref assignList
		e.assignList([]ir.Node{n.X}, []ir.Node{n.Y}, "assign", n)
	case ir.OASOP:
		n := n.(*ir.AssignOpStmt)
		// TODO(mdempsky): Worry about OLSH/ORSH?
		e.assignList([]ir.Node{n.X}, []ir.Node{n.Y}, "assign", n)
	case ir.OAS2:
		n := n.(*ir.AssignListStmt)
		e.assignList(n.Lhs, n.Rhs, "assign-pair", n)

	case ir.OAS2DOTTYPE: // v, ok = x.(type)
		n := n.(*ir.AssignListStmt)
		e.assignList(n.Lhs, n.Rhs, "assign-pair-dot-type", n)
	case ir.OAS2MAPR: // v, ok = m[k]
		n := n.(*ir.AssignListStmt)
		e.assignList(n.Lhs, n.Rhs, "assign-pair-mapr", n)
	case ir.OAS2RECV, ir.OSELRECV2: // v, ok = <-ch
		n := n.(*ir.AssignListStmt)
		e.assignList(n.Lhs, n.Rhs, "assign-pair-receive", n)

	case ir.OAS2FUNC: // 语句是个函数(方法)调用时
		n := n.(*ir.AssignListStmt)
		e.stmts(n.Rhs[0].Init()) // 递归解析函数内部的escape情况
		ks := e.addrs(n.Lhs)
		e.call(ks, n.Rhs[0], nil) // 把函数作为一个整体 和 当前上下文的escape分析
		e.reassigned(ks, n)
	case ir.ORETURN: // 函数的return
		n := n.(*ir.ReturnStmt)
		results := e.curfn.Type().Results().FieldSlice()
		dsts := make([]ir.Node, len(results))
		for i, res := range results { // 为return的所有属性占一个位置
			dsts[i] = res.Nname.(*ir.Name)
		}
		e.assignList(dsts, n.Results, "return", n)
	case ir.OCALLFUNC, ir.OCALLMETH, ir.OCALLINTER, ir.OCLOSE, ir.OCOPY, ir.ODELETE, ir.OPANIC, ir.OPRINT, ir.OPRINTN, ir.ORECOVER:
		e.call(nil, n, nil)
	case ir.OGO, ir.ODEFER:
		n := n.(*ir.GoDeferStmt)
		e.stmts(n.Call.Init()) // go 或 defer 函数内部的内容递归到内部去解析
		e.call(nil, n.Call, n) // 把函数作为一个整体 和 当前上下文的escape的分析
	//...
}
```
##### dcl 变量声明就是新建一个包裹这个变量的hole
```go
func (e *escape) dcl(n *ir.Name) hole {
	loc := e.oldLoc(n) // 变量声明的location早在InitFunc阶段就已经生成好了,所以这里用oldLoc取到这个location的地址
	loc.loopDepth = e.loopDepth //todo e的loopDepth 和 loc 的loopDepth的作用
	return loc.asHole()
}
```

##### assignList 解析赋值语句生成有向边
```go
func (e *escape) assignList(dsts, srcs []ir.Node, why string, where ir.Node) {
	ks := e.addrs(dsts)
	for i, k := range ks { 
		var src ir.Node
		if i < len(srcs) {
			src = srcs[i]
		}

		// 因为赋值后面src可能需要解析展开,这里封装一层,实际还是生成dst 到 src的有向边
		e.expr(k.note(where, why), src)
	}

	e.reassigned(ks, where)
}

// 创建一个srcNode(n) 到 dstHole(k) 的有向边
func (e *escape) expr(k hole, n ir.Node) {
	if n == nil {
		return
	}
	e.stmts(n.Init()) // n是个表达式的话,需要先解析
	// #ref exprSkipInit
	e.exprSkipInit(k, n)
}

func (e *escape) exprSkipInit(k hole, n ir.Node) {
	// ...

	switch n.Op() { //一些常见的赋值语句语法的有向边解析
	// ... 
	// src 是个普通的变量时,调用flow方法新建一条普通边,derefs值为0
	case ir.ONAME:
		n := n.(*ir.Name)
		// ...
		e.flow(k, e.oldLoc(n))

	// ... 
	case ir.OADDR: 	// x:= &y src是地址引用的情况
		n := n.(*ir.AddrExpr)
		// addr(n, “address-of”)新建了一个derefs值为 -1 的hole,这里就相当于新建了一条 derefs值为-1的边
		e.expr(k.addr(n, "address-of"), n.X) // "address-of"
	case ir.ODEREF: 	// x := *y 指针deref
		n := n.(*ir.StarExpr)
		// deref(n, "indirection") 新建了一个derefs值为1 的hole,这里相当于新建了一条 derefs 值为 1 的边
		e.expr(k.deref(n, "indirection"), n.X) // "indirection"
	case ir.ODOT, ir.ODOTMETH, ir.ODOTINTER: // 赋值的是个对象中的属性时
		n := n.(*ir.SelectorExpr)
		// note(n, "dot") 新建了一个derefs值为0 的hole,这里相当于建了一条derefs 值为0 的边
		e.expr(k.note(n, "dot"), n.X)
	case ir.ODOTPTR: // 对象指针同前面的`ODEREF`指针
		n := n.(*ir.SelectorExpr)
		e.expr(k.deref(n, "dot of pointer"), n.X) // "dot of pointer"
	// ...
	// 方法,函数或系统调用时,需要解析表达式的内容 
	case ir.OCALLMETH, ir.OCALLFUNC, ir.OCALLINTER, ir.OLEN, ir.OCAP, ir.OCOMPLEX, ir.OREAL, ir.OIMAG, ir.OAPPEND, ir.OCOPY, ir.OUNSAFEADD, ir.OUNSAFESLICE:
		e.call([]hole{k}, n, nil) // 新建一条dst 到 函数节点location的边(call会解析函数的输出) #ref call

	/****** new, make 等等语法需要调用spill(这个方法内会调用 addr(n,“xxx“))生成derefs为-1的hole,然后用这个hole生成与src节点的有向边 *****/
	case ir.ONEW: // new
		n := n.(*ir.UnaryExpr)
		e.spill(k, n) 

	case ir.OMAKESLICE: //make []
		n := n.(*ir.MakeExpr)
		e.spill(k, n)
		e.discard(n.Len)
		e.discard(n.Cap)
	case ir.OMAKEMAP: //make map
		n := n.(*ir.MakeExpr)
		e.spill(k, n)
		e.discard(n.Len)
	case ir.OCALLPART: // 方法(非调用)
		// Flow the receiver argument to both the closure and
		// to the receiver parameter.

		n := n.(*ir.SelectorExpr)
		closureK := e.spill(k, n)

		m := n.Selection

		// 保守的为该方法所有的返回属性创建一个heapHole
		var ks []hole
		for i := m.Type.NumResults(); i > 0; i-- {
			ks = append(ks, e.heapHole())
		}
		name, _ := m.Nname.(*ir.Name) // nName是该方法的参数列表
		paramK := e.tagHole(ks, name, m.Type.Recv()) // todo ??

		e.expr(e.teeHole(paramK, closureK), n.X)

	// ...
	/****   ***************************** *****/
	// 闭包
	case ir.OCLOSURE:
		n := n.(*ir.ClosureExpr)
		k = e.spill(k, n) // 先为闭包本身创建一个location ,并通过spill创建一个变量到该location 的 derefs值为-1的有向边
		e.closures = append(e.closures, closure{k, n})

		if fn := n.Func; fn.IsHiddenClosure() {
			// 遍历闭包中用到的闭包外变量
			for _, cv := range fn.ClosureVars { 
				if loc := e.oldLoc(cv); !loc.captured {
					// 标记这个变量被闭包使用
					loc.captured = true

					// Ignore reassignments to the variable in straightline code
					// preceding the first capture by a closure.
					if loc.loopDepth == e.loopDepth {
						loc.reassigned = false
					}
				}
			}
			// ...
			// 递归调用walkFunc解析闭包内的escape
			e.walkFunc(fn)
		}
		// ...
}

// spill 主动申请一个新location, 然后调用addr(n, "spill"),生成一个derefs为-1的hole,用这个hole生成一个与src节点derefs为-1的有向边
func (e *escape) spill(k hole, n ir.Node) hole {
	loc := e.newLoc(n, true)
	e.flow(k.addr(n, "spill"), loc)
	return loc.asHole()
}
```

##### call 把函数当作一个整体 作escape分析
```go
func (e *escape) call(ks []hole, call, where ir.Node) {
	topLevelDefer := where != nil && where.Op() == ir.ODEFER && e.loopDepth == 1
	if topLevelDefer {
		// force stack allocation of defer record, unless
		// open-coded defers are used (see ssa.go)
		where.SetEsc(ir.EscNever)
	}

	// 对函数的入参的有向图处理
	argument := func(k hole, arg ir.Node) {
		if topLevelDefer {
			// todo 函数的defer处理
			k = e.later(k)
		} else if where != nil {
			k = e.heapHole()
		}

		e.expr(k.note(call, "call parameter"), arg)
	}

	switch call.Op() {
	// ...

	case ir.OCALLFUNC, ir.OCALLMETH, ir.OCALLINTER: // 普通的函数,方法调用
		call := call.(*ir.CallExpr)
		typecheck.FixVariadicCall(call)

		// Pick out the function callee, if statically known.
		var fn *ir.Name
		// ...

		fntype := call.X.Type()
		if fn != nil {
			fntype = fn.Type()
		}

		if ks != nil && fn != nil && e.inMutualBatch(fn) {
			for i, result := range fn.Type().Results().FieldSlice() { // 取出函数的返回列表
				e.expr(ks[i], ir.AsNode(result.Nname)) // 在这里,函数的返回 于 函数的调用方建立了有向边的联系
			}
		}

		if r := fntype.Recv(); r != nil { // 方法有一个receiver时,需要对参数做处理,相当于把receiver本身作为函数的一个参数
			argument(e.tagHole(ks, fn, r), call.X.(*ir.SelectorExpr).X)
		} else {
			// Evaluate callee function expression.
			argument(e.discardHole(), call.X)
		}

		args := call.Args // 函数的参数与调用方的 有向边分析
		for i, param := range fntype.Params().FieldSlice() {
			argument(e.tagHole(ks, fn, param), args[i])
		}
	/** 一些系统自带函数的输入输出的有向边解析**/
	case ir.OAPPEND:
		// ...

	case ir.OCOPY:
		// ...

	case ir.OPANIC:
		// ...

	case ir.OCOMPLEX:
		// ...
	case ir.ODELETE, ir.OPRINT, ir.OPRINTN, ir.ORECOVER:
		// ...
	case ir.OLEN, ir.OCAP, ir.OREAL, ir.OIMAG, ir.OCLOSE:
		// ...

	case ir.OUNSAFEADD, ir.OUNSAFESLICE:
		// ...
	}
}
```

##### assignHeap 相当于为要发送到chan 的变量创建一个临时变量,这个临时变量的位置放在堆上,再计算变量与临时变量的有向边数值
```go
func (e *escape) assignHeap(src ir.Node, why string, where ir.Node) {
	e.expr(e.heapHole().note(where, why), src)
}
```

#### flowClosure 解析闭包
```go
func (b *batch) flowClosure(k hole, clo *ir.ClosureExpr) {
	for _, cv := range clo.Func.ClosureVars { // 遍历闭包所有用到的外部变量
		n := cv.Canonical()
		loc := b.oldLoc(cv) //找到外部变量的原始location

		// todo ??
		k := k
		if !cv.Byval() {
			k = k.addr(cv, "reference")
		}
		// 建立一条由闭包内到闭包外的变量location的有向边
		b.flow(k.note(cv, "captured by a closure"), loc)
	}
}
```

#### flow 生成一条边,根据边的derefs值决定src要不要escape
```go
// src/cmd/compile/internal/escape/escape.go
func (b *batch) flow(k hole, src *location) {
	if k.addrtaken {
		src.addrtaken = true
	}

	dst := k.dst
	if dst == &b.blankLoc {
		return
	}
	if dst == src && k.derefs >= 0 { // dst = dst, dst = *dst, ...
		return
	}
	if dst.escapes && k.derefs < 0 { // dst = &src 出现了dst的escapes值为true且derefs值为-1的情况,直接将src的escapes值也置为true(根据heap上的数据不能指向stack上的数据的原则)
		src.escapes = true
		return
	}

	// 加一条dst 到 src 的有向边,便于其他用到了dst的地方掂量自己需不需要escape
	dst.edges = append(dst.edges, edge{src: src, derefs: k.derefs, notes: k.notes})
}
```

#### walkAll 遍历所有location,标记所有需要escape的location
```go
// src/cmd/compile/internal/escape/escape.go
func (b *batch) walkAll() {
	// 将所有location结构存入一个队列‘todo’中
	todo := make([]*location, 0, len(b.allLocs)+1)

	// 定义了一个enqueue方法,把一个location入队 todo 队列中
	enqueue := func(loc *location) {
		if !loc.queued {
			todo = append(todo, loc)
			loc.queued = true
		}
	}

	for _, loc := range b.allLocs {
		enqueue(loc)
	}
	enqueue(&b.heapLoc)

	var walkgen uint32
	for len(todo) > 0 {
		// 将队列末尾的‘location’节点作为‘root’,
		root := todo[len(todo)-1]
		// todo列表中去除掉将要深入探究的root节点
		todo = todo[:len(todo)-1]
		root.queued = false

		walkgen++
		// #ref walkOne 深入root节点,同时传入了enqueue方法,当root引用的一个location 需要escape时,调用enqueue将该location入队todo中,下一个循环以该location为根,深入递归里面的边喝需要escape的location
		b.walkOne(root, walkgen, enqueue)
	}
}

// 以一个location为root,遍历该location的边,标记需要escape的location并通过enqueue将该location入队需要walk的节点列表中
func (b *batch) walkOne(root *location, walkgen uint32, enqueue func(*location)) {

	root.walkgen = walkgen
	root.derefs = 0 // (以root为初始点设为0)
	root.dst = nil

	todo := []*location{root} // LIFO queue
	for len(todo) > 0 {
		// 这里相当于pop出todo列表里的最后一个元素
		l := todo[len(todo)-1]
		todo = todo[:len(todo)-1]

		// 当前location的derefs属性
		derefs := l.derefs

		// 初始节点的derefs == 0 ,第一遍遍历时,只有root节点,所以addressOf 为 false,
		// 后续的遍历当出现derefs < 0 时表明该location被初始节点引用
		addressOf := derefs < 0
		if addressOf {·
			// For a flow path like "root = &l; l = x",
			// l's address flows to root, but x's does
			// not. We recognize this by lower bounding
			// derefs at 0.
			derefs = 0

			// If l's address flows to a non-transient
			// location, then l can't be transiently
			// allocated.
			if !root.transient && l.transient {
				l.transient = false
				enqueue(l)
			}
		}

		// 判断root是否在l的生命周期外依然存在
		// todo outlives
		if b.outlives(root, l) {
			// l's value flows to root. If l is a function
			// parameter and root is the heap or a
			// corresponding result parameter, then record
			// that value flow for tagging the function
			// later.
			if l.isName(ir.PPARAM) {
				if (logopt.Enabled() || base.Flag.LowerM >= 2) && !l.escapes {
					if base.Flag.LowerM >= 2 {
						fmt.Printf("%s: parameter %v leaks to %s with derefs=%d:\n", base.FmtPos(l.n.Pos()), l.n, b.explainLoc(root), derefs)
					}
					explanation := b.explainPath(root, l)
					if logopt.Enabled() {
						var e_curfn *ir.Func // TODO(mdempsky): Fix.
						logopt.LogOpt(l.n.Pos(), "leak", "escape", ir.FuncName(e_curfn),
							fmt.Sprintf("parameter %v leaks to %s with derefs=%d", l.n, b.explainLoc(root), derefs), explanation)
					}
				}
				l.leakTo(root, derefs)
			}

			// root的生命周期大于l,且root引用了l,则需要把l的escape值置换true
			if addressOf && !l.escapes {
				l.escapes = true
				// 还需要把l入队walk队列,递归l的边里需要escape的location
				enqueue(l)
				continue
			}
		}

		// 遍历所有指向当前location的边
		for i, edge := range l.edges {
			if edge.src.escapes { // 边的Src location已经确定要逃逸了,不需要再做处理
				continue
			}

			d := derefs + edge.derefs
			if edge.src.walkgen != walkgen || edge.src.derefs > d {
				//当前location相对于初始root location的引用信息权值小于已知的最小引用信息权值,则更新这个信息
				edge.src.walkgen = walkgen
				edge.src.derefs = d 
				edge.src.dst = l
				edge.src.dstEdgeIdx = i
				todo = append(todo, edge.src) // 将最新的最小引用权值的src location加入到todo列表中
			}
		}
	}
}
```

#### finish 对解析后的escape结果做些收尾工作
```go
// src/cmd/compile/internal/escape/escape.go
func (b *batch) finish(fns []*ir.Func) {
	for _, fn := range fns {
		fn.SetEsc(escFuncTagged) // 给函数设置escape阶段的状态
		// ...
	}

	for _, loc := range b.allLocs {
		n := loc.n
		// ...

		if loc.escapes { // 当location的escapes分析结果为true时,设置节点的Esc属性为 EscHeap
			
			n.SetEsc(ir.EscHeap)
		} else {
			n.SetEsc(ir.EscNone) // 普通location 设置Esc属性为EscNone
			if loc.transient { // todo transient的用处
				switch n.Op() {
				case ir.OCLOSURE:
					n := n.(*ir.ClosureExpr)
					n.SetTransient(true)
				case ir.OCALLPART:
					n := n.(*ir.SelectorExpr)
					n.SetTransient(true)
				case ir.OSLICELIT:
					n := n.(*ir.CompLitExpr)
					n.SetTransient(true)
				}
			}
		}
	}
}
```
