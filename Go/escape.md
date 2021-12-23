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
## `escape`分析过程入口

### Visitor 结构
```go
// cmd/compile/internal/ir/scc.go
type bottomUpVisitor struct {
	analyze  func([]*Func, bool) // 注入的分析方法组
	visitgen uint32 // 自增visitID
	nodeID   map[*Func]uint32 // 纪录一个函数是否已经被visit过
	stack    []*Func
}
```

### escape 结构
```go
// cmd/compile/internal/escape/escape.go
type batch struct {
	allLocs  []*location
	closures []closure

	heapLoc  location // 堆location
	blankLoc location // 空白location 用来存放一些 匿名或或用不到的变量 比如 _ := XXX 语句就需要为 "_"创建一个hole,这个hole的dst就是blankLoc
}

// 每解析一个函数时需要新建一个这样的结构,包裹了传入的batch,当前要解析的函数节点curfn,以及一个状态标记loopDepth表明当前函数内的解析深度(比如解析到一个for循环内时,loopDepth+1)
type escape struct {
	*batch
	curfn *ir.Func // 当前解析的函数

	labels map[*types.Sym]labelState // known labels

	loopDepth int //当前解析的深度(比如解析到当前函数的一个for循环内时,loopDepth+1)
}

// 表示一个赋值语句的hole
// 比如 "x = **p", 就新建一个 hole,  dst==x and derefs==2.
type hole struct {
	dst    *location
	derefs int // >= -1
	notes  *note

	// addrtaken indicates whether this context is taking the address of
	// the expression, independent of whether the address will actually
	// be stored into a variable.
	addrtaken bool

	// uintptrEscapesHack indicates this context is evaluating an
	// argument for a //go:uintptrescapes function.
	uintptrEscapesHack bool
}

// location 表示一个变量位置
type location struct {
	n         ir.Node  // 节点信息
	curfn     *ir.Func // 变量所在的函数
	edges     []edge   // 所有指向当前变量的边 #ref struct edge
	loopDepth int      // loopDepth at declaration

	// 如果当前变量是函数的返回属性,则这个数值不为0,数值n代表函数的第n个返回属性
	resultIndex int

	derefs  int // 用来记录在escape walk过程中,当前location 和 root location 的derefs值
	walkgen uint32

	// dst and dstEdgeindex track the next immediate assignment
	// destination location during walkone, along with the index
	// of the edge pointing back to this location.
	dst        *location
	dstEdgeIdx int

	queued bool // 在escape解析过程中标记这个location是否已经被push到了要作为root来walk的 todo队列里

	// 标记一个变量是否必须保存到堆上 escape!
	escapes bool

	// 如果变量的生命周期没有超过声明所在stack,则这个值为true
	transient bool

	// paramEsc records the represented parameter's leak set.
	paramEsc leaks

	captured   bool // 是否有一个闭包函数用到了这个变量
	reassigned bool // has this variable been reassigned?
	addrtaken  bool // has this variable's address been taken?
}

type edge struct {
	src    *location // 边的src的location (dst := src)
	derefs int // 边相对于dst的derefs值
	notes  *note
}
```
### Batch方法
分析调用的analyze方法是这里注入的 `Batch`方法
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
			// b.heapHole() 新建了一个 dst location指向 heapLoc的hole,
			// addr(loc.n, why)将新建的location的derefs值 -1
			// flow增加一个该hole 与 location的有向边
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

