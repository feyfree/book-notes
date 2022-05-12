# Basic Data Types

| 类型            | 备注     | 例子                                             |
| --------------- | -------- | ------------------------------------------------ |
| Basic types     | 基础类型 | 比如numbers， strings， booleans等等             |
| Aggregate types | 聚合类型 | 数组array， 结构体 struct， 或者是其他的组合类型 |
| Refrence types  | 引用类型 | 指针， 切片，maps， functions， channels         |
| Interface Types | 接口类型 |                                                  |

**关于引用类型**

they all refer to program variables or state in directly, so that the effect of an operation applied to one reference is observed by all copies of that reference

都是非直接引用程序中的变量或者状态， 所以对一个引用的修改， 会导致所有引用上面都能看到这个修改。

从表格中可以看到 数组是数组， 切片是切片， 实际是两种类型

## 1. Integers

无符号和有符号

int8 int16 int32 int64

对应

uint8 uint16 uint32 uint64



1. int 是最广泛使用的类型
2. rune 是 int32 的同义词， 字面value是通过 unicode 编码

2. byte 是 uint8 的同义词 ， 不会具体展示为 数字， 而是展示原始的数据
3. uintptr  底层编程会使用到， width 不确定， 但是能足够持有一个指针变量的位数



**注意类型之前的区别， 记住不是同一个类型**

1. 二进制数在内存中是以补码的形式存放的。
2. 补码首位是符号位，0表示此数为正数，1表示此数为负数。
3. 正数的补码、反码，都是其本身。
4. 负数的反码是：符号位为1，其余各位求反，但末位不加1 。
5. 负数的补码是：符号位不变，其余各位求反，末位加1 。
6. 所有的取反操作、加1、减1操作，都在有效位进行。



**位操作**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220510152216-go-bit-operations.png)

**Some Demo**

```go
package numbers

import (
	"fmt"
	"testing"
)

func TestUInt(t *testing.T) {
	var u uint8 = 255
	fmt.Println(u, u+1, u*u)
}

// result -- (注意这个是 int 是有符号位的)
// 127 -128 1
func TestInt8(t *testing.T) {
	var i int8 = 127
	fmt.Println(i, i+1, i*i)
}

func TestBitOperation(t *testing.T) {
	var x uint8 = 1<<1 | 1<<5
	var y uint8 = 1<<1 | 1<<2
	fmt.Printf("%08b\n", x)    // "00100010", the set {1, 5}
	fmt.Printf("%08b\n", y)    // "00000110", the set {1, 2}
	fmt.Printf("%08b\n", x&y)  // "00000010", the intersection {1}
	fmt.Printf("%08b\n", x|y)  // "00100110", the union {1, 2, 5}
	fmt.Printf("%08b\n", x^y)  // "00100100", the symmetric difference {2, 5}
	fmt.Printf("%08b\n", x&^y) // "00100000", the difference {5}
	for i := uint(0); i < 8; i++ {
		if x&(1<<i) != 0 { // membership test
			fmt.Println(i) // "1", "5"
		}
	}
	fmt.Printf("%08b\n", x<<1) // "01000100", the set {2, 6}
	fmt.Printf("%08b\n", x>>1) // "00010001", the set {0, 4}
}

func TestBitClear(t *testing.T) {
	var x uint8 = 2<<1 | 1<<5
	var y uint8 = 1<<1 | 1<<3
	fmt.Printf("%08b\n", x) // "00100100", the set {2, 5}
	fmt.Printf("%08b\n", y) // "00001010", the set {1, 3}
	// bit clear means： remove the bit both in (x, y) from x, return the value
	fmt.Printf("%08b\n", x&^y) // "00100100", the difference {2, 5}
}

func TestLoopVariable(t *testing.T) {
	medals := []string{"gold", "silver", "bronze"}
	for i := len(medals) - 1; i >= 0; i-- {
		fmt.Println(medals[i]) // "bronze", "silver", "gold"
	}
}

func TestShortVariable(t *testing.T) {
	// 10进制默认不为 0
	a := 10
	// 8 进制默认首位是 0
	b := 06
	// 16 进制默认首位是0x
	c := 0xff
	fmt.Printf("%d  %b\n", a, a)
	fmt.Printf("%o  %b\n", b, b)

	// %[1]o 表示还是使用的是 b
	// %#[1]o 表示还是使用的是 b, 并且添加对应的默认前缀 (16 进制对应可能是 %x 或者是 %X -> 对应是 0x 或者是 0X)
	fmt.Printf("%d %[1]o %#[1]o\n", b)

	fmt.Printf("%x  %b\n", c, c)

}

// Usually a Printf format string containing multiple % verbs
// would require the same number of extra operands, but the [1] ‘‘adverbs’’ after % tell Printf to
// use the first operand over and over again. Second, the # adverb for %o or %x or %X tells Printf
// to emit a 0 or 0x or 0X prefix respectively.
func TestFormat1(t *testing.T) {
	o := 0666
	fmt.Printf("%d %[1]o %#[1]o\n", o) // "438 666 0666"
	x := int64(0xdeadbeef)
	fmt.Printf("%d %[1]x %#[1]x %#[1]X\n", x)
	// Output:
	// 3735928559 deadbeef 0xdeadbeef 0XDEADBEEF
}

func TestFormat2(t *testing.T) {
	ascii := 'a'
	unicode := '国'
	newline := '\n'
	fmt.Printf("%d %[1]c %[1]q\n", ascii)   // "97 a 'a'"
	fmt.Printf("%d %[1]c %[1]q\n", unicode) // "22269 国 '国'"
	fmt.Printf("%d %[1]q\n", newline)       // "10 '\n'"
}

```

