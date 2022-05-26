# Low-Level Programming

## 基础知识

1. 字 （word）在 32-bit 的平台上 表示的是 4 bytes 在 64-bit 的平台上表示的是 8 bytes
2. 对齐就会有 hole， 但是对齐可以提升访问的效率 （知道从哪里算是开始）
3. Moving GC you的时候做garbage collect的时候会移动变量， 这个时候， 所有指向变量老的地址的指针， 都需要更新到指向新地址的。 

## unsafe.Pointer

```go'
import (
	"fmt"
	"unsafe"
)

func main() {
	//!+main
	var x struct {
		a bool
		b int16
		c []int
	}

	// equivalent to pb := &x.b
	pb := (*int16)(unsafe.Pointer(
		uintptr(unsafe.Pointer(&x)) + unsafe.Offsetof(x.b)))
	*pb = 42

	fmt.Println(x.b) // "42"
	//!-main
}

/*
//!+wrong
	// NOTE: subtly incorrect!
	tmp := uintptr(unsafe.Pointer(&x)) + unsafe.Offsetof(x.b)
	pb := (*int16)(unsafe.Pointer(tmp))
	*pb = 42
//!-wrong
*/
```

代码中的错误示例

因为uintptr 仅仅是一个number， 

1. 如果出现moving GC的时候， 这个number 是不会变的， 所以有可能还是修改在原来的 地址上面， 并没有对x.b进行修改
2. goroutines 的 stack 增长的时候， 可能有old stack -> new stack 这种复制分配的操作， 也有可能产生和moving GC 一样的问题

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220526111703-uintptr-pointer-tips.png)