// 给这个变量新建一个location,初始化时需要指定transient属性
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
		e.reassigned(ks, n) // 尝试给ks(dst为key的location的hole)打上reassigned标记

		e.block(n.Body) // 递归解析range的body内容
		e.loopDepth--

	case ir.OSWITCH: // switch语法
		n := n.(*ir.SwitchStmt)
		// ...
		for _, cas := range n.Cases { // 遍历每一个case节点
			e.discards(cas.List) // case的值都可以创建一个dst为blankLoc的hole,然后创建一个flow到这个dst的边
			e.block(cas.Body) // 递归解析所有case的body内容
		}

	case ir.OSELECT: // select 语法
		n := n.(*ir.SelectStmt)
		for _, cas := range n.Cases {
			e.stmt(cas.Comm)
			e.block(cas.Body) // 递归解析所有case的body内容
		}
	case ir.ORECV: // <-X语法 阻塞直到从Chan X中得到数据(但这个数据又不需要保存)
		n := n.(*ir.UnaryExpr)
		// e.discardHole()创建一个dst为blankLoc 的hole, 
		// exprSkipInit()解析变量n的形式(得到derefs值),创建一条dst 为blankLoc, src为n的location的边
		e.exprSkipInit(e.discardHole(), n)  
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

	case ir.OAS2FUNC: // 语句是将函数调用结果赋值时
		n := n.(*ir.AssignListStmt)
		ks := e.addrs(n.Lhs) // 为每一个左值生成一个hole(derefs 值 -1)
		e.call(ks, n.Rhs[0], nil) // 把右值作为一个整体 和 左值的hole进行分析 #ref escape.call
		e.reassigned(ks, n) // 尝试给ks(dst为左值的location的hole) 打上reassigned标记
	case ir.ORETURN: // 函数的return
		n := n.(*ir.ReturnStmt)
		results := e.curfn.Type().Results().FieldSlice()
		dsts := make([]ir.Node, len(results))
		for i, res := range results { // 为return的所有属性占一个位置
			dsts[i] = res.Nname.(*ir.Name)
		}
		e.assignList(dsts, n.Results, "return", n)
	case ir.OCALLFUNC, ir.OCALLMETH, ir.OCALLINTER, ir.OCLOSE, ir.OCOPY, ir.ODELETE, ir.OPANIC, ir.OPRINT, ir.OPRINTN, ir.ORECOVER:
		e.call(nil, n, nil)·
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
	loc.loopDepth = e.loopDepth //将location的loopDepth 值变为当前函数的解析深度(e.loopDepth)
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

	e.reassigned(ks, where) // 尝试给ks(包裹dst location的hole) 打上 reassigned 标记
}

// 创建一个srcNode(n) 到 dstHole(k) 的有向边
func (e *escape) expr(k hole, n ir.Node) {
	if n == nil {
		return
	}
	// 解析赋值语句右边的表达式 #ref exprSkipInit
	e.exprSkipInit(k, n)
}

// 解析一个变量节点的表达式(变量的形式可能是X, *X, &X, make xxxxx, 闭包等等)
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

	/****** new, make 等等语法需要调用spill(这个方法内会调用 k.addr(n,“xxx“))将hole k 的derefs值-1,然后生成一个dst为n节点的location,然后将这个新生成的location流向hole k(生成有向边) *****/
	case ir.ONEW: // new
		n := n.(*ir.UnaryExpr)
		e.spill(k, n) 

	case ir.OMAKESLICE: //make []
		n := n.(*ir.MakeExpr)
		e.spill(k, n)
		// make 语法的 len 和 cap参数只需要生成 一个 dst为 _blankLoc 的hole,然后用这个hole生成与对应参数的有向边
		e.discard(n.Len) 
		e.discard(n.Cap)
	case ir.OMAKEMAP: //make map
		n := n.(*ir.MakeExpr)
		e.spill(k, n)
		e.discard(n.Len)
	case ir.OCALLPART: // 方法(非调用)
		n := n.(*ir.SelectorExpr)
		// e.spill(k,n)先为右值节点创建一个新location, 将左值的hole k的derefs值-1,新location流向hole, 
		// 然后返回的是新建的一个包裹了新location的hole
		// closureK 就是dst为右值节点location的 hole
		closureK := e.spill(k, n) 
		m := n.Selection

		// 保守的为该方法所有的出参创建一个heapHole
		var ks []hole
		for i := m.Type.NumResults(); i > 0; i-- {
			ks = append(ks, e.heapHole())
		}
		name, _ := m.Nname.(*ir.Name) // Nname是该方法的节点结构
		paramK := e.tagHole(ks, name, m.Type.Recv()) // 返回一个包裹新建的临时location的hole,这个临时location流向heapLoc,也流向函数的出参
		// closureK 是调用的方法的hole
		// teeHole返回一个包裹新建location得hole, 这个location流向参数里的每一个hole(这里是paramK 和 colosureK)
		e.expr(e.teeHole(paramK, closureK), n.X) // 解析右边节点的表达式得到derefs值, 调整左边的hole的derefs值,将右边节点的location流向左边调整derefs值的hole; 注:这里右边节点是 X.sel 的X, 不是X.sel本身

	// ...
	/****   ***************************** *****/
	// 闭包
	case ir.OCLOSURE:
		n := n.(*ir.ClosureExpr)
		k = e.spill(k, n) // spill先为闭包本身创建一个location ,再创建一个k(左值的hole)到该location 的 derefs值为-1的有向边 #ref spill
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

// spill 为n生成一个新location, 然后调用k.addr(n, "spill"),将hole k的derefs值-1, 将新生成的location流向hole k(生成有向边)
func (e *escape) spill(k hole, n ir.Node) hole {
	loc := e.newLoc(n, true)
	e.flow(k.addr(n, "spill"), loc)
	return loc.asHole() // 返回的是一个包裹新location的hole(有的地方需要用到)
}

