## 版本
+ Github:golang/go
+ 分支:release-branch.go.1.17

## 序
slice相当于一个更灵活的动态数组

## 运行时的slice函数
### `makeslice`创建slice 
```go
// runtime/slice.go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	// slice 基于数组,同样需要一份内存空间,空间大小取决于数据Type * cap
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// ...
	}
	// 申请mem大小的空间并返回地址
	return mallocgc(mem, et, true)
}
```
### `growslice`增加slice大小
当往slice里不断加入数据(`append`),直到数据量超过slice的cap值时,需要调用growslice增加slice的cap大小
```go
// runtime/slice.go
// 参数cap是增加后的新cap值大小(这只是个期望值,实际为了效率会优化生成一个比这个值大的数字)
func growslice(et *_type, old slice, cap int) slice {
	if cap < old.cap { // 新cap值不允许比旧cap值小
		panic(errorString("growslice: cap out of range"))
	}

	newcap := old.cap
	doublecap := newcap + newcap // 原cap值 * 2
	if cap > doublecap { // 如果原cap值 * 2后仍然不满足期望值,则使用期望值作为新cap值
		newcap = cap
	} else {
		if old.cap < 1024 {
			newcap = doublecap // 原cap值比较小时,直接成倍增加
		} else {
			// 原cap已经比较大了则需要在空间和效率上取个平衡
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}	
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	// 根据slice数据类型对心cap值做些有话修正,最重要生成的cap大小为capmem
	// ...
	
	// 获取一片大小为capmem的新空间
		p = mallocgc(capmem, nil, false)
	// 把以前空间内的内容移动到新空间
	memmove(p, old.array, lenmem)

	// 返回新slice数据结构
	return slice{p, old.len, newcap}
}

type slice struct {
	array unsafe.Pointer //一个指向底层数据空间的指针
	len   int // 目前用到的空间大小
	cap   int // 实际占用的空间大小
}
```

### slicecopy
调用`copy(SLICEA, SLICEB)`
```go
// runtime/slice.go
func slicecopy(toPtr unsafe.Pointer, toLen int, fromPtr unsafe.Pointer, fromLen int, width uintptr) int {
	// 源slice的element数大于目标slice的element时,需要调整copy的数量
	n := fromLen
	if toLen < n {
		n = toLen
	}
	// element数量 * element 宽度
	size := uintptr(n) * width
	// 复制源slice指针的内容到目标slice指针的位置,是真的新拷贝了一份值,而不是把指针指到了原值上
		memmove(toPtr, fromPtr, size)

	return n
}
```

### slice取值
常见的slice取值情形
```go
// compile/internal/ir/node.go
	OSLICE       // X[Low : High] (X is untypechecked or slice)
	OSLICEARR    // X[Low : High] (X is pointer to array)
	OSLICESTR    // X[Low : High] (X is string)
	OSLICE3      // X[Low : High : Max] (X is untypedchecked or slice)
	OSLICE3ARR   // X[Low : High : Max] (X is pointer to array)

	OSLICEHEADER // sliceheader{Ptr, Len, Cap} (Ptr is unsafe.Pointer, Len is length, Cap is capacity)
```

#### slice在编译阶段的表达式结构
```go
// compile/internal/ir/expr.go
type SliceExpr struct {
	miniExpr
	X    Node // slice变量名
	Low  Node // 低位
	High Node // 高位
	Max  Node // max,一般切片表达式是要赋值给另一个新切片的,这个max相当于新切片的cap值
}
```
#### 编译阶段遇到slice(或array)取值的处理
```go
// compile/internal/walk/expr.go
func walkExpr1(n ir.Node, init *ir.Nodes) ir.Node {
	switch n.Op() {
	// ...
	case ir.OSLICE, ir.OSLICEARR, ir.OSLICESTR, ir.OSLICE3, ir.OSLICE3ARR:
		n := n.(*ir.SliceExpr)
		return walkSlice(n, init)
	// ...
	}
}

func walkSlice(n *ir.SliceExpr, init *ir.Nodes) ir.Node {
	// ...
	// slice表达式的变量名,低位,高位,max的表达式解析成具体的值
		n.X = walkExpr(n.X, init)
	n.Low = walkExpr(n.Low, init)
	n.High = walkExpr(n.High, init)
	n.Max = walkExpr(n.Max, init)
	// ...
}
```

#### 运行阶段的从一个slice里取一段赋值给另一个slice的源代码 todo