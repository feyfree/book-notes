# Function

function 需要注意的是 

1. function 实际也是 golang 的一种类型
2. 匿名函数和闭包之间的关系， 匿名函数可以获取到闭包函数的variables
3. defer 中的匿名函数可以获取到闭包函数的results
4. for - loop 循环， range 前面指针变量是会跟着loop变的， 相当于不断的换地址， 所以 for -loop 里面如果有 匿名函数捕捉这个变量的话， 尽量定义内部变量 和将当前那个指针地址挂钩。
5. defer 后面跟着一个函数的时候（设为A）， 注意 这个函数可能是另外一个函数（设为B）的返回值， B 会执行的， 然后返回这个A， 只是最后会调用这个A， 通过 A()
6. panic 函数可以自定义抛出对象， 并且可以通过recover 捕获，或者是 defer recover 捕获
7. 函数可以返回多个值， 函数可以接受可变长度的入参 （同一种类型可变长度的话， slice 对象作为入参的话， 需要unpack， 即后面添加 `...`）

```go
package ch05

import (
	"fmt"
	"log"
	"testing"
	"time"
)

func add(x int, y int) int {
	return x + y
}

func TestFunctionType(t *testing.T) {
	fmt.Printf("%T \n", add)
}

func TestIterationUnexpected(t *testing.T) {
	var data = []string{"a", "b", "c"}
	var funcs []func()
	// 这地方相当于是
	//d 指向 data[0]
	//d 指向 data[1]
	//d 指向 data[2]
	// 最后for 循环, d 这个指针相当于是指向了 data[2]的地方
	for _, d := range data {
		funcs = append(funcs, func() { fmt.Println(d) })
	}
	for i := 0; i < len(funcs); i++ {
		funcs[i]()
	} // c c c
}

func TestIterationExpected(t *testing.T) {
	var data = []string{"a", "b", "c"}
	var funcs []func()
	for _, d := range data {
		// d 我们把它认为是 outer
		// 通过创建 inner 捕获每次outer 指向的数据
		inner := d
		funcs = append(funcs, func() { fmt.Println(inner) })
	}
	for i := 0; i < len(funcs); i++ {
		funcs[i]()
	} // a b c
}

func sum(vals ...int) int {
	total := 0
	for _, val := range vals {
		total += val
	}
	return total
}

func f(...int) {}
func g([]int)  {}

func TestVariadic(t *testing.T) {
	fmt.Println(sum())           // "0"
	fmt.Println(sum(3))          // "3"
	fmt.Println(sum(1, 2, 3, 4)) // "10"

	values := []int{1, 2, 3, 4}
	// unpack slice
	fmt.Println(sum(values...)) // "10"

	fmt.Printf("%T\n", f) // "func(...int)"
	fmt.Printf("%T\n", g) // "func([]int)"

}

//!+main
func bigSlowOperation() {
	defer trace("bigSlowOperation")() // don't forget the extra parentheses
	// ...lots of work...
	time.Sleep(10 * time.Second) // simulate slow operation by sleeping
}

// Because an anonymous function can access its enclosing function’s variables, including named
// results, a deferred anonymous function can observe the function’s results.
func trace(msg string) func() {
	start := time.Now()
	log.Printf("enter %s", msg)
	return func() { log.Printf("exit %s (%s)", msg, time.Since(start)) }
}

//!-main

func TestTrace(t *testing.T) {
	bigSlowOperation()
}

func double(x int) (result int) {
	defer func() { fmt.Printf("double(%d) = %d\n", x, result) }()
	return x + x
}

func TestDouble(t *testing.T) {
	_ = double(4)
}

func TestDeferLoop1(t *testing.T) {
	var data = []string{"a", "b", "c"}
	for _, val := range data {
		defer func() { fmt.Println(val) }()
	} // c c c
}

func TestDeferLoop2(t *testing.T) {
	var data = []string{"a", "b", "c"}
	for _, val := range data {
		defer fmt.Println(val)
	} // c b a
}

func TestDeferLoop3(t *testing.T) {
	var data = []string{"a", "b", "c"}
	for _, val := range data {
		myPrint(val)
	} // a b c
}

func myPrint(val string) {
	defer fmt.Println(val)
}

func TestDefer1(t *testing.T) {
	// 保持函数签名一致
	var function func(x int)
	function = func(x int) {
		fmt.Printf("f(%d)\n", x+0/x) // panics if x == 0
		defer fmt.Printf("defer %d\n", x)
		function(x - 1)
	}
	function(3)
}

func TestPanic(t *testing.T) {
	type panicData struct {
	}
	defer func() {
		switch p := recover(); p {
		case nil:
			fmt.Println("OK")
		case panicData{}:
			fmt.Println("my panic data")
		default:
			panic(p)
		}
	}()
	panic(panicData{})
}

```

