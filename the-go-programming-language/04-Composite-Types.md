# Composite Types

组合类型， Go 里面主要讨论 4 个

1. arrays 数组
2. slices 切片
3. maps 
4. structs 结构体

最后我们看一下 Go Templates， 结合 JSON 和 HTML 看一下

## 1. Arrays

```go
package ch04

// Go 默认是值传递 所以如果用数组的话， 尽量用指针， 避免复制的开销

import (
	"crypto/sha256"
	"fmt"
	"testing"
)

type Currency int

const (
	USD Currency = iota
	EUR
	GBP
	RMB
)

func TestArray0(t *testing.T) {
	symbol := [...]string{USD: "$", EUR: "€", GBP: "£", RMB: "¥"}
	fmt.Println(RMB, symbol[RMB])

	// index = 3, value = 1 说明长度是 4， 并且 在 index < 3 的地方都是 0
	r := [...]int{3: 1} // [0 0 0 1]
	fmt.Println(r)

	a := [2]int{1, 2}
	b := [...]int{1, 2}
	c := [2]int{1, 3}
	fmt.Println(a == b, a == c, b == c) // "true false false"
	fmt.Println(&a[0], &b[0])
	d := [3]int{1, 2}
	// 这个会报编译错误 compile error: cannot compare [2]int == [3]int }
	// fmt.Println(a == d) //
	fmt.Println(d)
}

func TestArraySha256(t *testing.T) {
	// 注意这个是小写的 x 和 大写的 X 的sha256 的比较
	c1 := sha256.Sum256([]byte("x"))
	c2 := sha256.Sum256([]byte("X"))
	fmt.Printf("%x\n%x\n%t\n%T\n", c1, c2, c1 == c2, c1)
	// Output:
	// 2d711642b726b04401627ca9fbac32f5c8530fb1903cc4db02258717921a4881
	// 4b68ab3847feda7d6c62c1fbcbeebfa35eab7351ed5e78f4ddadea5df64b8015
	// false
	// [32]uint8
}

```

## 2. Slices

slice has three components

1. pointer 
2. length
3. cap

Go 内置了 2 个函数， 一个 len() 一个 cap()

```go
package ch04

import (
	"fmt"
	"testing"
)

func TestSliceStruct(t *testing.T) {
	a := [...]int{1, 2, 3}
	b := a[:1]
	fmt.Println(len(b), cap(b)) // 1 3

}

func reverse(s []int) {
	for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
		s[i], s[j] = s[j], s[i]
	}
}

func TestReverse(t *testing.T) {
	array := [...]int{1, 2, 3}
	slice := array[:]
	reverse(slice)
	fmt.Println(slice) // [3 2 1]
	fmt.Println(array) // [3 2 1]
}

// 判断一个slice是不是 empty, 用 len(s) 去判断
func TestExpression(t *testing.T) {
	var s []int // len(s) == 0, s == nil
	fmt.Println(len(s), s == nil)
	s = nil // len(s) == 0, s == nil
	fmt.Println(len(s), s == nil)
	s = []int(nil) // len(s) == 0, s == nil
	fmt.Println(len(s), s == nil)
	s = []int{} // len(s) == 0, s != nil
	fmt.Println(len(s), s == nil)
}

// 使用make 构建指定类型的slice
func TestMake(t *testing.T) {
	a := make([]int, 10)
	fmt.Println(len(a), cap(a))
	fmt.Println(len(a), cap(a))
	b := make([]string, 10, 16)
	fmt.Println(len(b), cap(b))
}

// 一些操作
func TestOperations(t *testing.T) {
	var runes []rune
	for _, r := range "Hello, 世界" {
		runes = append(runes, r)
	}
	fmt.Printf("%q\n", runes) // "['H' 'e' 'l' 'l' 'o' ',''''世' '界']"
}

func appendSlice(x []int, y ...int) []int {
	var z []int
	zlen := len(x) + len(y)
	if zlen <= cap(x) {
		// There is room to expand the slice.
		z = x[:zlen]
	} else {
		// There is insufficient space.
		// Grow by doubling, for amortized linear complexity.
		zcap := zlen
		if zcap < 2*len(x) {
			zcap = 2 * len(x)
		}
		z = make([]int, zlen, zcap)
		copy(z, x)
	}
	copy(z[len(x):], y)
	return z
}

func TestAppendSlice(t *testing.T) {
	x := []int{1, 2, 3}
	slice := appendSlice(x, 1, 2, 3)
	fmt.Println(slice)
}

//!+append
func appendInt(x []int, y int) []int {
	var z []int
	zlen := len(x) + 1
	if zlen <= cap(x) {
		// There is room to grow.  Extend the slice.
		z = x[:zlen]
	} else {
		// There is insufficient space.  Allocate a new array.
		// Grow by doubling, for amortized linear complexity.
		zcap := zlen
		if zcap < 2*len(x) {
			zcap = 2 * len(x)
		}
		z = make([]int, zlen, zcap)
		copy(z, x) // a built-in function; see text
	}
	z[len(x)] = y
	return z
}

//!+growth
func TestAppendGrowth(t *testing.T) {
	var x, y []int
	for i := 0; i < 10; i++ {
		y = appendInt(x, i)
		fmt.Printf("%d  cap=%d\t%v\n", i, cap(y), y)
		x = y
	}
}

//!-growth

/*
//!+output
0  cap=1   [0]
1  cap=2   [0 1]
2  cap=4   [0 1 2]
3  cap=4   [0 1 2 3]
4  cap=8   [0 1 2 3 4]
5  cap=8   [0 1 2 3 4 5]
6  cap=8   [0 1 2 3 4 5 6]
7  cap=8   [0 1 2 3 4 5 6 7]
8  cap=16  [0 1 2 3 4 5 6 7 8]
9  cap=16  [0 1 2 3 4 5 6 7 8 9]
//!-output
*/

func nonempty(strings []string) []string {
	i := 0
	for _, s := range strings {
		if s != "" {
			strings[i] = s
			i++
		}
	}
	return strings[:i]
}

func nonempty2(strings []string) []string {
	out := strings[:0] // zero-length slice of original
	for _, s := range strings {
		if s != "" {
			out = append(out, s)
		}
	}
	return out
}

func TestNonempty(t *testing.T) {
	data := []string{"one", "", "three"}
	fmt.Printf("%q\n", nonempty(data))  // `["one" "three"]`
	fmt.Printf("%q\n", data)            // `["one" "three" "three"]`
	fmt.Printf("%q\n", nonempty2(data)) // `["one" "three" "three"]`
}
```