// 当一个函数调用时,传入了外部参数,函数并不直接使用这些外部变量,而是使用ta们的复制,
// 所以需要新建一个location,一个dst为这个location的hole,
// 如果函数的入参会`leak`到函数的返回结果,还需要把这个hole流向函数对应的返回结构(即ks)
// 参数ks是函数调用结果的流向(接受方变量), fn是调用的函数, param是函数的参数本身
func (e *escape) tagHole(ks []hole, fn *ir.Name, param *types.Field) hole {

	if e.inMutualBatch(fn) { // todo ??
		return e.addr(ir.AsNode(param.Nname))
	}

	var tagKs []hole

	esc := parseLeaks(param.Note) // 根据参数的‘注解’初始化一个leaks结构,一般就是认为没有特殊注解
	if x := esc.Heap(); x >= 0 { //没有注解的情况话,这个x=0
		// e.heapHole() 创建一个dst为heapLoc的hole, push到tagKs中
		tagKs = append(tagKs, e.heapHole().shift(x))
	}
	// 把函数的出参也push到tagks中
	if ks != nil {
		for i := 0; i < numEscResults; i++ {
			if x := esc.Result(i); x >= 0 {
				tagKs = append(tagKs, ks[i].shift(x))
			}
		}
	}
	// e.teeHole()创建一个新location, 流向tagKs里的每一个hole
	return e.teeHole(tagKs...)
}

// 根据参数的‘注解’初始化一个leaks结构
func parseLeaks(s string) leaks {
	var l leaks
	if !strings.HasPrefix(s, "esc:") { // 一般的变量就是走这个逻辑,即没有特殊注解
		l.AddHeap(0)
		return l
	}
	copy(l[:], s[4:])
	return l
}