## 2. Floating-Point Numbers

```go
package numbers

import (
	"fmt"
	"math"
	"reflect"
	"testing"
)

func TestType0(t *testing.T) {
	const e = 2.71828
	fmt.Println(reflect.TypeOf(e)) // float64

	var f float32 = 16777216 // 1 << 24
	// ide 判定是false, 但是实际上是相等的 float32 可能存在溢出
	fmt.Println(f == f+1)
}

func TestFormat(t *testing.T) {
	for x := 0; x < 8; x++ {
		fmt.Printf("x = %d ex = %8.3f\n", x, math.Exp(float64(x)))
	}
}
```

浮点数不能用 相等去判定 ！！

## 3. Complex Numbers

```go
package numbers

import (
	"fmt"
	"testing"
)

func TestDemo(t *testing.T) {
	var x complex128 = complex(1, 2) // 1+2i
	var y complex128 = complex(3, 4) // 3+4i
	fmt.Println(x * y)               // "(-5+10i)"
	fmt.Println(real(x * y))         // "-5"
	fmt.Println(imag(x * y))         // "10"
}
```

real 和 imag 是 complex 的内置函数，也是在 keyword 中的

## 4. Boolean 

skip

## 5. Strings

认清现实

1. 我们编码的过程中， 我们写的字符串都是我们可阅读的
2. text strings 都会转译成为 unicode 点位编码的形式
3. string 是 immutable 的， 意味着无法修改字符串中的某一个位的数据

### 5.1 字符串常识

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220511153225-golang-strings-demo.png)

**escape sequences**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220511153358-go-escape-sequences.png)

```go
func TestEscape(t *testing.T) {
	fmt.Println(`\n`)      // \n 注意这个是raw string
	fmt.Println("\x61123") // a123  这种是16 进制 (最大 hh)
	fmt.Println("\101")    // a 这种是8进制 (377 最大)
}
```

### 5.2 Unicode

ASCII 编码,  用 7 位来表示 128 个常见字符。 显然对于不同种类的语言编程来说， 这个明显不够用

The answer is Unicode (unicode.org), which collects all of the characters in all of the world’s writing systems, plus accents and other diacritical marks, control codes like tab and carriage return, and plenty of esoterica, and assigns each one a standard number called a Unicode code point or, in Go terminology, a rune.

