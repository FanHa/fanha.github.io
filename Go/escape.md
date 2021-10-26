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
			b.walkFunc(fn)
		}
	}

	// We've walked the function bodies, so we've seen everywhere a
	// variable might be reassigned or have it's address taken. Now we
	// can decide whether closures should capture their free variables
	// by value or reference.
	for _, closure := range b.closures {
		b.flowClosure(closure.k, closure.clo)
	}
	b.closures = nil

	for _, loc := range b.allLocs {
		if why := HeapAllocReason(loc.n); why != "" {
			b.flow(b.heapHole().addr(loc.n, why), loc)
		}
	}

	b.walkAll()
	b.finish(fns)
}
```

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

	// Initialize resultIndex for result parameters.
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

	loc := &location{
		n:         n,
		curfn:     e.curfn,
		loopDepth: e.loopDepth,
		transient: transient,
	}
	e.allLocs = append(e.allLocs, loc)
	if n != nil {
		if n.Op() == ir.ONAME {
			n := n.(*ir.Name)
			if n.Curfn != e.curfn {
				base.Fatalf("curfn mismatch: %v != %v for %v", n.Curfn, e.curfn, n)
			}

			if n.Opt != nil {
				base.Fatalf("%v already has a location", n)
			}
			n.Opt = loc
		}
	}
	return loc
}
```