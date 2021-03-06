# Concurrency-With-Shared-Variables

通过共享变量， 也是实现并发的一个方案

## 1. Race Conditions

竞态问题， 就是多个 goroutine 同时访问一个地址， 会出现并发修改或者读写不一致的问题

如何避免竞态的问题

1. 避免写变量
2. 避免变量被多个goroutines 访问，或者说是 变量仅限某一个goroutine 修改
3. 允许多个goroutine 访问， 但是每次只有一个能访问 (mutual exclusion)

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220524182513-go-mantra01.png)

这句话的翻译

既然其他的goroutine 不能直接访问变量， 所以他们必须通过 channel 发送请求给指定 goroutine 去查询或者更新这个变量。 这就是 Go 的那句经典谚语： 不要通过共享内存去通信， 而是通过通信去共享内存。 

## 2. sync.Mutex

**理解**

1. critical section
2. happen before

**使用defer 进行 unlock 的时候， 需要注意**

1. defer 可能会扩大 lock 的范围， 实际上还不如显式释放更高效
2. 临界区越小越好， 上锁越迟越好

**Go 的 mutex 不是 re-entrant ( it is not possible to lock a mutex that is already locked)**

[Go的Mutex为什么不支持可重入](https://blog.csdn.net/eddycjy/article/details/121965136)

[Experimenting with GO](https://groups.google.com/g/golang-nuts/c/XqW1qcuZgKg/m/Ui3nQkeLV80J)

```go
// 测试可冲入锁 会报错
// === RUN   TestReentrant
// fatal error: all goroutines are asleep - deadlock!
func TestReentrant(t *testing.T) {
	var mutex sync.Mutex
	mutex.Lock()
	mutex.Lock()
}
```

mutex 的目的是 **要保证这些变量的不变性保持，不会在后续的过程中被破坏**

当我们使用 mutex 的时候， 确保它还有它守护的变量， 不被 exported, 不管是package-level 还是 在struct 结构体中

## 3. sync.RWMutex

读读不会互斥， 但是只要有写锁， 就会互斥

### 4. Memory Synchronization

1. 多核处理器，存在各自的 local cache
2. 是否存在指令重排的问题

编码准则

1. 有必要的话，将变量限定在一个goroutine 里面
2. 使用互斥

## 5. sync.Once

只能创建一次 （使用一次）这种语义， 可以避免重复创建的问题

## 6. The Race Detector

可以使用 `-race` 进行启动， go build， go run， go test 等等， go 会帮助检查竞态的问题

## 7. Goroutine Vs Threads

**growable stacks**

os threads 有固定的栈大小， 通常 2M大小， 所以对于某些场景， 可能存在过大或者过小的情况， goroutine 一般从 2K开始， 可以逐渐增加

**goroutine scheduling**

1. thread 一般是 os kernel 调度， 调度存在上下文切换
2. The Go runtime contains its own scheduler that uses a technique known as **m:n scheduling**, because it multiplexes (or schedules) m goroutines on n OS threads.

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220525095827-go-scheduling.png)

**GOMAXPROCS** 

GOMAXPROCS is the n in m:n scheduling