![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220513171112-two-slices-overlapping-array.png)

```go
Q2 := months[4:7]
summer := months[6:9]
fmt.Println(Q2) // ["April" "May" "June"]
fmt.Println(summer) // ["June" "July" "August"]
```

slice 需要注意的是，

1. 判断一个slice 是否为空， 用 len(s) 去判断，slice != nil 不代表slice 不为空
2. slice 指向数组， 对slice 的修改实际也会体现在 数组中， 两个overlapping 的 在一个数组上的
3. slices are not comprable 所以slice 不能用 == 号去判断slice 的相等
4. slice 作 append 操作的时候， 会有返回值， 因为一个slice append 过程中，底层的数组说不定会存在扩容等等 （拷贝机制）
5. slice[:0]. zero-length slice of original 

## 3. Maps

```go
package ch04

import (
	"fmt"
	"sort"
	"testing"
)

func TestMaps(t *testing.T) {

	ages := make(map[string]int) // mapping from strings to ints
	fmt.Println(ages)
	ages = map[string]int{
		"alice":   31,
		"charlie": 34,
	}
	fmt.Println(ages)
	delete(ages, "alice")
	fmt.Println(ages)
	ages["bob"] = ages["bob"] + 1
	fmt.Println(ages)
	// 用来判定是否存在一个 key
	_, ok := ages["allen"]
	if !ok {
		fmt.Println("Not contain allen")
	}
	for name, age := range ages {
		fmt.Printf("%s\t%d\n", name, age)
	}
	// 会报编译错误 Cannot take the address of 'ages["bob"]', 原因是map element 中的value 不能取地址
	// i := &ages["bob"]

	var names []string
	for name := range ages {
		names = append(names, name)
	}
	fmt.Println("----Next print sorted names with age----")
	sort.Strings(names)
	for _, name := range names {
		fmt.Printf("%s\t%d\n", name, ages[name])
	}
}

func TestMaps2(t *testing.T) {
	var ages map[string]int
	fmt.Println(ages == nil)    // "true"
	fmt.Println(len(ages) == 0) // "true"
	// nil map 不能分配entry
	//ages["carol"] = 21          // panic: assignment to entry in nil map
}

func equal(x, y map[string]int) bool {
	if len(x) != len(y) {
		return false
	}
	for k, xv := range x {
		if yv, ok := y[k]; !ok || yv != xv {
			return false
		}
	}
	return true
}

func TestEqual(t *testing.T) {
	x := map[string]int{"a": 1, "b": 2}
	y := map[string]int{"a": 1, "c": 2}
	fmt.Println(equal(x, y))
}

```