rune 实际上就是 int32 

```go
// byte is an alias for uint8 and is equivalent to uint8 in all ways. It is
// used, by convention, to distinguish byte values from 8-bit unsigned
// integer values.
type byte = uint8

// rune is an alias for int32 and is equivalent to int32 in all ways. It is
// used, by convention, to distinguish character values from integer values.
type rune = int32

// iota is a predeclared identifier representing the untyped integer ordinal
// number of the current const specification in a (usually parenthesized)
// const declaration. It is zero-indexed.
const iota = 0 // Untyped int.
```

```go
func TestRuneAndInt32(t *testing.T) {
	var a rune = 'a'
	fmt.Println(reflect.TypeOf(a))
}

// === RUN   TestRuneAndInt32
// int32
// --- PASS: TestRuneAndInt32 (0.00s)
```

### 5.3 UTF-8

了解一下UTF-8的几个特点

1. UTF-8 is a variable-length encoding of Unicode code points as bytes.
2. 1 - 4 个bytes， 对于是ASCII 码的话， 只用 1 byte， 大多数的 runes 用的都是2，3 个byte
3. 对于 1 字节 的字符，字节的第一位设为 0 ，后面 7 位为这个符号的 unicode 码。
4. 对于 n 字节 的字符 (n > 1)，第一个字节的前 n 位都设为1，第 n+1 位设为 0 ，后面字节的前两位一律设为 10 。剩下的没有提及的二进制位，全部为这个符号的 unicode 码。

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220511183247-utf-8-details.png)

```go
func TestRuneEscape(t *testing.T) {
	fmt.Println("世界")                       // 世界
	fmt.Println("\xe4\xb8\x96\xe7\x95\x8c") // 世界
	fmt.Println("\u4e16\u754c")             // 世界
	fmt.Println("\U00004e16\U0000754c")     // 世界
}
```

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220511184200-go-string-range-loop-utf8.png)

```go
func TestRune(t *testing.T) {
	s := "Hello, 世界"
	fmt.Println(len(s))                    // "13"
	fmt.Println(utf8.RuneCountInString(s)) // "9"

	for i := 0; i < len(s); {
		r, size := utf8.DecodeRuneInString(s[i:])
		fmt.Printf("%d\t%c\n", i, r)
		i += size
	}

	for i, r := range s {
		fmt.Printf("%d\t%q\t%d\n", i, r, r)
	}

	n := 0
	for _, _ = range s {
		n++
	}
	fmt.Println("loop times:", n) // 9

	n = 0
	// 相当于 n = utf8.RuneCountInString(s)
	for range s {
		n++
	}
	fmt.Println("loop times:", n) // 9
}
```

len(i) 实际是 byte 的长度

for range 实际循环次数 是 string 里面 rune 的长度

```go
func TestRuneAndString(t *testing.T) {
	// "program" in Japanese katakana
	s := "プログラム"
	// % x 中间有个空格， 表示每个 digits 中间有个空格
	fmt.Printf("% x\n", s) // "e3 83 97 e3 83 ad e3 82 b0 e3 83 a9 e3 83 a0"
	r := []rune(s)
	fmt.Printf("%x\n", r) // "[30d7 30ed 30b0 30e9 30e0]"

	fmt.Println(string(r))

	// Conversion from int to string interprets an integer value as a code point
	// Inspection info: Reports conversions of the form string(x) where x is an integer (but not byte or rune) type.
	// ide 会飘黄
	//fmt.Println(string(65))     // "A", not "65"
	//fmt.Println(string(0x4eac)) // "C"

	fmt.Println(string(rune(65)))     // "A", not "65"
	fmt.Println(string(rune(0x4eac))) // "京"

	// 无效rune 这地方会报compiler failure
	//fmt.Println(string(1234567))
}
```

不能通过 string(int) 进行 int 数据的转换 ！！！

### 5.4 Strings and Byte Slices

对于string 的操作， go 内置了 四大包

* bytes
* strings
* strconv
* unicode

各有特点