// 创建一个新location, 将这个hole流入参数里的每一个hole,
// 然后创建一个新hole,dst就是这个新location,并返回
func (e *escape) teeHole(ks ...hole) hole {
	// 创建一个新location
	loc := e.newLoc(nil, true)
	for _, k := range ks { // 遍历参数 hole 数组
		// 将新location 流向每一个参数hole
		e.flow(k, loc)
	}
	return loc.asHole()// 返回新建的包裹新location的hole
}
```

##### call 把函数调用当作一个整体 和左值 作escape分析
参数ks是函数调用结果的流向目的, call是函数本身, where语句节点(比如defer和go语法的节点包含了函数信息,也包含了defer 和 go的信息)
```go
func (e *escape) call(ks []hole, call, where ir.Node) {
	topLevelDefer := where != nil && where.Op() == ir.ODEFER && e.loopDepth == 1 // 一个函数的顶层defer语句需要作特别标记
	if topLevelDefer {
		// 标记整个defer节点为EscNever(never escape)
		where.SetEsc(ir.EscNever)
	}

	// 对函数的入参的流向(arg为入参的原始节点),流向函数
	argument := func(k hole, arg ir.Node) {
		if topLevelDefer { // 函数的defer处理
			// 相当于新建了个虚拟location, 新location流向 原来的入参k(hole) 所包裹的location #ref e.later
			k = e.later(k) 
		} else if where != nil { // 目前已知只有 go xxx() 或 defer xxx()的时候 这个where != nil
			k = e.heapHole() // 直接新建一个dst为heapLoc的hole
		}
		// 递归执行 hole k 与 入参原始节点间的escpae分析,生成有向边
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
				e.expr(ks[i], ir.AsNode(result.Nname)) // 在这里,‘解析函数的返回表达式’,流向`调用方接受函数结果的变量`
		}

		if r := fntype.Recv(); r != nil { // 方法有一个receiver时,相当于把receiver本身作为函数的一个参数处理
			//先调用e.tagHole新建一个包裹临时location得hole,将新建的hole流向heapLoc也流向函数的所有出参hole
			// 其中‘ks’是函数调用方用来接收函数调用结果的变量(们),‘fn‘是函数,’r‘是receiver
			// 然后调用argument 解析函数本身的表达式,流向新生成的hole
			argument(e.tagHole(ks, fn, r), call.X.(*ir.SelectorExpr).X)
		} else {
			argument(e.discardHole(), call.X) // e.discardHole()生成一个dst为blankLoc的hole,调用argument解析函数表达式,将函数表达式的location,流向新生成的hole
		}

		args := call.Args // 函数的参数与调用方的 有向边分析
		for i, param := range fntype.Params().FieldSlice() { 
			// e.tagHole() 为函数的参数新建一个包裹临时location得hole,这个hole流向函数的所有出参,
			// 然后将函数的入参所在的location流向新建的的hole
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

// 用来处理顶层defer语句,相当于新建一个虚拟location,然后把这个新location流向 原来的k(hole)所包裹的location(建立有向边)
func (e *escape) later(k hole) hole {
	loc := e.newLoc(nil, false) // 生成一个节点为nil的location
	e.flow(k, loc) // 新location流向参数hole k
	return loc.asHole()
}
```

##### assignHeap 相当于为要发送到chan 的变量创建一个临时变量,这个临时变量的位置放在堆上,再计算变量与临时变量的有向边数值
```go
func (e *escape) assignHeap(src ir.Node, why string, where ir.Node) {
	e.expr(e.heapHole().note(where, why), src)
}
```

##### reassigned 给一个location标记reassigned属性
```go
func (e *escape) reassigned(ks []hole, where ir.Node) {
	// ...

	for _, k := range ks {
		loc := k.dst
		// Variables declared by range statements are assigned on every iteration.
		if n, ok := loc.n.(*ir.Name); ok && n.Defn == where && where.Op() != ir.ORANGE {
			continue
		}
		loc.reassigned = true
	}
}
```

#### flowClosure 解析闭包
```go
func (b *batch) flowClosure(k hole, clo *ir.ClosureExpr) {
	for _, cv := range clo.Func.ClosureVars { // 遍历闭包所有用到的外部变量(闭包在使用外部变量时其实已经copy了一份,所以后面需要用n := cn.Cannoical来找到原始变量的一些属性)
		n := cv.Canonical() // 原始变量节点
		loc := b.oldLoc(cv) // 找到外部变量的原始location
		n.SetByval(!loc.addrtaken && !loc.reassigned && n.Type().Size() <= 128) // 根据属性设置原始变量节点的 `byval`属性(传值)
		if !n.Byval() {
			n.SetAddrtaken(true) // 如果原始变量不是值(是个址),需要设置节点flag为`Addrtaken`
		}
		k := k
		if !cv.Byval() { // 如果闭包使用的是个地址,需要把闭包本身的hole作处理(derefs -1)
			k = k.addr(cv, "reference")
		}
		// 用闭包的hole, 建立一条由闭包到变量location的有向边
		b.flow(k.note(cv, "captured by a closure"), loc)
	}
}
```

#### flow 生成一条边,根据dst的escapes值和边的derefs值决定src要不要escape
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

#### walkAll 遍历所有location,根据已经escape的location,标记所有需要escape的location
```go
// src/cmd/compile/internal/escape/escape.go
func (b *batch) walkAll() {
	// 将所有location结构存入一个队列‘todo’中,这个队列里的多有元素都将作为root walk一次
	todo := make([]*location, 0, len(b.allLocs)+1)

	// 定义了一个enqueue方法,把一个location入队 todo 队列中
	enqueue := func(loc *location) {
		if !loc.queued {
			todo = append(todo, loc)
			loc.queued = true //标记queued,当ta被拎出来walk时,会先标记回false,避免无限循环
		}
	}

	for _, loc := range b.allLocs { // 所有location都入队
		enqueue(loc)
	}
	enqueue(&b.heapLoc) // 把heapLoc也push到需要作为root walk的队列里(因为很多特殊变量会直接流向这个heapLoc)

	var walkgen uint32
	for len(todo) > 0 {
		// 相当于pop出todo的最后一个元素
		root := todo[len(todo)-1]
		todo = todo[:len(todo)-1]

		root.queued = false // 标记queued 为false,防无限循环

		walkgen++
		// #ref walkOne 深入root节点,同时传入了enqueue方法,当与root相关的location 需要escape时,调用enqueue将该location入队todo中,接下来的循环中需要以该location为root,递归该root的边和需要escape的location
		b.walkOne(root, walkgen, enqueue)
	}
}

// 以一个location为root,遍历该location的边(以及边的src的边,以及...),标记需要escape的location并通过enqueue将该location入队需要walk的节点列表中
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
		// 后续的遍历当出现derefs < 0 时表明该location被root节点引用了地址
		addressOf := derefs < 0
		if addressOf {·
			// For a flow path like "root = &l; l = x",
			// l's address flows to root, but x's does
			// not. We recognize this by lower bounding
			// derefs at 0.
			derefs = 0 

			// root 非临时变量, l的dst为root时,l也不能为临时变量(因为root的值为l的地址,l临时的话,root有指向未知区域的风险)
			if !root.transient && l.transient { // 这个&&后面的条件 和下面的设置值为false,防止了重复enqueue一个变量location
				l.transient = false
				enqueue(l) // 将l push到 需要作为root walk的队列里
			}
		}

		// 判断root是否在l的生命周期外依然存在
		if b.outlives(root, l) {
			// l 是函数的入参的情形处理
			if l.isName(ir.PPARAM) {
				l.leakTo(root, derefs) // 判断是否需要leakTo #ref leakTo
			}

			// root的生命周期大于l,且root(dst)与l(src)的边小于0,则需要把l的escapes值置换true
			if addressOf && !l.escapes {
				l.escapes = true
				// 还需要把l入队walk队列,递归l的边里需要escape的location(l已经escape了,那么流向l的变量可能也需要escape)
				enqueue(l)
				continue
			}
		}

		// 遍历所有当前location的边
		for i, edge := range l.edges {
			if edge.src.escapes { // 边的Src location已经确定要逃逸了,不需要再做处理
				continue
			}
			d := derefs + edge.derefs // 当前location相对root的derefs + 当前location的边的derefs
			if edge.src.walkgen != walkgen || edge.src.derefs > d {
				//当前location相对于初始root location的derefs值小于已知的这条边的src location的derefs值,则更新这条边的src location的信息
				edge.src.walkgen = walkgen
				edge.src.derefs = d 
				edge.src.dst = l
				edge.src.dstEdgeIdx = i
				todo = append(todo, edge.src) // 更新了信息后的边的src 需要加入到当前root的walk todo列表中
			}
		}
	}
}
```

##### leakTo(sink, derefs) 
只有一个函数的入参会调用这个leakTo
(l *location)是一个函数的入参,且流向了另一个函数内的变量(sink);
这个`leak`不是指内存泄漏，而是指该传入参数的内容的生命期，超过函数调用期，也就是函数返回后，该参数的内容仍然存活
```go
func (l *location) leakTo(sink *location, derefs int) {
	if !sink.escapes && sink.isName(ir.PPARAMOUT) && sink.curfn == l.curfn { // sink还没有被标记为escapes, sink是函数的返回属性,sink与l在同一个函数内
		ri := sink.resultIndex - 1
		if ri < numEscResults {
			// 设置入参location 的 paramEsc,表明这个入参会出参到函数的第几个返回值
			l.paramEsc.AddResult(ri, derefs)
			return
		}
	}

	// todo paramEsc 在哪里用到了
	l.paramEsc.AddHeap(derefs)
}
```

##### outlives(l, other) 判断`l`的存在时间是否大于`other`在stack上的生命周期
```go
func (b *batch) outlives(l, other *location) bool {
	if l.escapes { // l已经被标记需要escape到堆上了,堆的存在时间大于所有

		return true
	}

	// 当l是一个函数的返回属性时,因为不知道ta返回后会被调用者怎么折腾,保守的认为ta比其他相关变量声明周期更长
	if l.isName(ir.PPARAMOUT) {
		// 例外情况: 闭包的直接调用
		if containsClosure(other.curfn, l.curfn) && l.curfn.ClosureCalled() {
			return false
		}

		return true
	}

	// l 和 other 在同一个函数内,但l的在other所在的循环block{}的外面时,也认为l的生命周期大于other
	if l.curfn == other.curfn && l.loopDepth < other.loopDepth {
		return true
	}

	// other所在的函数是l所在的函数的内的闭包时,认为l的生命周期大雨other
	if containsClosure(l.curfn, other.curfn) {
		return true
	}

	return false
}
```

#### finish 对解析后的escape结果做些收尾工作
```go
// src/cmd/compile/internal/escape/escape.go
func (b *batch) finish(fns []*ir.Func) {
	for _, fn := range fns {
		fn.SetEsc(escFuncTagged) // 给函数设置escape阶段的状态

		narg := 0
		for _, fs := range &types.RecvsParams {
			for _, f := range fs(fn.Type()).Fields().Slice() {
				narg++
				f.Note = b.paramTag(fn, narg, f) //todo
			}
		}
	}

	for _, loc := range b.allLocs {
		n := loc.n
		// ...

		if loc.escapes { // 当location的escapes分析结果为true时,设置节点的Esc属性为 EscHeap
			
			n.SetEsc(ir.EscHeap)
		} else {
			n.SetEsc(ir.EscNone) // 普通location 设置Esc属性为EscNone
			if loc.transient { // 设置变量节点的transient属性
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