map 类型需要注意的是

1. nil map 不能分配 entry ， 也就是 如果仅仅是通过 var 变量声明了一下， 并不能直接 xxx[y] = z 这种形式插值
2. map element 中的 value 不能取地址
3. 判断一个值是否在 map 里面， 用 `	_, ok := ages["allen"]`这种形式来判断, 不能直接用key 取值 （不存在的默认是zero value）

## 4. Structs

```go
package ch04

import (
	"fmt"
	"go_demo/books_learning/gopl/ch04/structs"
	"testing"
)

func TestStruct(t *testing.T) {
	// 下面会编译报错， 因为 a, b 小写是不能被exported的
	//t := structs.T{1, 1}

	// 这样写也是可以的， 但是不推荐
	//e := structs.E{1, 2}
	e := &structs.E{X: 1, Y: 2}
	fmt.Println(*e)
}

// 匿名的话 struct 继承内部的 匿名部分的 sub fields
// 使用匿名 可以使用类似直接的sub fields 赋值， 但是无法进行直接的 构造
func TestAnonymousFields(t *testing.T) {
	var w structs.Wheel
	w.X = 8      // equivalent to w.Circle.Point.X = 8
	w.Y = 8      // equivalent to w.Circle.Point.Y = 8
	w.Radius = 5 // equivalent to w.Circle.Radius = 5
	w.Spokes = 20
	fmt.Println(w)

	w = structs.Wheel{
		Circle: structs.Circle{
			Point:  structs.Point{X: 8, Y: 8},
			Radius: 5,
		},
		Spokes: 20, // NOTE: trailing comma necessary here (and at Radius)
	}
	fmt.Printf("%#v\n", w)
	w.X = 42
	fmt.Printf("%#v\n", w)

}
```

```go
package structs

// T 如果内部field是小写的话， field是不能被外部包直接引用的
type T struct {
	a, b int
}

type E struct {
	X, Y int
}

type Point struct {
	X, Y int
}

//type Circle struct {
//	Center Point
//	Radius int
//}
//type Wheel struct {
//	Circle Circle
//	Spokes int
//}

// Circle 匿名写法
type Circle struct {
	Point
	Radius int
}

type Wheel struct {
	Circle
	Spokes int
}

```

structs 的注意点

1. struct 定义的 内部 field 如果是小写开头， 跨包引入， 无法访问
2. struct 的 初始化， 最好带上变量名
3. struct 支持匿名变量， 这样子 可以通过 最外层的 struct.FieldAAA 形式 来访问 匿名变量的 FieldAAA 。但是初始化的时候， 还是需要通过变量名的嵌套

```go
	w = structs.Wheel{
		Circle: structs.Circle{
			Point:  structs.Point{X: 8, Y: 8},
			Radius: 5,
		},
		Spokes: 20, // NOTE: trailing comma necessary here (and at Radius)
	}
```

## 5. JSON

```go
import (
	"encoding/json"
	"fmt"
	"log"
)

//!+
type Movie struct {
	Title  string
	Year   int  `json:"released"`
	Color  bool `json:"color,omitempty"`
	Actors []string
}

var movies = []Movie{
	{Title: "Casablanca", Year: 1942, Color: false,
		Actors: []string{"Humphrey Bogart", "Ingrid Bergman"}},
	{Title: "Cool Hand Luke", Year: 1967, Color: true,
		Actors: []string{"Paul Newman"}},
	{Title: "Bullitt", Year: 1968, Color: true,
		Actors: []string{"Steve McQueen", "Jacqueline Bisset"}},
	// ...
}

//!-

func show() {
	{
		//!+Marshal
		data, err := json.Marshal(movies)
		if err != nil {
			log.Fatalf("JSON marshaling failed: %s", err)
		}
		fmt.Printf("%s\n", data)
		//!-Marshal
	}

	{
		//!+MarshalIndent
		data, err := json.MarshalIndent(movies, "", "    ")
		if err != nil {
			log.Fatalf("JSON marshaling failed: %s", err)
		}
		fmt.Printf("%s\n", data)
		//!-MarshalIndent

		//!+Unmarshal
		var titles []struct{ Title string }
		if err := json.Unmarshal(data, &titles); err != nil {
			log.Fatalf("JSON unmarshaling failed: %s", err)
		}
		fmt.Println(titles) // "[{Casablanca} {Cool Hand Luke} {Bullitt}]"
		//!-Unmarshal
	}
}

```