1. strings 是 immutable 的， 所以程序里面会有很多 string 对象， 还有拷贝的过程。使用bytes.Buffer 类型可能更为高效
2. strconv 包封装了一些 类型的转换
3. unicode 包提供了一些 字面类型的判断 比如 IsDigit, IsLetter，IsUpper 等等

### 5.5 Conversions between Strings and Numbers

```go
func TestConversion0(t *testing.T) {
	x := 123
	y := fmt.Sprintf("%d", x)
	fmt.Println(y, strconv.Itoa(x)) // "123 123"

	fmt.Println(strconv.FormatInt(int64(x), 2)) // "1111011"

	s := fmt.Sprintf("x=%b", x) // "x=1111011
	fmt.Println(s)

	x, err := strconv.Atoi("123") // x is an int
	if err == nil {
		fmt.Println(x)
	}
	z, err := strconv.ParseInt("123", 10, 64) // base 10, up to 64 bits
	if err == nil {
		fmt.Println(z)
	}
}
```

## 6. Constants

```go
package constants

import (
	"fmt"
	"math"
	"reflect"
	"testing"
	"time"
)

const noDelay time.Duration = 0
const timeout = 5 * time.Minute

func TestConstants0(t *testing.T) {
	fmt.Printf("%T %[1]v\n", noDelay)     // "time.Duration 0"
	fmt.Printf("%T %[1]v\n", timeout)     // "time.Duration 5m0s
	fmt.Printf("%T %[1]v\n", time.Minute) // "time.Duration 1m0s"
}

const (
	a = 1
	b
	c = 2
	d
)

func TestConstants1(t *testing.T) {
	fmt.Println(a, b, c, d) // "1 1 2 2"
}

type Weekday int

const (
	Sunday Weekday = iota
	Monday
	Tuesday
	Wednesday
	Thursday
	Friday
	Saturday
)

func TestConstants2(t *testing.T) {
	fmt.Println(Sunday, Monday, Tuesday, Wednesday, Thursday, Friday, Saturday) // 0 1 2 3 4 5 6
}

type Flags uint

const (
	FlagUp           Flags = 1 << iota // is up
	FlagBroadcast                      // supports broadcast access capability
	FlagLoopBack                       // is a loopback interface
	FlagPointToPoint                   // belongs to a point-to-point link
	FlagMulticast                      // supports multicast access capability
)

func IsUp(v Flags) bool     { return v&FlagUp == FlagUp }
func TurnDown(v *Flags)     { *v &^= FlagUp }
func SetBroadcast(v *Flags) { *v |= FlagBroadcast }
func IsCast(v Flags) bool   { return v&(FlagBroadcast|FlagMulticast) != 0 }

func TestFlags(t *testing.T) {
	fmt.Println(FlagUp, FlagBroadcast, FlagLoopBack, FlagPointToPoint, FlagMulticast)
	var v Flags = FlagMulticast | FlagUp
	fmt.Printf("%b %t\n", v, IsUp(v)) // "10001 true"
	TurnDown(&v)
	fmt.Printf("%b %t\n", v, IsUp(v)) // "10000 false"
	SetBroadcast(&v)
	fmt.Printf("%b %t\n", v, IsUp(v))   // "10010 false"
	fmt.Printf("%b %t\n", v, IsCast(v)) // "10010 true"
}

const (
	_   = 1 << (10 * iota)
	KiB // 1024
	MiB // 1048576
	GiB // 1073741824
	TiB // 1099511627776 (exceeds 1 << 32)
	PiB // 1125899906842624
	EiB // 1152921504606846976
	ZiB // 1180591620717411303424 (exceeds 1 << 64)
	YiB // 1208925819614629174706176
)

func TestPower(t *testing.T) {
	// debug 的时候可以发现这几个常量都是 untyped int 类型
	fmt.Println(KiB, MiB, GiB, TiB, PiB)
	fmt.Println(YiB / ZiB)
}

// only constants can be untyped
// 一旦被复制给变量，就会有确切的类型
func TestUntypedConstants(t *testing.T) {
	var x float32 = math.Pi
	var y float64 = math.Pi
	var z complex128 = math.Pi
	fmt.Println(x, y, z)
	fmt.Println(math.Pi)

	// reflect.TypeOf 是无法反应untyped constants 的
	fmt.Println(reflect.TypeOf(math.Pi))
	var f float64 = 3 + 0i // untyped complex -> float64
	fmt.Println(reflect.TypeOf(f))
	f = 2 // untyped integer -> float64
	fmt.Println(reflect.TypeOf(f))
	f = 1e123 // untyped floating-point -> float64
	fmt.Println(reflect.TypeOf(f))
	f = 'a' // untyped rune -> float64
	fmt.Println(reflect.TypeOf(f))

}

func TestOperands(t *testing.T) {
	var f float64 = 212
	fmt.Println((f - 32) * 5 / 9)     // "100"; (f - 32) * 5 is a float64
	fmt.Println(5 / 9 * (f - 32))     // "0"; 5/9 is an untyped integer, 0
	fmt.Println(5.0 / 9.0 * (f - 32)) // "100"; 5.0/9.0 is an untyped float
}

func TestImplicit(t *testing.T) {
	i := 0      // untyped integer; implicit int(0)
	r := '\000' // untyped rune; implicit rune('\000')
	f := 0.0    // untyped floating-point; implicit float64(0.0)
	c := 0i     // untyped complex; implicit complex128(0i)
	fmt.Println(i, r, f, c)
}

func TestExplicitType(t *testing.T) {
	// short variable 依赖 go compiler
	// 指明类型的话还是主动显示声明类型
	var i = int8(0)
	// 或者
	// var i int8 = 0
	fmt.Printf("%T\n", i)

	fmt.Printf("%T\n", 0)      // "int"
	fmt.Printf("%T\n", 0.0)    // "float64"
	fmt.Printf("%T\n", 0i)     // "complex128"
	fmt.Printf("%T\n", '\000') // "int32" (rune)

}
```

