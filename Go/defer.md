## 版本
+ Github:golang/go
+ 分支:release-branch.go.1.17

## 生成ast时的defer 
从这里看出生成ast时并没有对defer后面跟的方法作特殊处理
```go
// cmd/compile/internal/walk/stmt.go
func walkStmt(n ir.Node) ir.Node {
    switch n.Op() {
    // ...
    case ir.ODEFER:
		n := n.(*ir.GoDeferStmt)
		ir.CurFunc.SetHasDefer(true) // 设置当前编译的函数的‘HasDefer’属性为true
		ir.CurFunc.NumDefers++ // 当前编译的函数的defer数量+1
		if ir.CurFunc.NumDefers > maxOpenDefers {
			// 一个函数最多允许 maxOpenDefers 个defer语句, todo openCoded
			ir.CurFunc.SetOpenCodedDeferDisallowed(true)
		}
        // ...
		fallthrough
	case ir.OGO:
		n := n.(*ir.GoDeferStmt)
		return walkGoDefer(n) // defer和go的解析都需要调用walkGoDefer #ref walkGoDefer
    // ...
    }
}

func walkGoDefer(n *ir.GoDeferStmt) ir.Node {
	var init ir.Nodes
	switch call := n.Call; call.Op() { //取出defer 后面跟着的函数节点
	// ...

	case ir.OCALLFUNC, ir.OCALLMETH, ir.OCALLINTER:
		call := call.(*ir.CallExpr)
		if len(call.KeepAlive) > 0 { // ?todo KeepAlive 的意义
			n.Call = wrapCall(call, &init)
		} else {
			n.Call = walkExpr(call, &init)
		}

	default:
		n.Call = walkExpr(call, &init)
	}
	if len(init) > 0 {
		init.Append(n)
		return ir.NewBlockStmt(n.Pos(), init)
	}
	return n
}
```

## 编译时的defer
```go
// internal/ssagen/pgen.go
func Compile(fn *ir.Func, worker int) {
	f := buildssa(fn, worker) // todo #ref buildssa
	// Note: check arg size to fix issue 25507.
	if f.Frontend().(*ssafn).stksize >= maxStackSize || f.OwnAux.ArgWidth() >= maxStackSize {
		largeStackFramesMu.Lock()
		largeStackFrames = append(largeStackFrames, largeStack{locals: f.Frontend().(*ssafn).stksize, args: f.OwnAux.ArgWidth(), pos: fn.Pos()})
		largeStackFramesMu.Unlock()
		return
	}
	pp := objw.NewProgs(fn, worker)
	defer pp.Free()
	genssa(f, pp)
	// Check frame size again.
	// The check above included only the space needed for local variables.
	// After genssa, the space needed includes local variables and the callee arg region.
	// We must do this check prior to calling pp.Flush.
	// If there are any oversized stack frames,
	// the assembler may emit inscrutable complaints about invalid instructions.
	if pp.Text.To.Offset >= maxStackSize {
		largeStackFramesMu.Lock()
		locals := f.Frontend().(*ssafn).stksize
		largeStackFrames = append(largeStackFrames, largeStack{locals: locals, args: f.OwnAux.ArgWidth(), callee: pp.Text.To.Offset - locals, pos: fn.Pos()})
		largeStackFramesMu.Unlock()
		return
	}

	pp.Flush() // assemble, fill in boilerplate, etc.
	// fieldtrack must be called after pp.Flush. See issue 20014.
	fieldtrack(pp.Text.From.Sym, fn.FieldTrack)
}

```

