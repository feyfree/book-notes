# Reflection

## 反射是什么

反射是指在程序运行期对程序本身进行访问和修改的能力。程序在编译时，变量被转换为内存地址，变量名不会被编译器写入到可执行部分。在运行程序时，程序无法获取自身的信息。

支持反射的语言可以在程序编译期将变量的反射信息，如字段名称、类型信息、结构体信息等整合到可执行文件中，并给程序提供接口访问反射信息，这样就可以在程序运行期获取类型的反射信息，并且有能力修改它们。

Go程序在运行期使用reflect包访问程序的反射信息。



## 为什么需要反射

有些时候我们需要写个函数， 但是入参是多种多样的， 无法以一种统一的接口形式传入。 我们**无法完全罗列**这种入参的类型 （入参是infinite）， 并且给出具体的执行逻辑。

我们需要通过反射来知道未知类型是如何表达的



## `reflect.Type`类型和值

通过调用`reflect.TypeOf`函数，我们可以从一个任何非接口类型的值创建一个`reflect.Type`值。 此`reflect.Type`值表示着此非接口值的类型。通过此值，我们可以得到很多此非接口类型的信息。 当然，我们也可以将一个接口值传递给一个`reflect.TypeOf`函数调用，但是此调用将返回一个表示着此接口值的动态类型的`reflect.Type`值。 实际上，`reflect.TypeOf`函数的唯一参数的类型为`interface{}`， `reflect.TypeOf`函数将总是返回一个表示着此唯一接口参数值的动态类型的`reflect.Type`值。 那如何得到一个表示着某个接口类型的`reflect.Type`值呢？ 我们必须通过下面将要介绍的一些间接途径来达到这一目的。

