## 版本
+ Github:golang/go
+ 分支:release-branch.go.1.17

## 序
bufio用来对一个io做一层封装,引入buffer,减少对嵌入io的直接读写的次数

## Reader
### 数据结构
```go
// bufio.go
type Reader struct {
	buf          []byte // buffer
	rd           io.Reader // 要封装和嵌入的Reader, 比如原生io.Reader
	r, w         int       // buffer当前的的读写位置,把从rd读的内容写到‘w’位置,外部从‘r’位置读,
	err          error
	//...
}
```

### 初始化
调用方可以指定原始要封装的Reader 和 buffer的大小
```go
// bufio.go
func NewReader(rd io.Reader) *Reader {
	return NewReaderSize(rd, defaultBufSize)
}
func NewReaderSize(rd io.Reader, size int) *Reader {
	// ...
	r := new(Reader)
	r.reset(make([]byte, size), rd) //新建一个指定大小的数组用来buffer,
	return r
}

func (b *Reader) reset(buf []byte, r io.Reader) {
	*b = Reader{
		buf:          buf,
		rd:           r,
        // ...
	}
}
```

### 读
```go
// bufio.go 
// 读取内容到参数传来的地址p中
func (b *Reader) Read(p []byte) (n int, err error) {
	n = len(p) // 目标地址最多能读多少内容
	if b.r == b.w { //如果buffer的读写地址相等,说明buffer空了
        // 当我们的目标地址空间足够大时,比整个buffer都大
		if len(p) >= len(b.buf) {
			// 直接从嵌入的Reader读内容,
			n, b.err = b.rd.Read(p)
            // ...
			return n, b.readErr()
		}
		//buffer的读写位置清0
		b.r = 0
		b.w = 0
		n, b.err = b.rd.Read(b.buf) // 调用嵌入Reader的方法再进一批数据
		b.w += n // 更新buffer的写位置
	}

	n = copy(p, b.buf[b.r:b.w]) // 将buffer的内容读取到目标位置 (用户层copy)
	b.r += n // 更新buffer的读位置
	return n, nil
}
```
## Writer
### 数据结构
```go
// bufio.go
type Writer struct {
	err error
	buf []byte // buffer
	n   int 
	wr  io.Writer // 要封装嵌入的原Writer
}

```
### 初始化
```go
// bufio.go
func NewWriter(w io.Writer) *Writer {
	return NewWriterSize(w, defaultBufSize)
}
func NewWriterSize(w io.Writer, size int) *Writer {

	return &Writer{
		buf: make([]byte, size), // 创建一个带下等于既定大小的数组buffer
		wr:  w, // 嵌入原Writer
	}
}
```
### 写
```go
// bufio.go
func (b *Writer) Write(p []byte) (nn int, err error) {
	for len(p) > b.Available() && b.err == nil { // 当要写的内容数量大于buffer剩余容量时,需要先flush掉buffer里已有内容
		var n int
		if b.Buffered() == 0 {
			// buffer已经清空了(却依然不能满足要写的内容容量),直接写到嵌入的Writer里
			n, b.err = b.wr.Write(p)
		} else {
            // 先写一部分把buffer填满,然后调用Flush,写入嵌入的Writer里
			n = copy(b.buf[b.n:], p)
			b.n += n
			b.Flush()
		}
        // 更新要写的内容的位置,下一个循环接着写
		nn += n
		p = p[n:]
	}

	n := copy(b.buf[b.n:], p) // 将内容写入buffer里
    // 更新buffer信息
	b.n += n
	nn += n
	return nn, nil
}

func (b *Writer) Flush() error {
    //...
    // 将已有buffer里的内容全部写入嵌入的Writer里
	if err != nil {
		if n > 0 && n < b.n {
			copy(b.buf[0:b.n-n], b.buf[n:b.n])
		}
		b.n -= n
		b.err = err
		return err
	}
	b.n = 0
	return nil
}
```

## Scaner