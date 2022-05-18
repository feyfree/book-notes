# Methods

Methods 需要了解的是

1. 可以作为OOP 领域的一个元素， 需要了解一些封装 和 组合这个基本概念
2. Go 语言通过 func 的使用对象， 通过类型或者类型指针， 将这个method 归到某个类型对象， 类型对象可以直接调用
3. 如果struct T 通过 匿名变量的形式引入了 P， 则 T 可以拥有 P 的相关属性， 包括归类到 P 的相关方法 （这种是通过编译的时候将重写一份方法， 以 T 类型作为归类）， P 方法的如参 形式保持不变
4. 方法不执行， 实际上可以拿出来 定义变量， 类型是 func 
5. 封装的好处是， 在内部package 里面， struct 封装的 field 如果是小写的话， 则 外部引入的话， 无法在外部直接通过 xxx.yyy 访问修改 yyy (xxx 比如是某个类型， yyy 是 fields)

```go
package ch06

import (
	"fmt"
	"go_demo/books_learning/gopl/ch06/geometry"
	"net/url"
	"testing"
	"time"
)

func TestDistance(t *testing.T) {
	perim := geometry.Path{{1, 1}, {5, 1}, {5, 4}, {1, 1}}
	fmt.Println(geometry.PathDistance(perim)) // "12", standalone function
	fmt.Println(perim.Distance())             // "12", method of geometry.Path
}

type IntList struct {
	Value int
	Tail  *IntList
}

func (list *IntList) Sum() int {
	if list == nil {
		return 0
	}
	return list.Value + list.Tail.Sum()
}

func TestIntList(t *testing.T) {
	a := &IntList{Value: 1}
	b := &IntList{Value: 2, Tail: a}
	fmt.Println(b.Sum())
}

func TestUrlValues(t *testing.T) {
	m := url.Values{"lang": {"en"}} // direct construction
	m.Add("item", "1")
	m.Add("item", "2")

	fmt.Println(m.Get("lang")) // "en"
	fmt.Println(m.Get("q"))    // ""
	fmt.Println(m.Get("item")) // "1"      (first value)
	fmt.Println(m["item"])     // "[1 2]"  (direct map access)

	m = nil
	fmt.Println(m.Get("item")) // ""
	m.Add("item", "3")         // panic: assignment to entry in nil map
}

type Rocket struct {
}

func (r *Rocket) Launch() func() {
	var times int
	return func() {
		times++
		fmt.Printf("Rocket Launch now: %d\n", times)
	}
}

func TestExpressionAndValue(t *testing.T) {
	p := geometry.Point{X: 1, Y: 2}
	q := geometry.Point{X: 4, Y: 6}
	distanceFromP := p.Distance        // method value
	fmt.Println(distanceFromP(q))      // "5"
	var origin geometry.Point          // {0, 0}
	fmt.Println(distanceFromP(origin)) // "2.23606797749979", ;5

	r := new(Rocket)
	f := r.Launch()
	time.AfterFunc(10*time.Second, func() {
		f()
	}) // Rocket Launch now: 2
	time.AfterFunc(1*time.Second, f) // Rocket Launch now: 1
	time.Sleep(15 * time.Second)
}

func TestInnerFields(t *testing.T) {
	// 外部引用无法对小写的内部 field 直接访问
	//set:= intset.IntSet{}
	//set.words
}

```