类型`reflect.Type`为一个接口类型，它指定了[若干方法](https://golang.google.cn/pkg/reflect/#Type)。 通过这些方法，我们能够观察到一个`reflect.Type`值所表示的Go类型的各种信息。 这些方法中的有些适用于[所有种类](https://golang.google.cn/pkg/reflect/#Kind)的类型，有些只适用于一种或几种类型。 通过不合适的`reflect.Type`属主值调用某个方法将在运行时产生一个恐慌。 请阅读`reflect`代码库中各个方法的文档来获取如何正确地使用这些方法。

一个例子：

```go
package main

import "fmt"
import "reflect"

func main() {
	type A = [16]int16
	var c <-chan map[A][]byte
	tc := reflect.TypeOf(c)
	fmt.Println(tc.Kind())    // chan
	fmt.Println(tc.ChanDir()) // <-chan
	tm := tc.Elem()
	ta, tb := tm.Key(), tm.Elem()
	fmt.Println(tm.Kind(), ta.Kind(), tb.Kind()) // map array slice
	tx, ty := ta.Elem(), tb.Elem()

	// byte是uint8类型的别名。
	fmt.Println(tx.Kind(), ty.Kind()) // int16 uint8
	fmt.Println(tx.Bits(), ty.Bits()) // 16 8
	fmt.Println(tx.ConvertibleTo(ty)) // true
	fmt.Println(tb.ConvertibleTo(ta)) // false

	// 切片类型和映射类型都是不可比较类型。
	fmt.Println(tb.Comparable()) // false
	fmt.Println(tm.Comparable()) // false
	fmt.Println(ta.Comparable()) // true
	fmt.Println(tc.Comparable()) // true
}
```

目前，Go支持[26种种类的类型](https://golang.google.cn/pkg/reflect/#Kind)。

在上面这个例子中，我们使用方法`Elem`来得到某些类型的元素类型。 实际上，此方法也可以用来得到一个指针类型的基类型。一个例子：

```go
package main

import "fmt"
import "reflect"

type T []interface{m()}
func (T) m() {}

func main() {
	tp := reflect.TypeOf(new(interface{}))
	tt := reflect.TypeOf(T{})
	fmt.Println(tp.Kind(), tt.Kind()) // ptr slice

	// 使用间接的方法得到表示两个接口类型的reflect.Type值。
	ti, tim := tp.Elem(), tt.Elem()
	fmt.Println(ti.Kind(), tim.Kind()) // interface interface

	fmt.Println(tt.Implements(tim))  // true
	fmt.Println(tp.Implements(tim))  // false
	fmt.Println(tim.Implements(tim)) // true

	// 所有的类型都实现了任何空接口类型。
	fmt.Println(tp.Implements(ti))  // true
	fmt.Println(tt.Implements(ti))  // true
	fmt.Println(tim.Implements(ti)) // true
	fmt.Println(ti.Implements(ti))  // true
}
```

上面这个例子同时也展示了如何通过间接的途径得到一个表示一个接口类型的`reflect.Type`值。

我们可以通过反射列出一个类型的所有方法和一个结构体类型的所有（导出和非导出）字段的类型。 我们也可以通过反射列出一个函数类型的各个输入参数和返回结果类型。

```go
package main

import "fmt"
import "reflect"

type F func(string, int) bool
func (f F) m(s string) bool {
	return f(s, 32)
}
func (f F) M() {}

type I interface{m(s string) bool; M()}

func main() {
	var x struct {
		F F
		i I
	}
	tx := reflect.TypeOf(x)
	fmt.Println(tx.Kind())        // struct
	fmt.Println(tx.NumField())    // 2
	fmt.Println(tx.Field(1).Name) // i
	// 包路径（PkgPath）是非导出字段（或者方法）的内在属性。
	fmt.Println(tx.Field(0).PkgPath) // 
	fmt.Println(tx.Field(1).PkgPath) // main

	tf, ti := tx.Field(0).Type, tx.Field(1).Type
	fmt.Println(tf.Kind())               // func
	fmt.Println(tf.IsVariadic())         // false
	fmt.Println(tf.NumIn(), tf.NumOut()) // 2 1
	t0, t1, t2 := tf.In(0), tf.In(1), tf.Out(0)
	// 下一行打印出：string int bool
	fmt.Println(t0.Kind(), t1.Kind(), t2.Kind())

	fmt.Println(tf.NumMethod(), ti.NumMethod()) // 1 2
	fmt.Println(tf.Method(0).Name)              // M
	fmt.Println(ti.Method(1).Name)              // m
	_, ok1 := tf.MethodByName("m")
	_, ok2 := ti.MethodByName("m")
	fmt.Println(ok1, ok2) // false true
}
```

从上面这个例子我们可以看出：

1. 对于非接口类型，`reflect.Type.NumMethod`方法只返回一个类型的所有导出的方法（包括通过内嵌得来的隐式方法）的个数，并且 方法`reflect.Type.MethodByName`不能用来获取一个类型的非导出方法； 而对于接口类型，则并无这些限制（Go 1.16之前的文档对这两个方法的描述不准确，并没有体现出这个差异）。 此情形同样存在于下一节将要介绍的`reflect.Value`类型上的相应方法。
2. 虽然`reflect.Type.NumField`方法返回一个结构体类型的所有字段（包括非导出字段）的数目，但是[不推荐](https://golang.org/pkg/reflect/#pkg-note-BUG)使用方法`reflect.Type.FieldByName`来获取非导出字段。

我们可以[通过反射来检视结构体字段的标签信息](https://golang.org/pkg/reflect/#StructTag)。 结构体字段标签的类型为`reflect.StructTag`，它的方法`Get`和`Lookup`用来检视字段标签中的键值对。 一个例子：

```go
package main

import "fmt"
import "reflect"

type T struct {
	X    int  `max:"99" min:"0" default:"0"`
	Y, Z bool `optional:"yes"`
}

func main() {
	t := reflect.TypeOf(T{})
	x := t.Field(0).Tag
	y := t.Field(1).Tag
	z := t.Field(2).Tag
	fmt.Println(reflect.TypeOf(x)) // reflect.StructTag
	// v的类型为string
	v, present := x.Lookup("max")     
	fmt.Println(len(v), present)      // 2 true
	fmt.Println(x.Get("max"))         // 99
	fmt.Println(x.Lookup("optional")) //  false
	fmt.Println(y.Lookup("optional")) // yes true
	fmt.Println(z.Lookup("optional")) // yes true
}
```

注意：

- 键值对中的键不能包含空格（Unicode值为32）、双引号（Unicode值为34）和冒号（Unicode值为58）。
- 为了形成键值对，所设想的键值对形式中的冒号的后面不能紧跟着空格字符。所以
  ``optional: "yes"``不形成键值对。
- 键值对中的值中的空格不会被忽略。所以
  ``json:"author, omitempty“``、
  ``json:" author,omitempty“``以及
  ``json:"author,omitempty“``各不相同。
- 每个字段标签应该呈现为单行才能使它的整个部分都对键值对的形成有贡献。

`reflect`代码包也提供了一些其它函数来动态地创建出来一些无名组合类型。

```go
package main

import "fmt"
import "reflect"

func main() {
	ta := reflect.ArrayOf(5, reflect.TypeOf(123))
	fmt.Println(ta) // [5]int
	tc := reflect.ChanOf(reflect.SendDir, ta)
	fmt.Println(tc) // chan<- [5]int
	tp := reflect.PtrTo(ta)
	fmt.Println(tp) // *[5]int
	ts := reflect.SliceOf(tp)
	fmt.Println(ts) // []*[5]int
	tm := reflect.MapOf(ta, tc)
	fmt.Println(tm) // map[[5]int]chan<- [5]int
	tf := reflect.FuncOf([]reflect.Type{ta},
				[]reflect.Type{tp, tc}, false)
	fmt.Println(tf) // func([5]int) (*[5]int, chan<- [5]int)
	tt := reflect.StructOf([]reflect.StructField{
		{Name: "Age", Type: reflect.TypeOf("abc")},
	})
	fmt.Println(tt)            // struct { Age string }
	fmt.Println(tt.NumField()) // 1
}
```



上面的例子并未展示和`reflect.Type`相关的所有函数和方法。 请阅读`reflect`标准库代码包的文档以获取如何使用这些函数和方法。

注意，到目前为止（Go 1.18），我们无法通过反射动态创建一个接口类型。这是Go反射目前的一个限制。

另一个限制是使用反射动态创建结构体类型的时候可能会有各种不完美的情况出现。

第三个限制是我们无法通过反射来声明一个新的类型。



下面的拷贝from

[反射](https://gfw.go101.org/article/reflection.html)

## `reflect.Value`类型和值

类似的，我们可以通过调用`reflect.ValueOf`函数，从一个非接口类型的值创建一个`reflect.Value`值。 此`reflect.Value`值代表着此非接口值。 和`reflect.TypeOf`函数类似，`reflect.ValueOf`函数也只有一个`interface{}`类型的参数。 当我们将一个接口值传递给一个`reflect.ValueOf`函数调用时，此调用返回的是代表着此接口值的动态值的一个`reflect.Value`值。 我们必须通过间接的途径获得一个代表一个接口值的`reflect.Value`值。

被一个`reflect.Value`值代表着的值常称为此`reflect.Value`值的底层值（underlying value）。

`reflect.Value`类型有[很多方法](https://golang.google.cn/pkg/reflect/)。 我们可以调用这些方法来观察和操纵一个`reflect.Value`属主值表示的Go值。 这些方法中的有些适用于所有种类类型的值，有些只适用于一种或几种类型的值。 通过不合适的`reflect.Value`属主值调用某个方法将在运行时产生一个恐慌。 请阅读`reflect`代码库中各个方法的文档来获取如何正确地使用这些方法。

一个`reflect.Value`值的`CanSet`方法将返回此`reflect.Value`值代表的Go值是否可以被修改（可以被赋值）。 如果一个Go值可以被修改，则我们可以调用对应的`reflect.Value`值的`Set`方法来修改此Go值。 注意：`reflect.ValueOf`函数直接返回的`reflect.Value`值都是不可修改的。

一个例子：

```go
package main

import "fmt"
import "reflect"

func main() {
	n := 123
	p := &n
	vp := reflect.ValueOf(p)
	fmt.Println(vp.CanSet(), vp.CanAddr()) // false false
	vn := vp.Elem() // 取得vp的底层指针值引用的值的代表值
	fmt.Println(vn.CanSet(), vn.CanAddr()) // true true
	vn.Set(reflect.ValueOf(789)) // <=> vn.SetInt(789)
	fmt.Println(n)               // 789
}
```



一个结构体值的非导出字段不能通过反射来修改。

```go
package main

import "fmt"
import "reflect"

func main() {
	var s struct {
		X interface{} // 一个导出字段
		y interface{} // 一个非导出字段
	}
	vp := reflect.ValueOf(&s)
	// 如果vp代表着一个指针，下一行等价于"vs := vp.Elem()"。
	vs := reflect.Indirect(vp)
	// vx和vy都各自代表着一个接口值。
	vx, vy := vs.Field(0), vs.Field(1)
	fmt.Println(vx.CanSet(), vx.CanAddr()) // true true
	// vy is addressable but not modifiable.
	fmt.Println(vy.CanSet(), vy.CanAddr()) // false true
	vb := reflect.ValueOf(123)
	vx.Set(vb)     // okay, 因为vx代表的值是可修改的。
	// vy.Set(vb)  // 会造成恐慌，因为vy代表的值是不可修改的。
	fmt.Println(s) // {123 
	fmt.Println(vx.IsNil(), vy.IsNil()) // false true
}
```

上例中同时也展示了如何间接地获取底层值为接口值的`reflect.Value`值。

从上两例中，我们可以得知有两种方法获取一个代表着一个指针所引用着的值的`reflect.Value`值：

1. 通过调用代表着此指针值的`reflect.Value`值的`Elem`方法。
2. 将代表着此指针值的`reflect.Value`值的传递给一个`reflect.Indirect`函数调用。 （如果传递给一个`reflect.Indirect`函数调用的实参不代表着一个指针值，则此调用返回此实参的一个复制。）

注意：`reflect.Value.Elem`方法也可以用来获取一个代表着一个接口值的动态值的`reflect.Value`值，比如下例中所示。

```go
package main

import "fmt"
import "reflect"

func main() {
	var z = 123
	var y = &z
	var x interface{} = y
	v := reflect.ValueOf(&x)
	vx := v.Elem()
	vy := vx.Elem()
	vz := vy.Elem()
	vz.Set(reflect.ValueOf(789))
	fmt.Println(z) // 789
}
```



`reflect`标准库包中也提供了一些对应着内置函数或者各种非反射功能的函数。 下面这个例子展示了如何利用这些函数将一个（效率不高的）自定义泛型函数绑定到不同的类型的函数值上。

```go
package main

import "fmt"
import "reflect"

func InvertSlice(args []reflect.Value) []reflect.Value {
	inSlice, n := args[0], args[0].Len()
	outSlice := reflect.MakeSlice(inSlice.Type(), 0, n)
	for i := n-1; i >= 0; i-- {
		element := inSlice.Index(i)
		outSlice = reflect.Append(outSlice, element)
	}
	return []reflect.Value{outSlice}
}

func Bind(p interface{}, 
		f func ([]reflect.Value) []reflect.Value) {
	// invert代表着一个函数值。
	invert := reflect.ValueOf(p).Elem()
	invert.Set(reflect.MakeFunc(invert.Type(), f))
}

func main() {
	var invertInts func([]int) []int
	Bind(&invertInts, InvertSlice)
	fmt.Println(invertInts([]int{2, 3, 5})) // [5 3 2]

	var invertStrs func([]string) []string
	Bind(&invertStrs, InvertSlice)
	fmt.Println(invertStrs([]string{"Go", "C"})) // [C Go]
}
```



如果一个`reflect.Value`值的底层值为一个函数值，则我们可以调用此`reflect.Value`值的`Call`方法来调用此函数。 每个`Call`方法调用接受一个`[]reflect.Value`类型的参数（表示传递给相应函数调用的各个实参）并返回一个同类型结果（表示相应函数调用返回的各个结果）。

```go
package main

import "fmt"
import "reflect"

type T struct {
	A, b int
}

func (t T) AddSubThenScale(n int) (int, int) {
	return n * (t.A + t.b), n * (t.A - t.b)
}

func main() {
	t := T{5, 2}
	vt := reflect.ValueOf(t)
	vm := vt.MethodByName("AddSubThenScale")
	results := vm.Call([]reflect.Value{reflect.ValueOf(3)})
	fmt.Println(results[0].Int(), results[1].Int()) // 21 9

	neg := func(x int) int {
		return -x
	}
	vf := reflect.ValueOf(neg)
	fmt.Println(vf.Call(results[:1])[0].Int()) // -21
	fmt.Println(vf.Call([]reflect.Value{
		vt.FieldByName("A"), // 如果是字段b，则造成恐慌
	})[0].Int()) // -5
}
```

请注意：非导出结构体字段值不能用做反射函数调用中的实参。 如果上例中的`vt.FieldByName("A")`被替换为`vt.FieldByName("b")`，则将产生一个恐慌。

下面是一个使用映射反射值的例子。

```go
package main

import "fmt"
import "reflect"

func main() {
	valueOf := reflect.ValueOf
	m := map[string]int{"Unix": 1973, "Windows": 1985}
	v := valueOf(m)
	// 第二个实参为Value零值时，表示删除一个映射条目。
	v.SetMapIndex(valueOf("Windows"), reflect.Value{})
	v.SetMapIndex(valueOf("Linux"), valueOf(1991))
	for i := v.MapRange(); i.Next(); {
		fmt.Println(i.Key(), "\t:", i.Value())
	}
}
```

注意：方法`reflect.Value.MapRange`方法是从Go 1.12开始才支持的。

下面是一个使用通道反射值的例子。

```go
package main

import "fmt"
import "reflect"

func main() {
	c := make(chan string, 2)
	vc := reflect.ValueOf(c)
	vc.Send(reflect.ValueOf("C"))
	succeeded := vc.TrySend(reflect.ValueOf("Go"))
	fmt.Println(succeeded) // true
	succeeded = vc.TrySend(reflect.ValueOf("C++"))
	fmt.Println(succeeded) // false
	fmt.Println(vc.Len(), vc.Cap()) // 2 2
	vs, succeeded := vc.TryRecv()
	fmt.Println(vs.String(), succeeded) // C true
	vs, sentBeforeClosed := vc.Recv()
	fmt.Println(vs.String(), sentBeforeClosed) // Go true
	vs, succeeded = vc.TryRecv()
	fmt.Println(vs.String()) // 
	fmt.Println(succeeded)   // false
}
```

`reflect.Value`类型的`TrySend`和`TryRecv`方法对应着只有一个`case`分支和一个`default`分支的[`select`流程控制代码块](https://gfw.go101.org/article/channel.html#select)。

我们可以使用`reflect.Select`函数在运行时刻来模拟具有不定`case`分支数量的`select`流程控制代码块。

```go
package main

import "fmt"
import "reflect"

func main() {
	c := make(chan int, 1)
	vc := reflect.ValueOf(c)
	succeeded := vc.TrySend(reflect.ValueOf(123))
	fmt.Println(succeeded, vc.Len(), vc.Cap()) // true 1 1

	vSend, vZero := reflect.ValueOf(789), reflect.Value{}
	branches := []reflect.SelectCase{
		{Dir: reflect.SelectDefault, Chan: vZero, Send: vZero},
		{Dir: reflect.SelectRecv, Chan: vc, Send: vZero},
		{Dir: reflect.SelectSend, Chan: vc, Send: vSend},
	}
	selIndex, vRecv, sentBeforeClosed := reflect.Select(branches)
	fmt.Println(selIndex)         // 1
	fmt.Println(sentBeforeClosed) // true
	fmt.Println(vRecv.Int())      // 123
	vc.Close()
	// 再模拟一次select流程控制代码块。因为vc已经关闭了，
	// 所以需将最后一个case分支去除，否则它可能会造成一个恐慌。
	selIndex, _, sentBeforeClosed = reflect.Select(branches[:2])
	fmt.Println(selIndex, sentBeforeClosed) // 1 false
}
```



一些`reflect.Value`值可能表示着不合法的Go值。 这样的值为`reflect.Value`类型的零值（即没有底层值的`reflect.Value`值）。

```go
package main

import "reflect"
import "fmt"

func main() {
	var z reflect.Value // 一个reflect.Value零值
	fmt.Println(z)      // 
	v := reflect.ValueOf((*int)(nil)).Elem()
	fmt.Println(v)      // 
	fmt.Println(v == z) // true
	var i = reflect.ValueOf([]interface{}{nil}).Index(0)
	fmt.Println(i)             // 
	fmt.Println(i.Elem())      // 
	fmt.Println(i.Elem() == z) // true
}
```



从上面的例子中，我们知道，使用空接口`interface{}`值做为中介，一个Go值可以转换为一个`reflect.Value`值。 逆过程类似，通过调用一个`reflect.Value`值的`Interface`方法得到一个`interface{}`值，然后将此`interface{}`断言为原来的Go值。 但是，请注意，调用一个代表着非导出字段的`reflect.Value`值的`Interface`方法将导致一个恐慌。

```go
package main

import (
	"fmt"
	"reflect"
	"time"
)

func main() {
	vx := reflect.ValueOf(123)
	vy := reflect.ValueOf("abc")
	vz := reflect.ValueOf([]bool{false, true})
	vt := reflect.ValueOf(time.Time{})

	x := vx.Interface().(int)
	y := vy.Interface().(string)
	z := vz.Interface().([]bool)
	m := vt.MethodByName("IsZero").Interface().(func() bool)
	fmt.Println(x, y, z, m()) // 123 abc [false true] true

	type T struct {x int}
	t := &T{3}
	v := reflect.ValueOf(t).Elem().Field(0)
	fmt.Println(v)             // 3
	fmt.Println(v.Interface()) // panic
}
```

`Value.IsZero`方法是Go 1.13中引进的，此方法用来查看一个值是否为零值。

从Go 1.17开始，[一个切片可以被转化为一个相同元素类型的数组的指针类型](https://gfw.go101.org/article/container.html#slice-to-array-pointer)。 但是如果在这样的一个转换中数组类型的长度过长，将导致恐慌产生。 因此Go 1.17同时引入了一个`Value.CanConvert(T Type)`方法，用来检查一个转换是否会成功（即不会产生恐慌）。

一个使用了`CanConvert`方法的例子：

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	s := reflect.ValueOf([]int{1, 2, 3, 4, 5})
	ts := s.Type()
	t1 := reflect.TypeOf(&[5]int{})
	t2 := reflect.TypeOf(&[6]int{})
	fmt.Println(ts.ConvertibleTo(t1)) // true
	fmt.Println(ts.ConvertibleTo(t2)) // true
	fmt.Println(s.CanConvert(t1))     // true
	fmt.Println(s.CanConvert(t2))     // false
}
```

## 谨慎使用反射

1. 使用反射的代码可能会很脆弱， 万一有个类型Error，可能就会导致panic
2. 反射的代码不易理解
3. 使用反射， 性能相对低一点。关键地方，敏感的地方应该避免使用反射