1. constants 可以通过 iota 进行创建
2. 许多constants 实际上在go compiler 运行的时候实际是 untyped 类型， 注意只有constant 可以是 untyped 类型， 一旦constant 赋给具体某个变量， 那个变量对应的数据就不是 untyped 类型， reflect.TypeOf 是无法反映 一个constant 的 untyped 类型， 都是会返回默认的类型的
3. constant 是无法 取址的， 就是不能用 & 符号取址

对于第三点

[参考stackoverflow的回答](https://stackoverflow.com/questions/35146286/find-address-of-constant-in-go)

Constants are **not** listed as *addressable*, and things that are not listed in the spec as *addressable* (quoted above) **cannot** be the operand of the address operator `&` (you can't take the address of them).

It is not allowed to take the address of a constant. This is for 2 reasons:

1. A constant may not have an address at all.
2. And even if a constant value is stored in memory at runtime, this is to help the runtime to keep constants that: *constant*. If you could take the address of a constant value, you could assign the address (pointer) to a variable and you could change that (the pointed value, the value of the constant). Robert Griesemer (one of Go's authors) wrote why it's not allowed to take a string literal's address: *"If you could take the address of a string constant, you could call a function [that assigns to the pointed value resulting in] possibly strange effects - you certainly wouldn't want the literal string constant to change."* ([source](https://groups.google.com/d/msg/golang-nuts/mKJbGRRJm7c/K3k3x6-huw4J))

```go
/*
const (
	deadbeef = 0xdeadbeef        // untyped int with value 3735928559
	aa       = uint32(deadbeef)  // uint32 with value 3735928559
	bb       = float32(deadbeef) // float32 with value 3735928576 (rounded up)
	cc       = float64(deadbeef) // float64 with value 3735928559 (exact)
	dd       = int32(deadbeef)   // compile error: constant overflows int32
	ee       = float64(1e309)    // compile error: constant overflows float64
	ff       = uint(-1)          // compile error: constant underflows uint
)
*/
```

