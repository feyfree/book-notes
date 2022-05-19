# Interfaces

## Tips

接口类型实际上 概括或者是抽象了 一些其他类型的一些行为。通过这种概括， 我们可以通过自定义实现接口定义的这些行为， 更加灵活方便。

Interface 需要注意的地方

1. Go 的接口类型与Java 接口类型有点不同， Java 更加显式， 比如使用 implements ， 然后重载这些方法， 而 Go 是 implicit 的。 不需要通过 implements，只需要实现 接口 下面声明的方法， 就可以认为是 这种接口类型 （An interface type specififies a set of methods that a concrete type must possess to be considered an instance of that interface.
2. 理解 concrete type 和 interface type ( It doesn’t expose the representation or internal structure of its values, or the set of basic operations they supp ort; it reveals only some of their methods. 所以当你 有一个 interface 的这么一个 变量的时候， 你只知道他们能做哪些事情 （声明的方法）
3. 接口可以组合使用

```go
type Reader interface {
		Read(p []byte) (n int, err error)
}
type Closer interface {
		Close() error
}

type ReadWriter interface {
    Reader
    Writer
}
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

4. interface statisfy

```go
func TestInterfaceSatisfy(t *testing.T) {
	var w io.Writer
	w = os.Stdout
	w.Write([]byte("hello"))
	// 注意  w 是 io.Writer 类型， 没有Close方法
	// 如果调用了 w.Close() 会无法编译
	// w.Close()
}
```

5. 理解interface value

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220519112957-go-nil-interface-value.png)

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220519113215-sample-interface-value-in-go.png)

6. An interface containing a nil pointer is non-nil. 比如将一个 *bytes.Buffer 的指针传入 一个接收入参为  interface 函数的话， 实际上传入的对象是 如下图所示。 代码示例如图下面所示

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220519113414-non-nil-interface-container-nil-pointer.png)

```go
// assign the nil pointer to the interface was a mistake
// nil pointer is not nil
func TestNilPointer(t *testing.T) {
	// 这地方注意是指针变量, 最好还是用 bytes.Buffer 然后 new
	var buf *bytes.Buffer
	fmt.Printf("%T\n", buf) // *bytes.Buffer
	//buf = new(bytes.Buffer)

	// 如果不通过 new(bytes.Buffer)操作的话， 会panic
	// 相当于是 有 type 无 value
	f(buf)
}

func f(out io.Writer) {
	fmt.Printf("%T\n", out) // *bytes.Buffer
	if out != nil {
		out.Write([]byte("Hello"))
	}
}
```

7. type assertions is an operation applied to an interface value。 注意是 interface value， 任何的 concrete value 是不能用 type assertion 的

```go
// type assertion like x.(T)
// 1. 如果 T 是具体类型， type assertion 会检查 x 的 dynamic value 是不是 T， 如果是会返回 x 的 dynamic value, 否则会报错(使用二元组可以避免 panic)
// 2. 如果 T 是interface 类型, 会检查 x 的 dynamic value 会不会 satisfy T
//
func TestTypeAssertions(t *testing.T) {
	var w io.Writer
	w = os.Stdout
	_, ok := w.(*os.File)
	if ok {
		fmt.Println("w holds : os.Stdout")
	}
	// panic: interface holds *os.File, not *bytes.Buffer
	//c := w.(*bytes.Buffer)

	_, right := w.(*bytes.Buffer)
	if !right {
		fmt.Println("w holds : not bytes.Buffer")
	}

	//y := os.Stdout
	// 这样写的话会编译不通过 invalid type assertion: y.(io.Writer) (non-interface type *os.File on left)
	// Type switches require an interface to introspect.
	// If you are passing a value of known type to it it bombs out.
	// If you make a function that accepts an interface as a parameter, it will work
	//_, isWriter := y.(io.Writer)
	//fmt.Println(isWriter)
}

func TestTypeAssertions2(t *testing.T) {

	var w io.Writer
	w = os.Stdout
	_ = w.(io.ReadWriter) // success: *os.File has both Read and Write

	//w = new(bytecounter.ByteCounter)
	// panic: *ByteCounter has no Read method
	//_ = w.(io.ReadWriter)

	rw := w.(io.ReadWriter)
	fmt.Printf("%T \n", rw)
}
```

8. Interfaces are only needed when there are two or more concrete types that must be dealt with in a uniform way.  当两个以上的具体类型， 需要以同样一种形式进行处理的时候， 这时候才需要 Interface 这种类型。同样接口可以达到解耦的目的

## Other Code

```go
package ch07

import (
	"bytes"
	"fmt"
	"io"
	"os"
	"testing"
)

func TestInterfaceSatisfy(t *testing.T) {
	var w io.Writer
	w = os.Stdout
	w.Write([]byte("hello"))
	// 注意  w 是 io.Writer 类型， 没有Close方法
	// 如果调用了 w.Close() 会无法编译
	// w.Close()
}

// empty interface mean that we can assign any value to the empty interface
func TestEmptyInterface(t *testing.T) {
	var any interface{}
	any = true
	any = 12.34
	any = "hello"
	any = map[string]int{"one": 1}
	fmt.Println(any)
	any = new(bytes.Buffer)
	fmt.Println(any)
}

func TestInterface(t *testing.T) {
	// 这地方注意不是指针变量
	var w io.Writer
	fmt.Printf("%T\n", w) // "<nil>"
	// os.Stdout 是 concrete type
	// w 是 interface type
	// 这个实际上存在一个 隐含的转换
	w = os.Stdout
	w.Write([]byte("hello"))
	fmt.Printf("%T\n", w) // "*os.File"

	// 显示转换
	w = io.Writer(os.Stdout)
	w.Write([]byte("hello"))

	w = new(bytes.Buffer)
	fmt.Printf("%T\n", w) // "*bytes.Buffer"
	w.Write([]byte("hello"))

	var x interface{} = []int{1, 2, 3}
	fmt.Println(x == x) // panic: comparing uncomparable type []int
}

// assign the nil pointer to the interface was a mistake
// nil pointer is not nil
func TestNilPointer(t *testing.T) {
	// 这地方注意是指针变量, 最好还是用 bytes.Buffer 然后 new
	var buf *bytes.Buffer
	fmt.Printf("%T\n", buf) // *bytes.Buffer
	//buf = new(bytes.Buffer)

	// 如果不通过 new(bytes.Buffer)操作的话， 会panic
	// 相当于是 有 type 无 value
	f(buf)
}

func f(out io.Writer) {
	fmt.Printf("%T\n", out) // *bytes.Buffer
	if out != nil {
		out.Write([]byte("Hello"))
	}
}

// type assertion like x.(T)
// 1. 如果 T 是具体类型， type assertion 会检查 x 的 dynamic value 是不是 T， 如果是会返回 x 的 dynamic value, 否则会报错(使用二元组可以避免 panic)
// 2. 如果 T 是interface 类型, 会检查 x 的 dynamic value 会不会 satisfy T
//
func TestTypeAssertions(t *testing.T) {
	var w io.Writer
	w = os.Stdout
	_, ok := w.(*os.File)
	if ok {
		fmt.Println("w holds : os.Stdout")
	}
	// panic: interface holds *os.File, not *bytes.Buffer
	//c := w.(*bytes.Buffer)

	_, right := w.(*bytes.Buffer)
	if !right {
		fmt.Println("w holds : not bytes.Buffer")
	}

	//y := os.Stdout
	// 这样写的话会编译不通过 invalid type assertion: y.(io.Writer) (non-interface type *os.File on left)
	// Type switches require an interface to introspect.
	// If you are passing a value of known type to it it bombs out.
	// If you make a function that accepts an interface as a parameter, it will work
	//_, isWriter := y.(io.Writer)
	//fmt.Println(isWriter)
}

func TestTypeAssertions2(t *testing.T) {

	var w io.Writer
	w = os.Stdout
	_ = w.(io.ReadWriter) // success: *os.File has both Read and Write

	//w = new(bytecounter.ByteCounter)
	// panic: *ByteCounter has no Read method
	//_ = w.(io.ReadWriter)

	rw := w.(io.ReadWriter)
	fmt.Printf("%T \n", rw)
}

```

