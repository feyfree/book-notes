# Goroutines and channels

## 背景

1. 利用并发去屏蔽I/O的延迟
2. 更好的利用当今多核处理器

## Go 的特点

### **两种并发编程**

1. goroutine 和 channel （CSP 模型），实体（goroutine）之间通过发送消息进行通信，这里发送消息时使用的就是通道，或者叫 channel。CSP 模型的关键是关注 channel，而不关注发送消息的实体
2. 多线程共享内存

### **Go 的 注意点**

1. 每个并发执行的活动都属于goroutine
2. 线程 和 goroutine 的区别是 quantitative （定量的）而不是 qualitative （定性的）
3. 程序启动的， main 函数运行 main goroutine， 其余的通过 go 关键字启动

### Channel 的 注意点

1.  channel 连接 goroutine， channel 是 goroutine 之间的通信机制。 channel 只能传导 设定好的类型的值（element type）
2. channel 的 相关使用

```go
ch := make(chan int) // ch has type 'chan int'
ch <- x // a send statement
x=<-ch // a receive expression in an assignment statement
<-ch // a receive statement; result is discarded
close(ch) // close a channel
ch = make(chan int) // unbuffered channel
ch = make(chan int, 0) // unbuffered channel
ch = make(chan int, 3) // buffered channel with capacity 3
```

3. unbuffered channels  （发送成功的条件是， channel 为空， 接受成功的条件是， channel 不为空）， 同步的语义 (synchronize), happens-before

```go
func main() {
	conn, err := net.Dial("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	done := make(chan struct{})
	go func() {
		io.Copy(os.Stdout, conn) // NOTE: ignoring errors
		log.Println("done")
		done <- struct{}{} // signal the main goroutine
	}()
	mustCopy(conn, os.Stdin)
	conn.Close()
	<-done // wait for background goroutine to finish
}

//!-

func mustCopy(dst io.Writer, src io.Reader) {
	if _, err := io.Copy(dst, src); err != nil {
		log.Fatal(err)
	}
}
```

4. 可以通过二元返回， 判断一个channel 是否是关闭的

```go
x, ok := <-naturals
if !ok {
		break // channel was closed and drained
}
```

5. channel 也可以通过 for range 来避免这种二元判断, 将其使用在channel上时，会自动等待channel的动作一直到channel被关闭

```go
func squarer(out chan<- int, in <-chan int) {
	time.Sleep(time.Second)
	for v := range in {
		out <- v * v
	}
	close(out)
}
```

6. 函数入参可以定义单向channel， 这样调用函数的时候， channel 传入会隐式 转化成为单向 channel， 并且在函数内部只支持单向操作
7. channel 也可以是 buffered， 可以通过 make(chan xxx, length) 创建， 可以通过 len 查看 buffered channel 长度, cap 查看buffered channel 的 容量

### select 关键字

`select`关键字用于多个channel的结合，这些channel会通过类似于**are-you-ready polling**的机制来工作。`select`中会有`case`代码块，用于发送或接收数据——不论通过`<-`操作符指定的发送还是接收操作准备好时，channel也就准备好了。在`select`中也可以有一个`default`代码块，其一直是准备好的。那么，在`select`中，哪一个代码块被执行的算法大致如下：

- 检查每个`case`代码块
- 如果任意一个`case`代码块准备好发送或接收，执行对应内容
- 如果多于一个`case`代码块准备好发送或接收，随机选取一个并执行对应内容
- 如果任何一个`case`代码块都没有准备好，等待
- 如果有`default`代码块，并且没有任何`case`代码块准备好，执行`default`代码块对应内容

### Channel 什么时候需要关闭

1. 告诉所有接收的goroutines， 数据已经发送完了， 这时候需要关闭 channel
2. 当 GC 处理器认为 channel 是不可达的时候，channel 资源会被回收 （不管有没有close 的操作， 文件操作还是需要记住需要调用close 操作的）
3. 关闭已经被关闭的 channel 会 panic ， 好比是关闭了一个 nil channel

### 如何取消

首先我们要明白

1. 一个goroutine 是无法直接终止另外一个 goroutine 的， 如果终止会讲他的所有共享变量处于一个 undefined 状态
2. Recall that after a channel has been closed and drained of all sent values, subsequent receive operations proceed immediately, yielding zero values. We can exploit this to create a broadcast mechanism: don’t send a value on the channel, close it
3. 增加一个传递关闭信号的 channel，receiver 通过信号 channel 下达关闭数据 channel 指令。senders 监听到关闭信号后，停止发送数据

```go
var done = make(chan int)

func cancelled() bool {
	select {
	case <-done:
		return true
	default:
		return false
	}
}
```

### Chat Server 示例

```go
package main

import (
	"bufio"
	"fmt"
	"log"
	"net"
)

//!+broadcaster
type client chan<- string // an outgoing message channel

var (
	entering = make(chan client)
	leaving  = make(chan client)
	messages = make(chan string) // all incoming client messages
)

func broadcaster() {
	clients := make(map[client]bool) // all connected clients
	for {
		select {
		case msg := <-messages:
			// Broadcast incoming message to all
			// clients' outgoing message channels.
			for cli := range clients {
				cli <- msg
			}

		case cli := <-entering:
			clients[cli] = true

		case cli := <-leaving:
			delete(clients, cli)
			close(cli)
		}
	}
}

//!-broadcaster

//!+handleConn
func handleConn(conn net.Conn) {
	ch := make(chan string) // outgoing client messages
	go clientWriter(conn, ch)

	who := conn.RemoteAddr().String()
	ch <- "You are " + who
	messages <- who + " has arrived"
	entering <- ch

	input := bufio.NewScanner(conn)
	for input.Scan() {
		messages <- who + ": " + input.Text()
	}
	// NOTE: ignoring potential errors from input.Err()

	leaving <- ch
	messages <- who + " has left"
	conn.Close()
}

func clientWriter(conn net.Conn, ch <-chan string) {
	for msg := range ch {
		fmt.Fprintln(conn, msg) // NOTE: ignoring network errors
	}
}

//!-handleConn

//!+main
func main() {
	listener, err := net.Listen("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}

	go broadcaster()
	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Print(err)
			continue
		}
		go handleConn(conn)
	}
}

//!-main

```