### buildssa
```go
// internal/ssagen/ssa.go
func buildssa(fn *ir.Func, worker int) *ssa.Func {
	// ...
	var s state // 初始化一个 state结构,记录当前Func的build情况 #ref state 

	s.hasdefer = fn.HasDefer() // 根据ast的信息,设置当前func是否有defer语句

	s.hasOpenDefers = base.Flag.N == 0 && s.hasdefer && !s.curfn.OpenCodedDeferDisallowed()
	switch {
	case base.Debug.NoOpenDefer != 0:
		s.hasOpenDefers = false
	case s.hasOpenDefers && (base.Ctxt.Flag_shared || base.Ctxt.Flag_dynlink) && base.Ctxt.Arch.Name == "386":
		// Don't support open-coded defers for 386 ONLY when using shared
		// libraries, because there is extra code (added by rewriteToUseGot())
		// preceding the deferreturn/ret code that we don't track correctly.
		s.hasOpenDefers = false
	}
	if s.hasOpenDefers && len(s.curfn.Exit) > 0 {
		// Skip doing open defers if there is any extra exit code (likely
		// race detection), since we will not generate that code in the
		// case of the extra deferreturn/ret segment.
		s.hasOpenDefers = false
	}
	if s.hasOpenDefers {
		// Similarly, skip if there are any heap-allocated result
		// parameters that need to be copied back to their stack slots.
		for _, f := range s.curfn.Type().Results().FieldSlice() {
			if !f.Nname.(*ir.Name).OnStack() {
				s.hasOpenDefers = false
				break
			}
		}
	}
	if s.hasOpenDefers &&
		s.curfn.NumReturns*s.curfn.NumDefers > 15 {
		// Since we are generating defer calls at every exit for
		// open-coded defers, skip doing open-coded defers if there are
		// too many returns (especially if there are multiple defers).
		// Open-coded defers are most important for improving performance
		// for smaller functions (which don't have many returns).
		s.hasOpenDefers = false
	}

	s.sp = s.entryNewValue0(ssa.OpSP, types.Types[types.TUINTPTR]) // TODO: use generic pointer type (unsafe.Pointer?) instead
	s.sb = s.entryNewValue0(ssa.OpSB, types.Types[types.TUINTPTR])

	s.startBlock(s.f.Entry)
	s.vars[memVar] = s.startmem
	if s.hasOpenDefers {
		// Create the deferBits variable and stack slot.  deferBits is a
		// bitmask showing which of the open-coded defers in this function
		// have been activated.
		deferBitsTemp := typecheck.TempAt(src.NoXPos, s.curfn, types.Types[types.TUINT8])
		deferBitsTemp.SetAddrtaken(true)
		s.deferBitsTemp = deferBitsTemp
		// For this value, AuxInt is initialized to zero by default
		startDeferBits := s.entryNewValue0(ssa.OpConst8, types.Types[types.TUINT8])
		s.vars[deferBitsVar] = startDeferBits
		s.deferBitsAddr = s.addr(deferBitsTemp)
		s.store(types.Types[types.TUINT8], s.deferBitsAddr, startDeferBits)
		// Make sure that the deferBits stack slot is kept alive (for use
		// by panics) and stores to deferBits are not eliminated, even if
		// all checking code on deferBits in the function exit can be
		// eliminated, because the defer statements were all
		// unconditional.
		s.vars[memVar] = s.newValue1Apos(ssa.OpVarLive, types.TypeMem, deferBitsTemp, s.mem(), false)
	}

	var params *abi.ABIParamResultInfo
	params = s.f.ABISelf.ABIAnalyze(fn.Type(), true)

	// Generate addresses of local declarations
	s.decladdrs = map[*ir.Name]*ssa.Value{}
	for _, n := range fn.Dcl {
		switch n.Class {
		case ir.PPARAM:
			// Be aware that blank and unnamed input parameters will not appear here, but do appear in the type
			s.decladdrs[n] = s.entryNewValue2A(ssa.OpLocalAddr, types.NewPtr(n.Type()), n, s.sp, s.startmem)
		case ir.PPARAMOUT:
			s.decladdrs[n] = s.entryNewValue2A(ssa.OpLocalAddr, types.NewPtr(n.Type()), n, s.sp, s.startmem)
		case ir.PAUTO:
			// processed at each use, to prevent Addr coming
			// before the decl.
		default:
			s.Fatalf("local variable with class %v unimplemented", n.Class)
		}
	}

	s.f.OwnAux = ssa.OwnAuxCall(fn.LSym, params)

	// Populate SSAable arguments.
	for _, n := range fn.Dcl {
		if n.Class == ir.PPARAM {
			if s.canSSA(n) {
				v := s.newValue0A(ssa.OpArg, n.Type(), n)
				s.vars[n] = v
				s.addNamedValue(n, v) // This helps with debugging information, not needed for compilation itself.
			} else { // address was taken AND/OR too large for SSA
				paramAssignment := ssa.ParamAssignmentForArgName(s.f, n)
				if len(paramAssignment.Registers) > 0 {
					if TypeOK(n.Type()) { // SSA-able type, so address was taken -- receive value in OpArg, DO NOT bind to var, store immediately to memory.
						v := s.newValue0A(ssa.OpArg, n.Type(), n)
						s.store(n.Type(), s.decladdrs[n], v)
					} else { // Too big for SSA.
						// Brute force, and early, do a bunch of stores from registers
						// TODO fix the nasty storeArgOrLoad recursion in ssa/expand_calls.go so this Just Works with store of a big Arg.
						s.storeParameterRegsToStack(s.f.ABISelf, paramAssignment, n, s.decladdrs[n], false)
					}
				}
			}
		}
	}

	// Populate closure variables.
	if !fn.ClosureCalled() {
		clo := s.entryNewValue0(ssa.OpGetClosurePtr, s.f.Config.Types.BytePtr)
		offset := int64(types.PtrSize) // PtrSize to skip past function entry PC field
		for _, n := range fn.ClosureVars {
			typ := n.Type()
			if !n.Byval() {
				typ = types.NewPtr(typ)
			}

			offset = types.Rnd(offset, typ.Alignment())
			ptr := s.newValue1I(ssa.OpOffPtr, types.NewPtr(typ), offset, clo)
			offset += typ.Size()

			// If n is a small variable captured by value, promote
			// it to PAUTO so it can be converted to SSA.
			//
			// Note: While we never capture a variable by value if
			// the user took its address, we may have generated
			// runtime calls that did (#43701). Since we don't
			// convert Addrtaken variables to SSA anyway, no point
			// in promoting them either.
			if n.Byval() && !n.Addrtaken() && TypeOK(n.Type()) {
				n.Class = ir.PAUTO
				fn.Dcl = append(fn.Dcl, n)
				s.assign(n, s.load(n.Type(), ptr), false, 0)
				continue
			}

			if !n.Byval() {
				ptr = s.load(typ, ptr)
			}
			s.setHeapaddr(fn.Pos(), n, ptr)
		}
	}

	// Convert the AST-based IR to the SSA-based IR
	s.stmtList(fn.Enter)
	s.zeroResults()
	s.paramsToHeap()
	s.stmtList(fn.Body)

	// fallthrough to exit
	if s.curBlock != nil {
		s.pushLine(fn.Endlineno)
		s.exit()
		s.popLine()
	}

	for _, b := range s.f.Blocks {
		if b.Pos != src.NoXPos {
			s.updateUnsetPredPos(b)
		}
	}

	s.f.HTMLWriter.WritePhase("before insert phis", "before insert phis")

	s.insertPhis()

	// Main call to ssa package to compile function
	ssa.Compile(s.f)

	if s.hasOpenDefers {
		s.emitOpenDeferInfo()
	}

	// Record incoming parameter spill information for morestack calls emitted in the assembler.
	// This is done here, using all the parameters (used, partially used, and unused) because
	// it mimics the behavior of the former ABI (everything stored) and because it's not 100%
	// clear if naming conventions are respected in autogenerated code.
	// TODO figure out exactly what's unused, don't spill it. Make liveness fine-grained, also.
	// TODO non-amd64 architectures have link registers etc that may require adjustment here.
	for _, p := range params.InParams() {
		typs, offs := p.RegisterTypesAndOffsets()
		for i, t := range typs {
			o := offs[i]                // offset within parameter
			fo := p.FrameOffset(params) // offset of parameter in frame
			reg := ssa.ObjRegForAbiReg(p.Registers[i], s.f.Config)
			s.f.RegArgs = append(s.f.RegArgs, ssa.Spill{Reg: reg, Offset: fo + o, Type: t})
		}
	}

	return s.f
}

type state struct {
	// configuration (arch) information
	config *ssa.Config

	// function we're building
	f *ssa.Func

	// Node for function
	curfn *ir.Func

	// labels in f
	labels map[string]*ssaLabel

	// unlabeled break and continue statement tracking
	breakTo    *ssa.Block // current target for plain break statement
	continueTo *ssa.Block // current target for plain continue statement

	// current location where we're interpreting the AST
	curBlock *ssa.Block

	// variable assignments in the current block (map from variable symbol to ssa value)
	// *Node is the unique identifier (an ONAME Node) for the variable.
	// TODO: keep a single varnum map, then make all of these maps slices instead?
	vars map[ir.Node]*ssa.Value

	// fwdVars are variables that are used before they are defined in the current block.
	// This map exists just to coalesce multiple references into a single FwdRef op.
	// *Node is the unique identifier (an ONAME Node) for the variable.
	fwdVars map[ir.Node]*ssa.Value

	// all defined variables at the end of each block. Indexed by block ID.
	defvars []map[ir.Node]*ssa.Value

	// addresses of PPARAM and PPARAMOUT variables on the stack.
	decladdrs map[*ir.Name]*ssa.Value

	// starting values. Memory, stack pointer, and globals pointer
	startmem *ssa.Value
	sp       *ssa.Value
	sb       *ssa.Value
	// value representing address of where deferBits autotmp is stored
	deferBitsAddr *ssa.Value
	deferBitsTemp *ir.Name

	// line number stack. The current line number is top of stack
	line []src.XPos
	// the last line number processed; it may have been popped
	lastPos src.XPos

	// list of panic calls by function name and line number.
	// Used to deduplicate panic calls.
	panics map[funcLine]*ssa.Block

	cgoUnsafeArgs bool
	hasdefer      bool // 是否有defer语句
	softFloat     bool
	hasOpenDefers bool // defer语句是否可以通过open coded 优化(原始defer的开销相对普通函数大,于是有了这个优化选项)

	// If doing open-coded defers, list of info about the defer calls in
	// scanning order. Hence, at exit we should run these defers in reverse
	// order of this list
	openDefers []*openDeferInfo
	// For open-coded defers, this is the beginning and end blocks of the last
	// defer exit code that we have generated so far. We use these to share
	// code between exits if the shareDeferExits option (disabled by default)
	// is on.
	lastDeferExit       *ssa.Block // Entry block of last defer exit code we generated
	lastDeferFinalBlock *ssa.Block // Final block of last defer exit code we generated
	lastDeferCount      int        // Number of defers encountered at that point

	prevCall *ssa.Value // the previous call; use this to tie results to the call op.
}
```