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
// todo Batch
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

	// ...todo
	// We've walked the function bodies, so we've seen everywhere a
	// variable might be reassigned or have it's address taken. Now we
	// can decide whether closures should capture their free variables
	// by value or reference.
	for _, closure := range b.closures {
		b.flowClosure(closure.k, closure.clo)
	}
	b.closures = nil

	// 遍历所有变量的location结构,形成一张变量节点间的有向图 #ref flow
	for _, loc := range b.allLocs {
		if why := HeapAllocReason(loc.n); why != "" {
			b.flow(b.heapHole().addr(loc.n, why), loc)
		}
	}

	// 遍历所有location,根据有向图,给需要逃逸的变量打上escapes标记
	b.walkAll()
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

	// 遍历当前Func的所有声明
	for _, n := range fn.Dcl {
		if n.Op() == ir.ONAME {
			// 为每一个变量创建一个location #ref newLoc
			e.newLoc(n, false)
		}
	}

	// 当前Func的返回参数的处理
	// todo ??
	for i, f := range fn.Type().Results().FieldSlice() {
		e.oldLoc(f.Nname.(*ir.Name)).resultIndex = 1 + i
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
```

#### walkFunc Func节点的
什么情形下需要把变量逃逸到堆上?
##### return
##### 闭包
##### 大变量
##### 循环逃逸
```go
// src/cmd/compile/internal/escape/escape.go
func (b *batch) walkFunc(fn *ir.Func) {
	e := b.with(fn)
	fn.SetEsc(escFuncStarted)

	//Goto相关语法的处理 todo
	ir.Visit(fn, func(n ir.Node) {
		switch n.Op() {
		case ir.OLABEL:
			n := n.(*ir.LabelStmt)
			if e.labels == nil {
				e.labels = make(map[*types.Sym]labelState)
			}
			e.labels[n.Label] = nonlooping

		case ir.OGOTO:
			// If we visited the label before the goto,
			// then this is a looping label.
			n := n.(*ir.BranchStmt)
			if e.labels[n.Label] == nonlooping {
				e.labels[n.Label] = looping
			}
		}
	})

	// 解析节点的body #ref block
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

	lno := ir.SetPos(n)
	defer func() {
		base.Pos = lno
	}()

	e.stmts(n.Init())

	switch n.Op() {
	default:
		base.Fatalf("unexpected stmt: %v", n)

	case ir.ODCLCONST, ir.ODCLTYPE, ir.OFALL, ir.OINLMARK:
		// nop

	case ir.OBREAK, ir.OCONTINUE, ir.OGOTO:
		// TODO(mdempsky): Handle dead code?

	case ir.OBLOCK:
		n := n.(*ir.BlockStmt)
		e.stmts(n.List)

	case ir.ODCL:
		// Record loop depth at declaration.
		n := n.(*ir.Decl)
		if !ir.IsBlank(n.X) {
			e.dcl(n.X)
		}

	case ir.OLABEL:
		n := n.(*ir.LabelStmt)
		switch e.labels[n.Label] {
		case nonlooping:
			if base.Flag.LowerM > 2 {
				fmt.Printf("%v:%v non-looping label\n", base.FmtPos(base.Pos), n)
			}
		case looping:
			if base.Flag.LowerM > 2 {
				fmt.Printf("%v: %v looping label\n", base.FmtPos(base.Pos), n)
			}
			e.loopDepth++
		default:
			base.Fatalf("label missing tag")
		}
		delete(e.labels, n.Label)

	case ir.OIF:
		n := n.(*ir.IfStmt)
		e.discard(n.Cond)
		e.block(n.Body)
		e.block(n.Else)

	case ir.OFOR, ir.OFORUNTIL:
		n := n.(*ir.ForStmt)
		e.loopDepth++
		e.discard(n.Cond)
		e.stmt(n.Post)
		e.block(n.Body)
		e.loopDepth--

	case ir.ORANGE:
		// for Key, Value = range X { Body }
		n := n.(*ir.RangeStmt)

		// X is evaluated outside the loop.
		tmp := e.newLoc(nil, false)
		e.expr(tmp.asHole(), n.X)

		e.loopDepth++
		ks := e.addrs([]ir.Node{n.Key, n.Value})
		if n.X.Type().IsArray() {
			e.flow(ks[1].note(n, "range"), tmp)
		} else {
			e.flow(ks[1].deref(n, "range-deref"), tmp)
		}
		e.reassigned(ks, n)

		e.block(n.Body)
		e.loopDepth--

	case ir.OSWITCH:
		n := n.(*ir.SwitchStmt)

		if guard, ok := n.Tag.(*ir.TypeSwitchGuard); ok {
			var ks []hole
			if guard.Tag != nil {
				for _, cas := range n.Cases {
					cv := cas.Var
					k := e.dcl(cv) // type switch variables have no ODCL.
					if cv.Type().HasPointers() {
						ks = append(ks, k.dotType(cv.Type(), cas, "switch case"))
					}
				}
			}
			e.expr(e.teeHole(ks...), n.Tag.(*ir.TypeSwitchGuard).X)
		} else {
			e.discard(n.Tag)
		}

		for _, cas := range n.Cases {
			e.discards(cas.List)
			e.block(cas.Body)
		}

	case ir.OSELECT:
		n := n.(*ir.SelectStmt)
		for _, cas := range n.Cases {
			e.stmt(cas.Comm)
			e.block(cas.Body)
		}
	case ir.ORECV:
		// TODO(mdempsky): Consider e.discard(n.Left).
		n := n.(*ir.UnaryExpr)
		e.exprSkipInit(e.discardHole(), n) // already visited n.Ninit
	case ir.OSEND:
		n := n.(*ir.SendStmt)
		e.discard(n.Chan)
		e.assignHeap(n.Value, "send", n)

	case ir.OAS:
		n := n.(*ir.AssignStmt)
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

	case ir.OAS2FUNC:
		n := n.(*ir.AssignListStmt)
		e.stmts(n.Rhs[0].Init())
		ks := e.addrs(n.Lhs)
		e.call(ks, n.Rhs[0], nil)
		e.reassigned(ks, n)
	case ir.ORETURN:
		n := n.(*ir.ReturnStmt)
		results := e.curfn.Type().Results().FieldSlice()
		dsts := make([]ir.Node, len(results))
		for i, res := range results {
			dsts[i] = res.Nname.(*ir.Name)
		}
		e.assignList(dsts, n.Results, "return", n)
	case ir.OCALLFUNC, ir.OCALLMETH, ir.OCALLINTER, ir.OCLOSE, ir.OCOPY, ir.ODELETE, ir.OPANIC, ir.OPRINT, ir.OPRINTN, ir.ORECOVER:
		e.call(nil, n, nil)
	case ir.OGO, ir.ODEFER:
		n := n.(*ir.GoDeferStmt)
		e.stmts(n.Call.Init())
		e.call(nil, n.Call, n)

	case ir.OTAILCALL:
		// TODO(mdempsky): Treat like a normal call? esc.go used to just ignore it.
	}
}
```

#### flow 更具前面关于变量的解析信息,形成一个`图`
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
	if dst.escapes && k.derefs < 0 { // dst = &src
		if base.Flag.LowerM >= 2 || logopt.Enabled() {
			pos := base.FmtPos(src.n.Pos())
			if base.Flag.LowerM >= 2 {
				fmt.Printf("%s: %v escapes to heap:\n", pos, src.n)
			}
			explanation := b.explainFlow(pos, dst, src, k.derefs, k.notes, []*logopt.LoggedOpt{})
			if logopt.Enabled() {
				var e_curfn *ir.Func // TODO(mdempsky): Fix.
				logopt.LogOpt(src.n.Pos(), "escapes", "escape", ir.FuncName(e_curfn), fmt.Sprintf("%v escapes to heap", src.n), explanation)
			}

		}
		src.escapes = true
		return
	}

	// TODO(mdempsky): Deduplicate edges?
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

#### finish
```go
// src/cmd/compile/internal/escape/escape.go
func (b *batch) finish(fns []*ir.Func) {
	// Record parameter tags for package export data.
	for _, fn := range fns {
		fn.SetEsc(escFuncTagged)

		narg := 0
		for _, fs := range &types.RecvsParams {
			for _, f := range fs(fn.Type()).Fields().Slice() {
				narg++
				f.Note = b.paramTag(fn, narg, f)
			}
		}
	}

	for _, loc := range b.allLocs {
		n := loc.n
		if n == nil {
			continue
		}
		if n.Op() == ir.ONAME {
			n := n.(*ir.Name)
			n.Opt = nil
		}

		// Update n.Esc based on escape analysis results.

		if loc.escapes {
			if n.Op() == ir.ONAME {
				if base.Flag.CompilingRuntime {
					base.ErrorfAt(n.Pos(), "%v escapes to heap, not allowed in runtime", n)
				}
				if base.Flag.LowerM != 0 {
					base.WarnfAt(n.Pos(), "moved to heap: %v", n)
				}
			} else {
				if base.Flag.LowerM != 0 {
					base.WarnfAt(n.Pos(), "%v escapes to heap", n)
				}
				if logopt.Enabled() {
					var e_curfn *ir.Func // TODO(mdempsky): Fix.
					logopt.LogOpt(n.Pos(), "escape", "escape", ir.FuncName(e_curfn))
				}
			}
			n.SetEsc(ir.EscHeap)
		} else {
			if base.Flag.LowerM != 0 && n.Op() != ir.ONAME {
				base.WarnfAt(n.Pos(), "%v does not escape", n)
			}
			n.SetEsc(ir.EscNone)
			if loc.transient {
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
