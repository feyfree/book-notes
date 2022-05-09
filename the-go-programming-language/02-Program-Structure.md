## 1. Names

**25 Keywords**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220509135746-go-25-keywords.png)

**predeclared names**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220509135848-go-predeclared-names.png)

## 2. Declarations

 There are four major kinds of declarations: **var, const, type**, and **func**

```go
package boiling

import "fmt"

const boilingF = 212.0

func boiling() {
	var f = boilingF
	var c = (f - 32) * 5 / 9
	fmt.Printf("boiling point = %g°F or %g°C\n", f, c)
	// Output:
	// boiling point = 212°F or 100°C
}
```

## 3. Variables

```go
var name type = expression
```

1. 如果 `type`被省略的话， 则是依据字面上看的含义
2. 如果`expression`被省略的话， 使用的是 zero value,比如数字相关是 0， 布尔类型 是 false, 字符串类型是 “”， 接口或者引用类型（slice, point er, map, channel, function）是 nil

```go
var i, j, k int // int, int, int
var b, f, s = true, 2.3, "four" // bool, float64, string
```

### 3.1 short variable declarations

```go
f, err := os.Open(infile)
```

### 3.2 pointers

比如数组 x[i] , 或者是对象 x.f 实际上， 并没有明显的变量名。

指针做了这样的工作， 指针 是 地址， 是数据存储的地址

所以访问对象的数据， 可以通过变量名， 也可以通过指针

**取址操作**

```go
x := 1
p := &x // p, of type *int, points to x
fmt.Println(*p) // "1"
*p = 2 // equivalent to x = 2
fmt.Println(x) // "2"
```

指针是可以比较的，当指向同一个地址的时候， 即为相等

### 3.3 new 操作

内置函数 new ， new(T) 的实际操作是

1. 创建了一个未命名的类型T的数据
2. 初始化为 zero value
3. 并且返回数据的指针（地址）， 是有类型的， 类型为 *T

### 3.4 变量的生命周期

内循环， 函数内部的中间变量， 等等一般是在栈分配

**escape**

```go
var global *int

func f() {
	var x int
	x = 1
	global = &x
}
```

Here , x must be heap-allocated because it is still reachable from the variable global after f has returned, despite being declared as a local variable; we say **x escapes from f**

```go
func g() {
	y := new(int)
	*y = 1
}
```

而g() 函数的*y 分配在stack 也是安全的， 因为g函数结束了， *y也访问不到， 可以被回收



所以避免  生命周期短的变量 和 长期存在的变量产生连接， 这样会阻止垃圾收集器回收这段内存



## 4. Assignments

### 4.1 Tuple Assignment

```go
func fib(n int) int {
	x, y := 0, 1
	for i := 0; i < n; i++ {
		x, y = y, x+y
	}
	return x
}

func gcd(x int, y int) int {
	for y != 0 {
		x, y = y, x%y
	}
	return x
}
```

## 5. Type Declarations

```go
package tempconv0

import "fmt"

type Celsius float64
type Fahrenheit float64

const (
	AbsoluteZeroC Celsius = -273.15
	FreezingC     Celsius = 0
	BoilingC      Celsius = 100
)

func CToF(c Celsius) Fahrenheit { return Fahrenheit(c*9/5 + 32) }

func FToC(f Fahrenheit) Celsius { return Celsius((f - 32) * 5 / 9) }

//!-

func (c Celsius) String() string { return fmt.Sprintf("%g°C", c) }
```

Celsius 和 Fahrenheit 是不能做 == ， 不然会报 type miss 这种报错

## 6. Packages and Files

### 6.1 import

The language specification doesn’t define where these strings come from or what they mean;

**it’s up to the tools to interpret them**

所以 import 的路径包名， 取决于你是使用什么工具， 现在用的基本都是 go mudules

### 6.2 package initialization

```go
package popcount

// pc[i] is the population count of i.
var pc [256]byte

func init() {
	for i := range pc {
		pc[i] = pc[i/2] + byte(i&1)
	}
}

// PopCount returns the population count (number of set bits) of x.
func PopCount(x uint64) int {
	return int(pc[byte(x>>(0*8))] +
		pc[byte(x>>(1*8))] +
		pc[byte(x>>(2*8))] +
		pc[byte(x>>(3*8))] +
		pc[byte(x>>(4*8))] +
		pc[byte(x>>(5*8))] +
		pc[byte(x>>(6*8))] +
		pc[byte(x>>(7*8))])
}
```

init 函数不能被调用或者引用， 当程序运行的时候， 根据他们被使用声明的时候自动调用

## 7. Scope

```go
package scope

import (
	"fmt"
	"math/rand"
	"testing"
	"time"
)

func TestVariableScope1(t *testing.T) {
	x := "hello"
	for _, x := range x {
		x := x + 'A' - 'a'
		fmt.Printf("%c", x)
	}
}

func f() int {
	seed := rand.NewSource(time.Now().UnixNano())
	random := rand.New(seed)
	return random.Intn(100)
}

func g(x int) int {
	return x ^ x
}

func TestVariableScope2(t *testing.T) {
	if x := f(); x == 0 {
		fmt.Println("--1--")
		fmt.Println(x)
	} else if y := g(x); x == y {
		fmt.Println("--2--")
		fmt.Println(x, y)
	} else {
		fmt.Println("--3--")
		fmt.Println(x, y)
	}
	// if-else 之外实际是访问不到的
	// fmt.Println(x, y)
}
```