Go 中， 并没有 json 这种单独出现的类型， 

```go
type Movie struct {
	Title  string
	Year   int  `json:"released"`
	Color  bool `json:"color,omitempty"`
	Actors []string
}
```

这种写法实际上是 对struct 对象的 进行json 序列化的时候 如何将 Field 进行转化

Go 内置了 json 的 编码库， 可以直接使用

## 6. Template

### 6.1 text template

```go
package gotemplate

import (
	"go_demo/books_learning/gopl/ch04/github"
	"log"
	"os"
	"testing"
	"text/template"
	"time"
)

//!+gotemplate
const templ = `{{.TotalCount}} gotemplate:
{{range .Items}}----------------------------------------
Number: {{.Number}}
User:   {{.User.Login}}
Title:  {{.Title | printf "%.64s"}}
Age:    {{.CreatedAt | daysAgo}} days
{{end}}`

//!-gotemplate

//!+daysAgo
func daysAgo(t time.Time) int {
	return int(time.Since(t).Hours() / 24)
}

//!-daysAgo

//!+exec
var report = template.Must(template.New("issuelist").
	Funcs(template.FuncMap{"daysAgo": daysAgo}).
	Parse(templ))

func TestReport(t *testing.T) {
	terms := []string{"repo:golang/go", "is:open", "json", "decoder"}
	result, err := github.SearchIssues(terms)
	if err != nil {
		log.Fatal(err)
	}
	if err := report.Execute(os.Stdout, result); err != nil {
		log.Fatal(err)
	}
}

//!-exec

func noMust() {
	//!+parse
	report, err := template.New("report").
		Funcs(template.FuncMap{"daysAgo": daysAgo}).
		Parse(templ)
	if err != nil {
		log.Fatal(err)
	}
	//!-parse
	result, err := github.SearchIssues(os.Args[1:])
	if err != nil {
		log.Fatal(err)
	}
	if err := report.Execute(os.Stdout, result); err != nil {
		log.Fatal(err)
	}
}

/*
//!+output
$ go build gopl.io/ch4/issuesreport
$ ./issuesreport repo:golang/go is:open json decoder
13 gotemplate:
----------------------------------------
Number: 5680
User:   eaigner
Title:  encoding/json: set key converter on en/decoder
Age:    750 days
----------------------------------------
Number: 6050
User:   gopherbot
Title:  encoding/json: provide tokenizer
Age:    695 days
----------------------------------------
...
//!-output
*/
```

### 6.2 html template

```go
package gotemplate

import (
	"go_demo/books_learning/gopl/ch04/github"
	"log"
	"os"
	"testing"
)

//!+gotemplate
import "html/template"

var issueList = template.Must(template.New("issuelist").Parse(`
<h1>{{.TotalCount}} gotemplate</h1>
<table>
<tr style='text-align: left'>
  <th>#</th>
  <th>State</th>
  <th>User</th>
  <th>Title</th>
</tr>
{{range .Items}}
<tr>
  <td><a href='{{.HTMLURL}}'>{{.Number}}</a></td>
  <td>{{.State}}</td>
  <td><a href='{{.User.HTMLURL}}'>{{.User.Login}}</a></td>
  <td><a href='{{.HTMLURL}}'>{{.Title}}</a></td>
</tr>
{{end}}
</table>
`))

//!-gotemplate

//!+
func TestHtml(t *testing.T) {
	//result, err := github.SearchIssues(os.Args[1:])
	terms := []string{"repo:golang/go", "is:open", "json", "decoder"}
	result, err := github.SearchIssues(terms)
	if err != nil {
		log.Fatal(err)
	}
	if err := issueList.Execute(os.Stdout, result); err != nil {
		log.Fatal(err)
	}
}
```

escape 

```go
import (
	"html/template"
	"log"
	"os"
	"testing"
)

func TestAutoEscape(t *testing.T) {
	const templ = `<p>A: {{.A}}</p><p>B: {{.B}}</p>`
	temp := template.Must(template.New("escape").Parse(templ))
	var data struct {
		A string        // untrusted plain text
		B template.HTML // trusted HTML
	}
	data.A = "<b>Hello!</b>"
	data.B = "<b>Hello!</b>"
	if err := temp.Execute(os.Stdout, data); err != nil {
		log.Fatal(err)
	}
}
```

