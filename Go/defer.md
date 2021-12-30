## 版本
+ Github:golang/go
+ 分支:release-branch.go.1.17

## 入口
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
		if len(call.KeepAlive) > 0 {
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